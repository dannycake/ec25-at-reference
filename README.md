# Quectel EC25 — AT command reference and field notes

Notes from driving a **Quectel EC25-AFX** (mini-PCIe, firmware `EC25AFXGAR07A04M1G`) on a
US carrier over USB: SMS, MMS send *and* receive, LTE data, and GNSS.

Most of this is in Quectel's manuals somewhere. What is here that isn't there: which
commands actually behave differently from the documentation, and the specific failures that
cost time. Every ✅ below was run on real hardware.

Legend — ✅ verified on hardware · 📋 listed by `AT+CLAC` on this firmware · 📖 documented but not exercised here

---

## Start here: five things that will cost you an afternoon

### 1. `AT+CLAC` is not exhaustive

It returns 252 entries on this firmware and **omits commands that demonstrably work** —
`+CMGS`, `+CMGL`, `+QGPS`, `+QHTTPCFG`, `+QMTCFG`, `+QICSGP`, `+QENG`, `+QLTS`, `+CRSM`.
Never conclude a command is unsupported because CLAC didn't list it. Test it.

### 2. `AT+CMGS` has a prompt race that looks like success

The documented flow is `AT+CMGS="<number>"`, wait for `>`, send the body, then Ctrl-Z
(`0x1A`). If you pipe all of it at once, the body starts arriving before the module emits
`>`, and the leading characters are **silently swallowed**. You still get `+CMGS: <ref>`
and an `OK`. The message just arrives truncated.

With busybox `microcom`, `-d 300` fixes it — *"wait up to DELAY ms for TTY output before
sending every next byte"* makes the write self-synchronising against the prompt.

### 3. There is no non-mutating way to list SMS

`AT+CMGL="ALL"` flips every `REC UNREAD` to `REC READ` as a side effect. The spec's
non-mutating form is `AT+CMGL="ALL",1` — **this firmware answers `ERROR`**. So unread state
has to be tracked by your client, not asked of the modem.

### 4. The message store silently drops when full

`AT+CPMS?` reports `used,total` — the cap is **255**. When it fills, incoming messages are
**discarded with no error and no indication**. Any real client must copy messages into its
own store and `AT+CMGD` them off the module. This is a correctness requirement.

### 5. UCS2 mode hex-encodes the phone number too

To send emoji or non-GSM characters you need all three of:

```
AT+CMGF=1
AT+CSCS="UCS2"
AT+CSMP=17,167,0,8      # DCS 8 = UCS2
```

…and then **the destination number in `AT+CMGS` is also a UCS2 string** and must be
hex-encoded like the body. Passing it as plain ASCII fails. Astral-plane characters become
UTF-16 surrogate pairs — 🦨 (U+1F9A8) goes out as `D83EDDA8`.

Inbound, when `CSCS` is `GSM`, UCS2 messages come back as raw hex. That's correct
behaviour, not corruption — decode it as UTF-16BE.

---

## MMS — the part with no library

MMS is **not** an AT feature. Sending is:

1. Encode an **M-Send.req PDU** (OMA MMS Encapsulation, WSP binary format)
2. Bring up a cellular data session
3. **HTTP POST** it to the carrier's MMSC with
   `Content-Type: application/vnd.wap.mms-message`
4. Parse the **M-Send.conf** response

Receiving is the mirror: a binary **WAP-push SMS** arrives carrying a URL; GET that URL over
the same cellular path and parse the **M-Retrieve.conf** multipart.

### WSP encoding rules that bite

| Rule | Detail |
|---|---|
| Short integer | `0x80 \| value`, values 0–127 only |
| Text string | bytes + NUL. **If the first byte is ≥ 0x80 it must be quoted with a leading `0x7F`** or the parser desyncs |
| Quoted string | `0x22`, the text, then NUL — **no closing quote** |
| Value length | one byte if < 31, else `0x1F` followed by a uintvar |
| Uintvar | base-128, big-endian, high bit set on all but the last byte |

⚠️ **`Content-ID` is a quoted-string, not a text-string.** Send it unquoted and the MMSC
returns `0xE2 Error-permanent-message-format-corrupt`. The socket accepts every byte and the
MMSC even allocates a message ID, so it reads like a delivery problem rather than an
encoding one. The parser desyncs on the `<`.

⚠️ **Header order is not free.** `X-Mms-Message-Type`, `X-Mms-Transaction-ID` and
`X-Mms-MMS-Version` must be the first three; **`Content-Type` must be last** before the body.

⚠️ **The body is not MIME.** It's WSP multipart: a uintvar part count, then per part a
`(HeadersLen, DataLen, headers, data)` quad — where `HeadersLen` covers the content-type
byte(s) *and* the part headers.

### What an MMSC actually accepts

All three of these were accepted (`status=0x80`, `1000:OK`):

| | Structure |
|---|---|
| A | `multipart.mixed`, image only, no Content-ID |
| B | `multipart.mixed` + quoted Content-ID + Content-Location |
| C | `multipart.related` + SMIL + quoted Content-IDs |

So **SMIL is optional for delivery** — include it for layout control, not to satisfy the
carrier. Well-known content types: `image/jpeg` `0x1E`, `image/png` `0x20`, `image/gif`
`0x1D`, `text/plain` `0x03`, `multipart.mixed` `0x23`, `multipart.related` `0x33`.
**WebP is not a well-known type — convert it.** PNG carries alpha through intact, though
whether it *renders* is the receiving app's decision.

### Networking traps

⚠️ **The MMSC only resolves on carrier DNS** and lives on RFC1918 space inside the carrier
network. Your system resolver can't see it, so `curl` fails with `http=000` before sending a
byte. Resolve against the DNS the data session handed you and pass `--resolve`.

⚠️ **The retrieval host may be a CNAME**, and `dig +short` prints the CNAME *first*. Piping
to `head -n1` grabs the alias, `--resolve` gets a hostname where it wants an IP, and you get
`http=000` again — indistinguishable from having no route. Filter for an A record.

⚠️ **ICMP to the MMSC is typically blocked.** Ping reports 100% loss while TCP/80 connects
fine. Don't use ping as your reachability test.

⚠️ **The POST must originate from the cellular source address**, or it egresses your default
interface and never finds a carrier-private host.

---

## GNSS

```
AT+QGPS=1            start (modes 1–4: standalone / MS-based / MS-assisted / speed-optimal)
AT+QGPSEND           stop
AT+QGPSLOC=2         get a fix   (+CME ERROR: 516 = not fixed yet)
AT+QGPSCFG="outport" where NMEA goes — "usbnmea" streams it on a separate ttyUSB
```

Accuracy is **<2.5 m CEP-50** (Qualcomm IZat Gen8C Lite; GPS/GLONASS/BDS/Galileo/QZSS).
TTFF cold 26–28 s; with gpsOneXTRA, warm 2.2–3.8 s, hot 1.5–3.4 s, cold reduced by 18–30 s.

⚠️ **`AT+QGPSXTRA?` defaults to `0`.** Assisted GPS is off out of the box; enabling it needs
a data session but turns a half-minute cold start into a couple of seconds.

**Diagnosing no fix, from the NMEA alone:**

| Symptom | Meaning |
|---|---|
| No `$G.GSV` sentences at all | Zero satellites *visible* — almost always no antenna connected |
| `$G.GSV` present, `$GPGGA` fix quality `0` | Receiver is working; you have fewer than 4 satellites, or SNR too low (want > 30 dB-Hz) |
| `$GPRMC` field `V` | Void — no valid fix |

### The active-vs-passive antenna question

The GNSS pin supplies **2.85 V** bias, so active antennas work directly. But Quectel's
hardware design recommends a **passive** antenna *when B13 or B14 are supported*, because
of transmit harmonics. Work out which bands that actually means:

| Band | Uplink (MHz) | 2nd harmonic | Lands on GPS L1 (1575.42)? |
|---|---|---|---|
| **B13** | 777–787 | 1554–1574 | at the edge |
| **B14** | 788–798 | 1576–1596 | **directly on it** |
| B71 | 663–698 | 1326–1396 | no |
| B12 | 699–716 | 1398–1432 | no |
| B5 | 824–849 | 1648–1698 | no |
| B2 / B4 | 1850+ / 1710+ | — | no |

B13 is Verizon, B14 is FirstNet. **If your carrier doesn't put you on those, an active
antenna is fine** — and active is what you want, since it's ~28 dB of LNA.

⚠️ Ceramic patch antennas need a **ground plane** (roughly 50–70 mm under a 25 mm patch) and
are **directional** — they must face the sky. A magnetic puck includes its own ground plane;
a bare patch on plastic underperforms its spec.

---

## Command tables

### Basic / Hayes

| Command | | Purpose |
|---|---|---|
| `ATE0` / `ATE1` | ✅ | Echo off/on. **Turn it off in a program** or every response is prefixed by your command |
| `ATI` | ✅ | Manufacturer, model, firmware revision |
| `AT+CMEE=2` | ✅ | **Set this first** — verbose error text instead of bare `ERROR` |
| `AT&F` / `AT&W` / `AT&V` / `ATZ` | 📋 | Factory reset / save profile / dump config / reset to profile |
| `ATQ` `ATV` `ATX` | 📋 | Result-code suppression / verbosity / call-progress detail |
| `AT&C` `AT&D` `AT\Q` | 📋 | DCD behaviour / DTR behaviour / flow control |
| `ATS0` `ATS3` `ATS4` `ATS5` | 📋 | Auto-answer rings, line terminators, backspace |
| `ATD` `ATA` `ATH` `ATO` | 📋 | Dial / answer / hang up / return to data mode |
| `AT+IPR` `AT+ICF` `AT+IFC` | 📋 | UART baud / framing / flow control (moot over USB) |

### Device identity

| Command | | Purpose |
|---|---|---|
| `AT+GSN` / `AT+CGSN` | ✅ | IMEI |
| `AT+GMI` `AT+GMM` `AT+GMR` | 📋 | Manufacturer / model / revision individually |
| `AT+QVERSION` `AT+QSUBSYSVER` | 📋 | Firmware and subsystem versions |
| `AT+QCPUSN` | 📋 | CPU serial number |
| `AT+QNETIFMAC` | 📋 | Network interface MAC |
| `AT+QUSBSPEED` | 📋 | USB speed configuration |

### SIM

| Command | | Purpose |
|---|---|---|
| `AT+CPIN?` | ✅ | SIM state; `READY` = present and unlocked |
| `AT+QCCID` | ✅ | ICCID |
| `AT+CIMI` | ✅ | IMSI |
| `AT+CNUM` | ✅ | **The SIM's own number** — so you never need it configured |
| `AT+CRSM` | ✅ | **Raw SIM file access** — read/write elementary files directly |
| `AT+CPBS=?` | ✅ | Phonebook stores: `SM` SIM · `ME` module · `DC` dialled · `MC` missed · `RC` received · `EN` emergency |
| `AT+CPBR` / `AT+CPBW` | 📖 | Read / write phonebook entries |
| `AT+CLCK` / `AT+CPWD` | 📖 | Facility lock / change password |
| `AT+QSIMDET` / `AT+QSIMSTAT` | 📖 | SIM hot-plug detection and status |

### Network

| Command | | Purpose |
|---|---|---|
| `AT+COPS?` | ✅ | Current operator and access technology (`7` = LTE) |
| `AT+COPS=?` | 📋 | **Scan every carrier in range.** Blocks ~60 s |
| `AT+CREG?` `AT+CGREG?` `AT+CEREG?` | ✅ | CS / GPRS / **EPS** registration. On LTE-only, `CEREG` is the one |
| `AT+CFUN` | ✅ | `0` minimum (radio off, SIM alive) · `1` full · `4` airplane |
| `AT+QNWINFO` | ✅ | Tech / operator / band / channel in one line |
| `AT+QCFG="band"` | ✅ | Read/set band mask — lock a band for range testing |
| `AT+QCFG="nwscanmode"` / `"nwscanseq"` | 📖 | Restrict RAT / set search order |
| `AT+QENG="servingcell"` | ✅ | **Full serving-cell dump** — MCC, MNC, Cell ID, PCI, EARFCN, band, bandwidth, TAC, RSRP, RSRQ, RSSI, SINR |
| `AT+QENG="neighbourcell"` | 📖 | Neighbour cells |

### Signal and diagnostics

| Command | | Purpose |
|---|---|---|
| `AT+CSQ` | ✅ | RSSI as an index — dBm = `−113 + 2n` |
| `AT+QCSQ` | ✅ | Richer: RSSI, RSRP, SINR, RSRQ together |
| `AT+QTEMP` | ✅ | **Three temperature sensors**, °C |
| `AT+CBC` | 📋 | Supply voltage as the module sees it |
| `AT+CEER` | 📋 | Extended error report for the last failure |
| `AT+QJDCFG` | ✅ | **Jamming detection** — per-metric thresholds and a URC on detect |
| `AT+QADC` | ✅ | **Two readable ADC channels** |
| `AT+QLINUXCPU` | 📋 | Internal CPU load (the module runs Linux inside) |

### Packet data

| Command | | Purpose |
|---|---|---|
| `AT+CGDCONT` | ✅ | Define/read PDP contexts. **Modern carriers auto-provision these — don't hand-set the APN** |
| `AT+CGPADDR` | 📋 | IP assigned to a context |
| `AT+CGCONTRDP` | 📋 | Full dynamic params **including the network's DNS servers** |
| `AT+QICSGP` | ✅ | Quectel context config — 16 contexts, APN/user/pass/auth |
| `AT+QIACT` / `AT+QIDEACT` | 📖 | Activate / deactivate |
| `AT+CGEQOS` `+CGEQREQ` `+CGQMIN` | 📋 | QoS profiles |
| `AT+CGTFT` | 📋 | Traffic flow templates |

### SMS

| Command | | Purpose |
|---|---|---|
| `AT+CMGF=1` | ✅ | Text mode (`0` = PDU mode) |
| `AT+CMGL="ALL"` | ✅ | List. ⚠️ Mutates read state; `,1` form is **ERROR** here |
| `AT+CMGR=<i>` | ✅ | Read one; also marks it read |
| `AT+CMGS="<num>"` | ✅ | Send — body then Ctrl-Z. ⚠️ Wait for `>` |
| `AT+CMGD=<i>` | 📖 | **Delete** — required, the store caps at 255 |
| `AT+CPMS?` | ✅ | Storage selection and occupancy |
| `AT+CNMI` | ✅ | New-message indication. `2,1,0,0,0` = store and emit `+CMTI` |
| `AT+CSCA?` | ✅ | Service centre address — must be set or sends fail |
| `AT+CSCS` | ✅ | TE charset, `GSM` or `UCS2` |
| `AT+CSMP` | ✅ | First octet / validity / **DCS** |
| `AT+CMGW` / `AT+CMSS` | 📖 | Write to storage / send from storage |

### The module's own IP stack

| Command | | Purpose |
|---|---|---|
| `AT+QIOPEN` | ✅ | Open a socket — 12 available: `TCP`, `UDP`, **`TCP LISTENER`**, `UDP SERVICE` |
| `AT+QISEND` / `AT+QIRD` / `AT+QICLOSE` | 📖 | Send / read / close |
| `AT+QPING` | ✅ | ICMP ping from the module |
| `AT+QNTP` | ✅ | NTP client |
| `AT+QIDNSCFG` / `AT+QIDNSGIP` | 📖 | Configure DNS / resolve a name |

**`TCP LISTENER` means the module can be a server** — whether that's reachable depends on
whether your carrier NATs you (most do).

### Application protocols, in-module

| Command | | Purpose |
|---|---|---|
| `AT+QHTTPCFG` | ✅ | HTTP(S) client — TLS context, headers, content type, basic auth, custom headers |
| `AT+QHTTPURL` `QHTTPGET` `QHTTPPOST` `QHTTPREAD` | 📖 | Set URL, GET, POST, read response |
| `AT+QMTCFG` | ✅ | MQTT client — v3.1/3.1.1, TLS, keepalive, clean session, **will messages** |
| `AT+QMTOPEN` `QMTCONN` `QMTSUB` `QMTPUB` | 📖 | Open, connect, subscribe, publish |
| `AT+QFTPCFG` | ✅ | FTP(S) — account, binary/ASCII, active/passive, TLS, resume |
| `AT+QSSLCFG` | ✅ | **Full TLS stack** — version, 28 cipher suites by ID, CA cert, client cert+key, PSK, SNI, ALPN, session cache, DTLS |

⚠️ `QSSLCFG` also exposes a family of `ignore*` verification overrides. Those are how
devices ship without validating certificates. Leave them alone.

### File system

| Command | | Purpose |
|---|---|---|
| `AT+QFLST` | ✅ | List files stored on the module |
| `AT+QFLDS` | ✅ | Free/total space on a volume |
| `AT+QFUPL` / `AT+QFDWL` | 📖 | Upload / download over the AT port |
| `AT+QFOPEN` `QFREAD` `QFWRITE` `QFCLOSE` `QFDEL` | 📖 | Handle operations |

### Power management

| Command | | Purpose |
|---|---|---|
| `AT+CFUN=0/1/4` | ✅ | The cheap way to park the radio |
| `AT+CPSMS` | ✅ | **PSM** — negotiate deep sleep with the network (TAU + active timers) |
| `AT+CEDRXS` | ✅ | **eDRX** — longer paging cycles, sleep between them |
| `AT+QSCLK` | 📋 | Slow clock / sleep when the host is idle |
| `AT+QPOWD` | ✅ | Controlled power-down — `0` immediate, `1` graceful network detach |

### Time

| Command | | Purpose |
|---|---|---|
| `AT+QLTS=2` | ✅ | **Network time — no data session, no NTP.** Returns local time with UTC offset and a validity flag |
| `AT+CCLK?` | 📖 | Module RTC |
| `AT+QNTP` | ✅ | NTP over a data context |

`AT+QLTS` is underrated: for a battery device that sleeps for days, the carrier is a better
clock source than anything else available, and it costs no data.

### Audio / voice

Present in silicon on voice-capable variants. Useless without a codec and PCM routing.

| Command | | Purpose |
|---|---|---|
| `AT+QAUDMOD` | ✅ | Audio mode 0–5 |
| `AT+QDAI` | 📋 | Digital audio interface (PCM format, master/slave) |
| `AT+QAUDPLAY` `QAUDSTOP` `QAUDRD` | 📋 | Play / stop / record on the module |
| `AT+QTONEDET` `QDTMFDETSET` | 📋 | **DTMF detection** — decode touch-tones from a call |
| `AT+QTTS` `AT+QWTTS` `AT+QTTSETUP` | 📋 | **Text-to-speech, in the module** |
| `AT+QEEC` | 📋 | Echo cancellation tuning |
| `AT+CLVL` / `AT+CMUT` | 📋 | Volume / mute |
| `AT+CLCC` | 📋 | List current calls |

### Miscellaneous

| Command | | Purpose |
|---|---|---|
| `AT+CUSD` | ✅ | **USSD** — `*123#`-style codes. Note many MVNOs acknowledge and never answer |
| `AT+CMUX` | 📋 | **Multiplex one serial port into several channels** — how you get concurrent AT and data without extra ports |
| `AT+QFLOWCOUNT` / `AT+QGDCNT` | ✅ | Data counters. ⚠️ These track the **module's own** PDP contexts, not traffic over `qmi_wwan`/RMNET — don't use them as a usage meter for a host-driven session |
| `AT+QIPPTCFG` | 📋 | IP passthrough — bridge the cellular IP straight to the host |
| `AT+QFILTER` | 📋 | Packet filtering |
| `AT+QURCCFG="urcport"` | ✅ | Where URCs are emitted |
| `AT+QFOTADL` | 📋 | ⚠️ DFOTA — OTA firmware update from a URL. As dangerous as it sounds |
| `AT+CLAC` | ✅ | List supported commands. ⚠️ Incomplete — see the top |

### Qualcomm proprietary (`$QC*`)

Roughly 49 `$QC*` commands sit under Quectel's layer, exposed by the Qualcomm baseband.
Undocumented, heavily overlapping the `+Q` equivalents, and **not a stable interface** — a
firmware update can change them without notice. Prefer the `+Q` or standard `+C` form.

For recognition only: `$QCSQ` (signal), `$QCRSRP`/`$QCRSRQ` (LTE measurements),
`$QCBANDPREF` (band preference), `$QCSYSMODE`, `$QCPDPP` (PDP auth),
`$QCSIMSTAT`/`$QCPINSTAT`, `$QCANTE` (**antenna detection**), `$QCCTM` (thermal
mitigation), `$QCDRX`, `$QCPWRDN`, `$QCCLAC`.

---

## Talking to it

Four USB serial ports appear (`ttyUSB0-3`) plus a QMI control device. **`ttyUSB2` is
normally the AT port**; NMEA streams on another when GNSS is enabled.

On a system with no python or socat, busybox `microcom` is enough:

```sh
printf 'AT+CPIN?\r' | microcom -t 2000 -s 115200 /dev/ttyUSB2
```

⚠️ `microcom` takes a lock at `/var/lock/LCK..ttyUSB2`. Two concurrent invocations — or one
that dies badly — leave a stale lock, and every later call fails with `can't create … File
exists`, which reads exactly like the modem having vanished. `rm -f` it.

⚠️ If you open the port yourself, open it `O_NONBLOCK` (so you don't hang on carrier detect)
and then **clear that flag** — writes to a non-blocking tty fail with `EAGAIN`.

⚠️ **ModemManager will claim these ports** if it's running, and it reconfigures things behind
you — it changed `AT+CNMI` on this unit and left it changed. It also ingests SMS off the
modem into its own store. Pick one owner.

---

## License

CC0 / public domain. Corrections welcome.
