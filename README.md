# Alfano 6 — Bluetooth Protocol Specification (Reverse-Engineered)

**Status:** Working draft v0.3 · Reverse-engineered from three HCI snoop captures (full memory dump + single-session download + full multi-session driven dump) for interoperability with owned hardware (LapWing / PerchWerks).
**Device:** Alfano 6 (kart lap timer / data logger), firmware shown as `1T`, "Professional" mode.
**Transport:** Bluetooth **Classic** (BR/EDR) — **SPP / RFCOMM**. *Not* BLE. Web Bluetooth cannot reach this device; see [§9 Implementation](#9-implementation-notes).

> **Confidence legend used throughout:**
> ✅ **Confirmed** — directly observed and verified in the capture.
> 🟡 **Inferred** — strongly implied by structure/correlation, not yet proven.
> ❓ **Unknown** — needs another capture to resolve.

---

## 1. Capture provenance

| | |
|---|---|
| Source | Android HCI snoop log (`btsnoop_hci.log`), BTSnoop v1, H4 (HCI UART) |
| Link | ACL handle `0x06`, bonded Classic connection |
| Duration | ~307 s, full session-memory download |
| Device state | **Engine OFF, no GPS lock** ("No GPS Signal" on screen) — live telemetry values are idle/placeholder |
| Active profile | `Dame` |
| Active track | `*P* SHORT OKC` (Short config, Orlando Kart Center) |

A **second capture** ("only download last session" app setting, same engine-off state) was taken and used to validate the protocol. It performed the identical handshake and read **only** the 4 config pages + page `0x00F9` + one record fetch (`4E26B2`) — the same record ID and contents as the first capture. This **confirms** the page-numbering semantics, the always-first config pages, and the stability of the record ID (see [§4.1](#41-0x2b-read_page-page-sequence-) and [§7.1](#71-download-modes-)).

**Implication:** All *framing* is mapped and now cross-validated across two sessions. All *live-value semantics* (RPM/speed/GPS/temps scaling and offsets) remain **pending an engine-on + GPS-lock capture** — but note the realistic caveat in [§8](#8-known-unknowns--next-captures): the official apps expose **no live-telemetry view at all**, so a continuous live stream may simply not exist over this interface.

---

## 2. Transport layer ✅

| Parameter | Value |
|---|---|
| Protocol | RFCOMM over L2CAP (PSM `0x0003`) |
| Service | Serial Port Profile (SPP) |
| SPP service UUID | `00001101-0000-1000-8000-00805F9B34FB` |
| RFCOMM server channel | **6** (DLCI 12) |
| Flow control | **Credit-based** (CL field `0xF`/`0xE` in PN) |
| Max frame size (N1) | 990 bytes |
| Initial credits | 7 (host→device), 2 (device→host) |

The phone performs SDP discovery (PSM `0x0001`) first, then opens RFCOMM. A client using `createRfcommSocketToServiceRecord(SPP_UUID)` (Android) or an SDP-aware RFCOMM connect (desktop) will resolve channel 6 automatically — **you do not need to hardcode the channel.**

### RFCOMM session setup (observed order)
1. SABM/UA on DLCI 0 (multiplexer start)
2. **PN** (Parameter Negotiation) for DLCI 12 → credit-based FC, N1=990
3. SABM/UA on DLCI 12 (open data channel)
4. **MSC** modem-status handshake (both directions)
5. Data exchange (UIH frames on DLCI 12)
6. DISC/UA on teardown

> Implementers using a high-level SPP socket API (Android `BluetoothSocket`, Python `pybluez`, Rust `bluetooth-serial-port`/`bluer`) get all of the above for free — you read/write a **byte stream** and never touch RFCOMM framing directly. The credit octets, FCS, and DLCI are handled by the OS Bluetooth stack.

---

## 3. Application frame format

The application protocol rides on the SPP byte stream. There are two directions:

- **TX** = host → Alfano (commands)
- **RX** = Alfano → host (responses / records / live data)

### 3.1 Command frame (TX) ✅

**Every command is a fixed 258-byte frame:**

```
┌────────┬──────────────────────────┬───────────────────┬──────────┐
│ opcode │ arguments                │ zero padding       │ checksum │
│ 1 byte │ N bytes                  │ to fill 257 bytes  │ 1 byte   │
└────────┴──────────────────────────┴───────────────────┴──────────┘
  byte 0   bytes 1 .. (256-?)          ...zeros...          byte 257
└──────────────────── 257 bytes covered by checksum ───────┘
                                                           checksum byte
```

- Total length: **258 bytes**, always.
- Bytes after the arguments are zero-padded out to offset 256.
- **Checksum (byte 257) = XOR of all 257 preceding bytes.** ✅ Verified across 20 frames with varied payloads (20/20).

```python
def build_command(opcode: int, args: bytes = b"") -> bytes:
    body = bytes([opcode]) + args
    body = body.ljust(257, b"\x00")        # pad to 257
    checksum = 0
    for b in body:
        checksum ^= b
    return body + bytes([checksum])         # 258 bytes total
```

### 3.2 Response frame (RX) — variable length ✅

Responses are **variable length** (observed 1–514 bytes per logical message). They are delivered as the SPP byte stream; one logical response may arrive split across multiple OS read() calls (and was split across multiple RFCOMM UIH frames in the capture). Reassembly is by message structure, not by frame boundary.

Two response shapes matter:
- **Fixed-magic records** beginning `80 FF FE FD …` — telemetry/session records (see [§6](#6-telemetry-record-80fffefd)).
- **Untagged blocks** — 1-byte ack (`FF`), the 16-byte live snapshot, and 129-byte summary/config blocks (see [§5](#5-response-blocks)).

> ❓ The exact end-of-message delimiter for RX is not yet pinned down. In practice, drive the protocol **request/response lockstep** (send a command, read until the matching response is complete) rather than free-running parsing. The state machine in [§7](#7-session-download-state-machine) does this.

---

## 4. Command reference

| Opcode | Name (assigned) | Args (after opcode) | Count | Meaning |
|---|---|---|---|---|
| `0x2D` | `INIT` ✅ | ASCII `"000000"` then zeros | 1 | Session init / hello. First frame sent. Device replies `FF`. |
| `0x36` | `STATUS` 🟡 | `07 01 00 00 00 50 09 00 …` | 1 | Config/status query. Device replies with 16-byte live snapshot. |
| `0x2B` | `READ_PAGE` ✅ | `<page:u16le> 00 80` | 75 | Read a memory/directory page. Device replies with a 129-byte block. |
| `0x17` | `FETCH_RECORD` ✅ | `<recordID:3> 02` | 83 | Fetch the full record identified by a 3-byte ID. Device replies with an `80FFFEFD` record. |

### 4.1 `0x2B READ_PAGE` page sequence ✅

Observed page argument (`u16` little-endian) progression:

```
0x0011, 0x000F, 0x000E, 0x0010,           ← 4 config/metadata pages
0x00F9, 0x00F8, 0x00F7, … 0x00B4,         ← descending sweep, ~70 session-record summaries (newest first)
0x00F8                                     ← (second pass / re-read begins)
```

✅ Interpretation (confirmed by the second capture): the first 4 pages (`0x0011, 0x000F, 0x000E, 0x0010`) are **always read first** and return device config + the **driver-profile list** — byte-identical in both captures. Page **`0x00F9` is the newest/last session**; the descending `0xF9…0xB4` sweep walks **session history newest→oldest**. The app's "only download last session" setting reads exactly the config pages + `0x00F9` + one fetch — that's how this was proven. Each session page returns a 129-byte *summary* containing the record's 3-byte ID, fetched via `0x17`. The summary's leading 3 bytes are a **stable fetch key** for the session: `0x4E26B2` addressed the newest session in both captures despite being taken at different times. (This fetch key differs from the stamp inside the record — see [§4.2](#42-0x17-fetch_record-).)

### 4.2 `0x17 FETCH_RECORD` ✅

The 3-byte ID in the `0x17` command is the **fetch key**, taken from the front of the preceding `0x2B` summary block. ⚠️ **Correction (v0.2):** this fetch key is *not* the same as the value at offset 4 of the returned record — the record carries its own distinct **internal stamp** there. Verified in the clean single-session capture:

```
0x2B page 0xF9 → RX summary starts 4e26b2 → TX 17 4e26b2 02 → RX 80fffefd 9828b2 …
                      (fetch key)             (fetch key)        (internal stamp ≠ key)
```

So treat the summary's leading 3 bytes purely as an opaque fetch handle; do not expect it to equal any field inside the record. Both values are stable for a given session across captures.

---

## 5. Response blocks

### 5.1 `FF` — ready/ack ✅
Single byte `0xFF`, sent by the device immediately after `INIT`. Treat as "ready."

### 5.2 16-byte live snapshot 🟡
Returned after `0x36 STATUS`. Example (engine off, idle):

```
07 ec02 ec02 c900 ec02 ec02 9324 08 4c3d
```

- Byte 0 = `0x07` (type/length tag, 🟡).
- Subsequent `u16le` words are live sensor channels. With engine off they read constant (`0x02EC` ≈ 748 repeated). **Field mapping requires engine-on capture.** ❓
- `0x2493` (`93 24` LE) is a recurring device/session marker that also appears inside records (🟡 magic/serial).

> ⚠️ **Reality check:** this 16-byte block is sent **once**, as the response to `STATUS` during the handshake — it was **not** observed streaming, and the official phone/desktop apps expose **no live-data view**. A continuous live feed may not exist over this interface. The open experiment (see [§8](#8-known-unknowns--next-captures)) is whether **repeatedly polling `0x36`** with the engine running returns changing values. If it does, this is the live channel; if not, LapWing's Alfano integration is **session-download only** — which is fine, since that path is fully mapped and post-session comparison is the core use case.

### 5.3 129-byte summary / config blocks ✅ (framing) / 🟡 (fields)
Returned by `0x2B READ_PAGE`. Fixed length **129 bytes**. Two sub-types observed:

- **Config/driver pages** (first few): contain ASCII driver names. Example decodes to:
  `Driver ` ×N, and the active profile `Dame`. Names are space-padded fixed-width fields separated by `08` bytes.
- **Session-record summaries**: begin with the **3-byte record ID** (offsets 0–2) used by `0x17`. Remaining bytes are a per-session summary (lap counts, best lap, timestamps — exact layout ❓).

No byte offset is constant across *all* 129-byte blocks (they're polymorphic by page type), so parse them **per request context**, not by a single template.

---

## 6. Telemetry record (`80FFFEFD`)

Returned by `0x17 FETCH_RECORD`. Variable length (observed 100–514 bytes; one logical record may split across reads). **71 records** in this capture.

### 6.1 Header layout ✅ (offsets) / 🟡 (field meaning)

| Offset | Len | Field | Conf | Notes |
|---|---|---|---|---|
| 0 | 4 | Magic `80 FF FE FD` | ✅ | Constant across all records |
| 4 | 3 | Internal record stamp | ✅ | Distinct from the `0x17` fetch key (record `9828B2` for fetch key `4E26B2`); 🟡 likely a timestamp or internal sequence |
| 7 | 1 | Flags/type | 🟡 | e.g. `0x0B` |
| 8 | 3 | Timestamp / counter | 🟡 | e.g. `27 2A 0E` — varies per record |
| 11 | ~97 | Session header / summary telemetry | 🟡 | Lap counts, best lap, config (`0c 03 01 1c 27 …`), session metadata |
| **108** | var | **Track name** (ASCII) | ✅ | `*P* SHORT OKC`, space-padded |
| … | | additional fields | ❓ | |
| **289** | 16 | **Driver name** (ASCII) | ✅ | `Dame           ` fixed 16-wide, space-padded |
| ~313 | var | Sample / trackmap table | 🟡 | repeating 7-byte entries, see §6.3 |

> Offsets 108 and 289 were stable across records in this capture but may shift with record length — locate the ASCII fields by scanning rather than assuming fixed offsets in production code.

### 6.2 ASCII string fields ✅
- **Track name** carries a mode prefix: `*P*` = **Professional** mode (matches on-device display). Track DB strings seen: `*P* SHORT OKC`, `*P* LONG OKC`, `*P* ORLANDO KART`.
- **Driver name**: fixed 16-character, space-padded field (`Dame`).

### 6.3 Trailing sample table 🟡
From ~offset 313 to end, a repeating **7-byte** structure:

```
05 80 80 80 80 <idx:u16le>   05 80 80 80 80 <idx:u16le>   …
```

- `<idx>` increments monotonically across entries (≈ +0x66 per step in the sample seen).
- 🟡 Likely a per-sample or per-beacon table. The `80 80 80 80` quartet is probably a placeholder for GPS coordinates **not recorded because there was no GPS lock** — expect real lat/lon (or packed deltas) here in an engine-on/GPS capture. ❓

---

## 7. Session-download state machine ✅

```
CONNECT (SPP, ch 6)
  └─ TX  INIT            2D "000000" …                 ── XOR
        RX  FF                                          (ready)
  └─ TX  STATUS          36 07 01 00 00 00 50 09 …     ── XOR
        RX  <16-byte live snapshot>
  └─ for page in [0x0011, 0x000F, 0x000E, 0x0010]:     (config/driver pages)
        TX  READ_PAGE    2B <page:u16le> 00 80         ── XOR
        RX  <129-byte config/driver block>
  └─ for page in descending(0x00F9 … 0x00B4):          (session records, newest→oldest)
        TX  READ_PAGE    2B <page:u16le> 00 80         ── XOR
        RX  <129-byte summary>  → extract recordID[0:3]
        TX  FETCH_RECORD 17 <recordID:3> 02            ── XOR
        RX  <80FFFEFD record>  → parse (§6)
  └─ (repeat / idle)
DISCONNECT
```

🟡 The exact stop condition for the descending sweep (how the host knows it has reached the oldest record) is not yet confirmed — likely an empty/sentinel summary, or a count read from a config page. For a first implementation, stop when a summary is all-`FF`/empty or when the record ID stops advancing.

### 7.1 Download modes ✅

Two modes are confirmed, differing only in how many session pages get swept:

| Mode | Pages read | Records fetched | Use |
|---|---|---|---|
| **Last session only** | config (`0x11,0x0F,0x0E,0x10`) + `0x00F9` | 1 (newest) | Fast; maps to the app's "only download last session" toggle |
| **Full memory dump** | config + `0x00F9` descending to `~0x00B4` | ~70 (all stored) | Complete history |

Both begin with the identical handshake + config pages. For LapWing, **last-session-only is the right default** — a few hundred milliseconds, pulls the just-completed run. Offer full-dump as an explicit "import all history" action.

### 7.2 Two-phase structure of a full download ✅ (structure) / ❓ (sample fields)

A **full** download is two phases, not one — this matches the app UI showing it pull "individual laps, pit out, and other chunks":

1. **Index phase** (the `2B`/`17` interleaved sweep in §7): retrieves ~70 `80FFFEFD` **segment/header records** — per-lap, pit-in/out, and other session "chunks." These carry metadata (track, driver, stamps, summary stats) but **not** the high-rate samples.
2. **Bulk sample phase**: *after* the index sweep, the app issues a long run of `17 FETCH_REC` commands **with no interleaved `2B`**, walking **sequential memory addresses** — the 3-byte fetch arg increments by 2 each call (e.g. `78ce10 → 78d010 → 78d210 …` rolling the high byte). This streams the **raw high-rate sample memory**: the actual per-sample telemetry. ~280 reads observed for one capture.

A client that only does phase 1 gets the lap list and summary stats but **no detailed traces**. To reconstruct full sessions you must do **both** phases and stitch the sample stream against the segment headers.

**Scope warning:** a full dump spans **all stored sessions across all tracks**, not one session — the driven capture contained `SHORT OKC`, `LONG OKC`, *and* `ORLANDO KART` headers. To isolate a single session, filter header records by track string + contiguous stamp/timestamp range before stitching samples.

---

## 8. Known unknowns / next captures

To finish the spec, capture in this order and re-run the parser:

1. **CSV export of a known session + that session's raw dump** — *now the single most valuable input.* A real 6-lap driven session has been captured (full dump, both phases), and the sample stream clearly contains real structured telemetry — but the **per-sample field encoding is NOT yet decoded.** Inferring fields (GPS, lap times, RPM) from the binary alone does **not** converge: numeric-range guessing produces false positives (e.g. values landing in a plausible GPS-latitude range were confirmed bogus once their spatial span was sanity-checked). The reliable method is **known-plaintext alignment** — take the CSV the official app exports for a specific session, line its decoded values (lap times, GPS lat/lon, RPM, channels) against the raw bytes of that same session's records, and read the offsets/scales off directly. Until then, treat all sample-level field offsets as ❓.
2. **Engine running + poll `0x36` repeatedly** — the official apps have **no live view**, so a live stream may not exist. Test it directly: with the engine running, send `STATUS` (`0x36`) on a loop and watch whether the 16-byte response changes. Changing values ⇒ a live channel LapWing can use for a real-time overlay. Static values (or no response to repeated polls) ⇒ accept that Alfano-over-BT is **session-download only**.
3. **Change a setting in the app (driver name, track, sample rate)** — reveals **write** opcodes (none seen in this read-only download capture).
4. **A second, longer session** — confirm the `0x2B` page-numbering scheme and the sweep stop condition.

Open questions to resolve:
- Full 129-byte summary layout (lap count / best lap offsets).
- Header fields at record offsets 7–107.
- RX message framing/delimiter (currently handled by request/response lockstep).
- Live-stream subscribe command (if any).
- Write/config-set commands.

---

## 9. Implementation notes

### 9.1 Platform reality (read this first)
- **Web Bluetooth (browser / PWA): NOT POSSIBLE.** The Web Bluetooth API is BLE/GATT only and cannot open a Classic RFCOMM/SPP socket. A pure web build of LapWing will never see this device.
- **Tauri (LapWing's actual stack): YES, via the native layer.**
  - Tauri **webview/JS frontend** → same limitation as a browser, can't do RFCOMM.
  - Tauri **native shell** → can. On **Android** (primary target), implement a Tauri plugin that uses `android.bluetooth.BluetoothSocket.createRfcommSocketToServiceRecord(SPP_UUID)` in Kotlin/JNI and bridges the byte stream to JS over the plugin's IPC channel.
  - **Desktop** later: Linux `bluer` (BlueZ RFCOMM), Windows `windows-rs` Bluetooth APIs, macOS `IOBluetooth` (most painful).

### 9.2 Reference flow (transport-agnostic pseudocode)
```rust
// 1. Connect SPP (channel resolved via SDP from SPP_UUID)
let mut sock = rfcomm_connect(device_addr, SPP_UUID)?;

// 2. Handshake
sock.write_all(&build_command(0x2D, b"000000"))?;
expect_byte(&mut sock, 0xFF)?;                       // ready
sock.write_all(&build_command(0x36, &hex!("07 01 00 00 00 50 09 00")))?;
let _snapshot = read_message(&mut sock)?;            // 16-byte live snapshot

// 3. Config/driver pages
for page in [0x0011u16, 0x000F, 0x000E, 0x0010] {
    sock.write_all(&build_read_page(page))?;
    let block = read_message(&mut sock)?;            // 129 bytes
    parse_config_or_drivers(&block);
}

// 4. Session records (newest → oldest)
let mut page = 0x00F9u16;
loop {
    sock.write_all(&build_read_page(page))?;
    let summary = read_message(&mut sock)?;          // 129 bytes
    if is_empty_summary(&summary) { break; }
    let rec_id = &summary[0..3];
    sock.write_all(&build_command(0x17, &[rec_id[0], rec_id[1], rec_id[2], 0x02]))?;
    let record = read_message(&mut sock)?;           // 80FFFEFD …
    parse_record(&record);                           // §6
    page -= 1;
}

fn build_read_page(p: u16) -> [u8;258] {
    build_command(0x2B, &[p as u8, (p>>8) as u8, 0x00, 0x80])
}
// build_command = opcode + args, zero-pad to 257, append XOR of those 257 bytes.
```

### 9.3 Suggested repo layout for a sample
```
/alfano-bt/
  ALFANO6_BT_PROTOCOL.md        ← this file
  src/
    protocol.rs                 ← build_command(), parsers, record structs (pure, no I/O — unit-testable)
    transport_desktop.rs        ← bluer/serial RFCOMM for laptop testing
    transport_android.rs        ← Tauri plugin shim (Kotlin/JNI BluetoothSocket)
  tests/
    fixtures/                   ← captured frames (hex) for offline parser tests
    parse_tests.rs              ← feed §A example frames, assert decoded fields
```
Keep `protocol.rs` **I/O-free** so it can be unit-tested against captured hex fixtures (the frames in [§A](#appendix-a-example-frames)) with no Bluetooth hardware in the loop.

---

## Appendix A: Example frames

Real bytes from the capture (truncated where noted). `cr` = RFCOMM credit octet (handled by OS stack; not part of the app payload).

**TX INIT**
```
2d 30 30 30 30 30 30 00 …(zeros)… 2d        (258 B, ASCII "-000000…", XOR=0x2D)
```
**TX READ_PAGE 0x00F9**
```
2b f9 00 00 80 00 …(zeros)… 52              (258 B, XOR=0x52)
```
**TX FETCH_RECORD 0x4E26B2**
```
17 4e 26 b2 02 00 …(zeros)… cf              (258 B, XOR=0xCF)
```
**RX live snapshot (16 B, idle)**
```
07 ec02 ec02 c900 ec02 ec02 9324 08 4c3d
```
**RX driver-list summary (129 B)** — decodes to `Driver …`, `Dame`
```
44 72 69 76 65 72 20 04 …  50 01 44 61 6d 65 20 20 … (… "Driver "… "P.Dame "…)
```
**RX telemetry record (head, 80FFFEFD)**
```
80 ff fe fd 9828b2 0b 272a0e 06 1a00 df00 4008 0000 1f80 1f29 …
   … 2a 50 2a 20 53 48 4f 52 54 20 4f 4b 43 20 …   (offset 108: "*P* SHORT OKC ")
   … 44 61 6d 65 20 20 20 20 20 20 20 20 20 20 20  (offset 289: "Dame           ")
   … 05 80 80 80 80 <idx16> 05 80 80 80 80 <idx16> … (trailing sample table)
```

---

## Appendix B: Decoded semantics summary

| Item | Value(s) observed |
|---|---|
| Active driver | `Dame` |
| Driver profile slots | `Dame`, `Driver 2`–`Driver 6` |
| Track database | `*P* SHORT OKC`, `*P* LONG OKC`, `*P* ORLANDO KART` |
| Mode prefix | `*P*` = Professional |
| Device/session marker | `0x2493` (`93 24` LE) |
| Stored session records | ~71 (this capture) |

---

*End of spec v0.1. Generated from `btsnoop_hci.log` (handle 0x06, RFCOMM ch 6). Update confidence markers as engine-on/GPS captures land.*
