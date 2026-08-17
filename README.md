# Quectel EC25 — AT command reference and field notes

Notes from driving a **Quectel EC25-AFX** (mini-PCIe, firmware `EC25AFXGAR07A04M1G`) on a
US carrier over USB: SMS, MMS send *and* receive, LTE data, and GNSS.

Most of this is in Quectel's manuals somewhere. What is here that isn't there: which
commands actually behave differently from the documentation, and the specific failures that
cost time. Every ✅ below was run on real hardware.

Legend — ✅ verified on hardware · ⚠️ works with a caveat · ✗ tried and refused ·
📋 listed by `AT+CLAC` on this firmware · 📖 documented but not exercised here

**What "verified" covers here:** SMS (send/receive, UCS2), MMS (send *and* retrieve, full
PDU encode/decode), LTE data, GNSS, outbound **and** inbound VoLTE with in-module TTS,
auto-answer, PSM/eDRX negotiation, MQTT, and the module's whole IP stack — DNS, ping, NTP,
TCP, UDP, TCP listener, HTTP, HTTPS, raw TLS, FTP and on-module file storage. Where
something was tried and *failed*, the failure and its cause are written down rather than
omitted.

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

## Voice, VoLTE and TTS

The module has a **text-to-speech engine**, and it can speak into a live call. Both the
synthesis and the uplink audio path live inside the module, so **the host needs no audio
hardware at all** — no codec, no speaker, no microphone. Verified: a 254-character message
was spoken into a VoLTE call and heard clearly by the recipient.

### ⚠️ On an LTE-only network, `ATD` fails until you force IMS on

Out of the box, dialling returns a bare `ERROR` — instantly, with nothing reaching the
network:

```
AT+QCFG="ims"      ->  +QCFG: "ims",0,1     # 0 = follow network  ->  ATD FAILS
AT+QCFG="ims",1                             # force IMS on
AT+QCFG="ims"      ->  +QCFG: "ims",1,1     #                     ->  ATD WORKS
```

The cause is `AT+CEMODE?` reporting **`1`** — "CS/PS mode 1", meaning the module intends to
place voice calls by **falling back to 2G/3G**. On a carrier that has shut those down there
is nothing to fall back to, so it refuses before dialling.

⚠️ **`AT+CEMODE=3` (PS mode 1 — voice over IMS) is rejected** with
`+CME ERROR: operation not supported`, even though `AT+CEMODE=?` advertises `(0-3)`. You
cannot fix it the obvious way; forcing `QCFG="ims",1` is what works.

Things that were *not* the problem, and are worth ruling out quickly:
`AT+QCFG="volte/disable"` was already `0` (not disabled), the NV item
`/nv/item_files/ims/IMS_enable` already read `01`, and `AT+QMBNCFG="list"` showed the
carrier's VoLTE profile already **selected and activated**.

`AT+CEER` distinguishes the two failure modes: a refusal before dialling versus a call that
reached the network and later ended — different category values.

### Speaking into a call

```
AT+QWTTS=<ulmute>,<dlmute>,<mode>[,<text>]
  <ulmute>  0 = mute uplink, 1 = unmute   <- 1 is what lets the far end hear it
  <dlmute>  0 = mute downlink, 1 = unmute
  <mode>    0 = stop · 1 = start, UCS2 · 2 = start, ASCII/GBK direct
  <text>    max 960 bytes
Reports  +QWTTS: 0  when finished.
```

Local playback is the sibling: `AT+QTTS=<mode>,<text>` → `+QTTS: 0`.
`AT+QTTSETUP=<type>,<param>,<value>` adjusts speed and volume.

### Audio error codes

| `+CME ERROR` | Meaning |
|---|---|
| 901 | Audio unknown error |
| 902 | Audio invalid parameters |
| 903 | **Audio operation not supported** — `QWTTS` with no active call |
| 904 | **Audio device busy** — TTS already playing |

### Measured speech rate

| Context | sec/char |
|---|---|
| Local `QTTS` playback | **0.064** |
| **In-call `QWTTS`** | **0.090** |

In-call is ~40 % slower — the network paces the stream at real speech rate. **Budget from
the in-call figure**: 960 bytes is roughly **85 seconds** of speech, not 60.

### ⚠️ Two traps when detecting call state

**`+CLCC` lists data contexts as calls.** Expect permanent entries like:

```
+CLCC: 1,1,0,1,0,"",128
             ^ ^
             | mode=1 = DATA (voice is 0)
             stat=0 = "active"
```

These are PDP contexts and they are **always there**. Code that checks only `<stat>` matches
them and reports a connected call that does not exist. **Require `<mode>==0` first**, then
read `<stat>` (0=active, 2=dialing, 3=alerting, 4=incoming).

During a real inbound call the list looked like this — note the genuine call is **not
first**:

```
+CLCC: 1,1,0,1,0,"",128                <- data context
+CLCC: 2,1,0,1,0,"",128                <- data context
+CLCC: 4,1,0,0,0,"<number>",145        <- the actual voice call
+CLCC: 3,1,0,1,0,"",128                <- data context
```

**`ATD` blocks until the call connects** — it returned `OK` after 3.6 s, immediately
followed by `+COLP`. Polling `AT+CLCC` while waiting is both unnecessary and harmful: every
command between dial and speech is dead air on the uplink. Read the stream and fire on
`+COLP`:

```
ATD<number>;                     # ';' makes it a VOICE call, not data
  ... blocks ...
OK
+COLP: "<number>",129,,,         # far end connected -> speak now
AT+QWTTS=1,1,2,"..."
+QWTTS: 0                        # finished
```

`+COLP` requires `AT+COLP=1`. `AT+CLIP=1`, `AT+CRC=1`, `AT+CSSN=1,1` give the inbound and
supplementary-service URCs.

### Inbound calls and unattended answering

Inbound is a **separate provisioning question** from outbound and can fail on its own.

```
+CRING: VOICE                     with AT+CRC=1 (plain "RING" otherwise)
+CLIP: "<number>",145,"",0,,0     caller ID, needs AT+CLIP=1
ATA                               answer -> OK
AT+CPAS                        ->  +CPAS: 4     (3 = ringing, 4 = in call, 0 = idle)
AT+QWTTS=1,1,2,"..."           ->  +QWTTS: 0
ATH
```

`ATS0=<n>` auto-answers after n rings, so the module can pick up and speak with no host
interaction at all. ⚠️ **Restore `ATS0=0`** — left armed it silently answers every call.

⚠️ **A call diverting to voicemail may mean the module simply has no service**, not that
inbound is unprovisioned. Confirm `AT+QNWINFO` shows a real cell before concluding anything
about call handling — a network with an unreachable subscriber forwards immediately.

## Multiplexing (CMUX) — likely a dead end over USB

`AT+CMUX` would let several logical channels share one AT port, so a daemon could hold the
URC stream while other code issues commands. Two things block it:

**The module refuses on the USB AT port:**

```
AT+CMUX=?          +CMUX: (0),(0-2),(1-8),(1-32768),(1-255),(0-100),(2-255),(1-255),(1-7)
AT+CMUX=0,0,5,127  +CME ERROR: operation not allowed
```

⚠️ A **refusal, not a syntax error** — the test form advertises full parameter ranges and
then declines the write. Plausible reason: CMUX exists to multiplex a single physical UART,
and over USB the module already exposes four independent ttyUSB ports. Unverified, since a
USB adapter exposes no UART pins to test against.

**And Linux needs a mux driver you may not have.** Terminating GSM 07.10 requires the
`n_gsm` line discipline (`CONFIG_N_GSM`), which produces `/dev/gsmtty*`. Check before
planning around it:

```sh
zgrep N_GSM /proc/config.gz     # or grep your /boot/config-*
cat /proc/tty/ldiscs            # n_gsm appears here when available
```

Two mainstream kernels checked for this document had `# CONFIG_N_GSM is not set`.

⚠️ **Have a recovery route staged before experimenting.** If the module *does* enter mux
mode, the port stops answering plain AT — including the command to undo it. On a
4-port composite there is usually a **second AT port** to issue `AT+CFUN=1,1` from;
otherwise a USB deauthorize/reauthorize via
`/sys/bus/usb/devices/<dev>/authorized` resets it.

## Power saving — PSM and eDRX

```
AT+CEREG=4                                enable extended registration reporting
AT+CPSMS=1,,,"<TAU bits>","<active bits>" request
AT+CFUN=0 / AT+CFUN=1                     re-attach so it rides the ATTACH
AT+CPSMS?  /  AT+CEREG?                   read back what was GRANTED
```

Timer octets are 3GPP TS 24.008: 3 bits of unit, 5 bits of value.

| T3412 ext (TAU) | | T3324 (Active Time) | |
|---|---|---|---|
| `000` | 10 min | `000` | 2 sec |
| `001` | 1 hour | `001` | 1 min |
| `010` | 10 hours | `010` | 6 min |
| `011` | 2 sec | `111` | deactivated |
| `100` | 30 sec | | |
| `101` | 1 min | | |
| `110` | 320 hours | | |
| `111` | deactivated | | |

✳️ **Read the granted values, not your request.** One network here granted a requested
1-hour TAU but **re-expressed it** — asked as `1 × 1 hour` (`00100001`), granted as
`6 × 10 min` (`00000110`). Same duration, different encoding. The granted timers appear in
extended `+CEREG` (the last two fields) as well as `+CPSMS?`.

⚠️ **eDRX may be silently refused.** `AT+CEDRXS=1,4,"0101"` returns `OK` while
`AT+CEDRXRDP` continues to report `0` — requested, not granted. Note also that
`AT+CEDRXS?` returns `ERROR` on some firmware while `AT+CEDRXRDP` works; read state with
the latter.

⚠️ **PSM can cost you network service, silently.** After a PSM test the module reported
`+CGATT: 1` (claiming attached) while `AT+QNWINFO` said `No Service` and `AT+CEREG?` timed
out. **`AT+CGATT` is not a reachability check.** An `AT+CFUN=0/1` cycle restores it.

## MQTT in the module

The module is a real MQTT endpoint — it holds the connection itself, so it keeps working
while the host that drives it is busy or asleep.

```
AT+QMTCFG="version",0,4                    MQTT 3.1.1
AT+QMTCFG="keepalive",0,60
AT+QMTOPEN=0,"broker.example.com",1883  ->  +QMTOPEN: 0,0
AT+QMTCONN=0,"<client-id>"              ->  +QMTCONN: 0,0,0
AT+QMTSUB=0,1,"some/topic",0            ->  +QMTSUB: 0,1,0,0

incoming arrives unsolicited:
+QMTRECV: 0,0,"some/topic","payload"

AT+QMTPUB=0,0,0,0,"some/topic"          then body + Ctrl-Z (0x1A)
                                        ->  +QMTPUB: 0,0,0
AT+QMTCONN?                             ->  +QMTCONN: 0,3     (3 = connected)
AT+QMTDISC=0
```

⚠️ `AT+QMTPUB` takes its payload the same way `AT+CMGS` does — write, then **Ctrl-Z** — so
the same prompt race applies.

⚠️ Needs an active PDP context. The MQTT stack runs over the module's own data session, not
over your host's network.

## The module as an IP host

It is a complete TCP/IP stack, not a bridge. Verified end to end with **no host data
session at all** — only the module's own context — so none of it was being carried by the
machine driving it:

```
AT+QICSGP=1,1,"<apn>","","",1        configure the module's own context
AT+QIACT=1                            activate it
AT+QIDNSGIP=1,"example.com"       ->  +QIURC: "dnsgip"      DNS
AT+QPING=1,"1.1.1.1",4,4          ->  +QPING: 0,…           ICMP
AT+QNTP=1,"time.google.com",123   ->  +QNTP: 0              sets the module clock
AT+QIOPEN=1,0,"TCP","host",80,0,0                           TCP client
AT+QIOPEN=1,1,"UDP","host",53,0,0                           UDP
AT+QIOPEN=1,2,"TCP LISTENER","127.0.0.1",0,2020,0           server socket
AT+QHTTPGET=80                    ->  +QHTTPGET: 0,200      HTTP
AT+QHTTPREADFILE="ufs:page.html",80                         response straight to flash
AT+QFLDS="UFS"                    ->  ~16 MB filesystem
```

⚠️ **The module's stack and the host's `qmi_wwan`/RMNET path are separate.** Bytes sent via
`QIOPEN` never appear on the host's `wwan` interface and are not counted by its statistics —
which is also why `AT+QGDCNT` reads near-zero after a host-driven data session.

### ⚠️ HTTPS silently requires SNI

The most expensive finding here. With TLS otherwise correctly configured:

```
AT+QSSLCFG="sslversion",1,4 · "seclevel",1,0 · "ciphersuite",1,0XFFFF
AT+QHTTPGET=80   ->  +QHTTPGET: Http unknown error

AT+QSSLCFG="sni",1,1                    <- the fix
AT+QHTTPGET=80   ->  +QHTTPGET: 0,200
```

**SNI is off by default**, and essentially every modern host sits behind a CDN or shared IP
that needs it. The error mentions neither TLS nor SNI. Set it first when debugging HTTPS.

### ⚠️ `AT+QIOPEN`'s access_mode decides how you read

The last parameter, and getting it wrong looks like a broken socket:

| mode | behaviour |
|---|---|
| `0` buffer | data is buffered — read with **`AT+QIRD`** |
| `1` direct push | data is **pushed to the port unsolicited**; `AT+QIRD` returns **`ERROR`** |
| `2` transparent | the port becomes a raw pipe |

In mode 1 the data arrives perfectly and `QIRD` still errors, because there is nothing
buffered to read — it already went out the port.

### ⚠️ Two more contracts worth knowing

**`AT+QISEND=<id>,<len>` is exact.** Announce a length, then write precisely that many
bytes — CRLFs included. Announce the wrong number and the socket waits.

**`AT+QHTTPREAD` consumes the response.** Call it and then `AT+QHTTPREADFILE` and you get
`Http no get/post request` — there is nothing left. Pick one: out the port, or to a file.

**`TCP LISTENER` opens successfully**, so the module can accept inbound connections — but on
a carrier that NATs you (most do; we saw `172.56.x` carrier-NAT addressing) it will not be
reachable from the public internet.

### Raw TLS sockets, separate from HTTPS

`AT+QSSLOPEN` gives a TLS client independent of the HTTP stack:

```
AT+QSSLCFG="sslversion",2,4 · "seclevel",2,0 · "ciphersuite",2,0XFFFF · "sni",2,1
AT+QSSLOPEN=1,2,0,"example.com",443,0  ->  +QSSLOPEN: 0,0
AT+QSSLSEND=0,<len>   + exactly <len> bytes  ->  SEND OK
AT+QSSLRECV=0,400                      ->  +QSSLRECV: 400  HTTP/1.1 200 OK …
```

⚠️ Needs the same **SNI** setting as HTTPS.

### FTP — and a local-path error that impersonates a network failure

Control channel and file download both work:

```
AT+QFTPCFG="contextid",1 · "account","<user>","<pass>" · "filetype",1 · "transmode",1
AT+QFTPOPEN="<host>",21        ->  +QFTPOPEN: 0,0
AT+QFTPPWD                     ->  +QFTPPWD: 0,/
AT+QFTPGET="readme.txt","COM:" ->  +QFTPGET: 0,379      379 bytes over the data channel
```

⚠️ **Error `613` is a LOCAL FILE PATH problem, not a network one.** Downloading to a
`ufs:` path fails with `613`, which looks exactly like passive FTP failing through carrier
NAT — a plausible and completely wrong diagnosis. The identical download to **`"COM:"`**
(stream out the serial port) returns `0,<bytes>`, proving the data connection is fine.
**Target `COM:` first to separate local-path failures from network failures.**

⚠️ **The on-module filesystem path convention is inconsistent between subsystems.**
`AT+QHTTPREADFILE="ufs:page.html"` succeeds while `AT+QFTPGET=…,"ufs:…"` returns `613`,
`AT+QFDEL="ufs:page.html"` gives `Invalid input value`, and `AT+QFLST` produced no listing
for any pattern tried — even though `AT+QFLDS="UFS"` reports the volume correctly. Treat
on-module storage as usable only via whichever subsystem you have actually tested.

⚠️ Plain FTP is widely retired, so pick a test target carefully — `ftp.gnu.org` has port 21
closed and the module correctly reports `+QFTPOPEN: 606,0`.

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
| `AT+QISEND` / `AT+QIRD` / `AT+QICLOSE` | ✅ | Send / read / close |
| `AT+QPING` | ✅ | ICMP ping from the module |
| `AT+QNTP` | ✅ | NTP client |
| `AT+QIDNSCFG` / `AT+QIDNSGIP` | ✅ | Configure DNS / resolve a name |

**`TCP LISTENER` means the module can be a server** — whether that's reachable depends on
whether your carrier NATs you (most do).

### Application protocols, in-module

| Command | | Purpose |
|---|---|---|
| `AT+QHTTPCFG` | ✅ | HTTP(S) client — TLS context, headers, content type, basic auth, custom headers |
| `AT+QHTTPURL` `QHTTPGET` `QHTTPREAD` `QHTTPREADFILE` | ✅ | Set URL, GET, POST, read response |
| `AT+QMTCFG` | ✅ | MQTT client — v3.1/3.1.1, TLS, keepalive, clean session, **will messages** |
| `AT+QMTOPEN` `QMTCONN` `QMTSUB` `QMTPUB` `QMTDISC` | ✅ | Open, connect, subscribe, publish |
| `AT+QFTPCFG` | ✅ | FTP(S) — account, binary/ASCII, active/passive, TLS, resume |
| `AT+QFTPOPEN` `QFTPPWD` `QFTPGET` `QFTPCLOSE` | ✅ | Connect, navigate, download, close |
| `AT+QSSLOPEN` `QSSLSEND` `QSSLRECV` `QSSLCLOSE` | ✅ | **Raw TLS sockets**, independent of the HTTP stack |
| `AT+QSSLCFG` | ✅ | **Full TLS stack** — version, 28 cipher suites by ID, CA cert, client cert+key, PSK, SNI, ALPN, session cache, DTLS |

⚠️ `QSSLCFG` also exposes a family of `ignore*` verification overrides. Those are how
devices ship without validating certificates. Leave them alone.

### File system

| Command | | Purpose |
|---|---|---|
| `AT+QFLST` | ⚠️ | Meant to list files — returned **no listing** for every pattern tried here |
| `AT+QFLDS` | ✅ | Free/total space on a volume |
| `AT+QFUPL` / `AT+QFDWL` | 📖 | Upload / download over the AT port |
| `AT+QFOPEN` `QFREAD` `QFWRITE` `QFCLOSE` `QFDEL` | 📖 | Handle operations |

### Power management

| Command | | Purpose |
|---|---|---|
| `AT+CFUN=0/1/4` | ✅ | The cheap way to park the radio |
| `AT+CPSMS` | ✅ | **PSM** — granted on a live carrier; read the *granted* timers, not your request |
| `AT+CEDRXS` | ⚠️ | **eDRX** — returns `OK` but may be **silently refused**; check `AT+CEDRXRDP`. `AT+CEDRXS?` errors on this firmware |
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
| `AT+QTTS` / `AT+QWTTS` / `AT+QTTSETUP` | ✅ | **TTS** — local playback and speaking into a live call |
| `AT+QDAI` | 📋 | Digital audio interface (PCM format, master/slave) |
| `AT+QAUDPLAY` `QAUDSTOP` `QAUDRD` | 📋 | Play / stop / record on the module |
| `AT+QTONEDET` `QDTMFDETSET` | 📋 | **DTMF detection** — decode touch-tones from a call |
| `AT+QTTS` `AT+QWTTS` `AT+QTTSETUP` | 📋 | **Text-to-speech, in the module** |
| `AT+QEEC` | 📋 | Echo cancellation tuning |
| `AT+CLVL` / `AT+CMUT` | 📋 | Volume / mute |
| `AT+CLCC` | ✅ | List current calls — ⚠️ **includes data contexts**, filter on `<mode>==0` |

### Miscellaneous

| Command | | Purpose |
|---|---|---|
| `AT+CUSD` | ✅ | **USSD** — codes are accepted; note many MVNOs acknowledge with "a message will be sent" and never send one |
| `AT+CMUX` | ✗ | Multiplexing — **refused on the USB AT port** (`operation not allowed`), and Linux needs `CONFIG_N_GSM` |
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

## Sources

**The primary source for this document is the hardware.** Everything marked ✅ was run
against a real EC25-AFX on a live carrier network, and the notes exist precisely where
observed behaviour diverged from the written specs. Where a figure came from a document
rather than a measurement, it is cited below.

### Quectel documentation

- [EC25 Series LTE Standard Specification, V2.7](https://quectel.com/content/uploads/2024/03/Quectel_EC25_Series_LTE_Standard_Specification_V2.7-1.pdf)
  — the per-variant feature table: supported bands, VoLTE/SMS/QMI availability, carrier
  certifications, and which variants are data-only. Worth reading before buying a variant;
  the naming does not make the differences obvious.
- [EC25 & EC21 GNSS AT Commands Manual, V1.1](https://sixfab.com/wp-content/uploads/2018/09/Quectel_EC25EC21_GNSS_AT_Commands_Manual_V1.1.pdf)
  — the `AT+QGPS*` family, NMEA output configuration, and gpsOneXTRA.
- [EC25 Mini PCIe Hardware Design, V1.1](https://www.dragino.com/downloads/downloads/datasheet/other_vendors/EC25/Quectel_EC25_Mini_PCIe_Hardware_Design_V1.1.pdf)
  — the three antenna connectors, the **2.85 V GNSS bias** figure, active vs passive antenna
  requirements, and the B13/B14 harmonic recommendation that the GNSS section above works
  through.
- [LTE EC25 series](https://www.quectel.com/product/lte-ec25-series/) ·
  [LTE EC25 Mini PCIe series](https://www.quectel.com/product/lte-ec25-mini-pcie-series/)
  — product pages; GNSS accuracy (<2.5 m CEP-50, Qualcomm IZat Gen8C Lite) and TTFF figures.

### Protocol specifications

The MMS section is an implementation of these; neither is Quectel's and neither is
discoverable from the AT command set.

- **OMA Multimedia Messaging Service Encapsulation Protocol** (OMA-TS-MMS_ENC) — the
  M-Send.req / M-Send.conf / M-Notification.ind / M-Retrieve.conf PDU structures, header
  field codes, and response status values such as `0xE2`.
- **WAP-230-WSP**, WAP Forum Wireless Session Protocol — the binary encoding primitives:
  uintvar, text-string, **quoted-string**, value-length, short-integer, the well-known
  content-type and header-field assignments, and the multipart body layout. The
  quoted-string requirement for `Content-ID` comes from here.
- **3GPP TS 27.007** (AT commands for UE) and **TS 27.005** (SMS AT commands) — the
  standard `+C*` commands. Where this document says a standard command behaves
  non-standardly, that is the baseline it is measured against.

### Notes on provenance

- `AT+CLAC` was captured from firmware `EC25AFXGAR07A04M1G`. Command availability and the
  `$QC*` set **vary by variant and firmware**; treat the 📋 entries as "present on that
  build" rather than universal.
- Behaviour attributed to carriers (APN auto-provisioning, MMSC on carrier-private DNS,
  MVNOs not answering USSD, ICMP blocked to the MMSC) was observed on **one MVNO on one US
  network**. The mechanisms generalise; the specifics may not.
- No claim here rests on a vendor marketing page.

## License

CC0 / public domain. Corrections welcome.
