# SynScanLink SDK: Mount Protocol Findings

*As of 2026-10-03*

Sky-Watcher's own SynScanLink library drives motor controllers newer than MC 3.38 with the `:X` extended protocol, and on those boards it never sends the classic `:K` stop. These notes record what the library does, in our own words, so third-party drivers can talk to the same hardware the same way.

## Contents

- [Scope and provenance](#scope-and-provenance)
- [Protocol selection by firmware](#protocol-selection-by-firmware)
- [Transport and timing](#transport-and-timing)
- [Connect sequence](#connect-sequence)
- [The `:X` protocol as the library uses it](#the-x-protocol-as-the-library-uses-it)
- [Axis motion](#axis-motion)
- [GOTO](#goto)
- [Pulse guiding and guide rates](#pulse-guiding-and-guide-rates)
- [Extended settings and feature flags](#extended-settings-and-feature-flags)
- [Tracking, trail points and rate precision](#tracking-trail-points-and-rate-precision)
- [Other capabilities](#other-capabilities)
- [Model differences and defaults](#model-differences-and-defaults)
- [Notes for driver authors](#notes-for-driver-authors)
- [Open questions](#open-questions)

## Scope and provenance

- **Source:** SynScanLink SDK v1.0.0 for iOS and macOS, binaries dated December 2024, as distributed by Sky-Watcher. The library is built from the SynScan app's code base.
- **Method:** the exported and debug symbol names, type layouts, constants, and the order and arguments of calls inside the library's functions were examined. No code, headers or disassembly are reproduced here.
- **Purpose:** interoperability. Every item below is a fact about how software talks to the mount, not how Sky-Watcher wrote its code.
- **Confidence:** nothing here was checked on the wire. Anything marked *inferred* comes from a name rather than from the call sequence itself.
- **Units:** the library works in radians and radians per second internally. Sidereal is 7.2921158 × 10⁻⁵ rad/s.

## Protocol selection by firmware

The library switches every axis operation to the `:X` protocol when the motor-controller firmware is **newer than 3.38** (3.39 and up). It reads the version from axis 1 at connect, caches the choice for the connection, and never compares axis 2's version.

| Operation | MC 3.38 and older (legacy) | MC 3.39 and newer (`:X`) |
| --- | --- | --- |
| Move at a rate | `:G` + `:I` + `:J`, stopping first when the mode or direction changes | `:X 02` with a signed rate |
| Stop | `:K` | `:X 02` with rate 0 |
| GOTO | `:G` + `:H` + `:M` + `:J` | `:X 04` (position plus rate on arrival) |
| Read one axis position | `:j` (24-bit) | `:X 00` index 03 (32-bit) |
| Read both axis positions | `:j` twice | `:X 0F` (one command) |
| Set position | `:E` (24-bit) | `:X 01` (32-bit) |
| Counts per revolution | `:a` | `:X 00` index 02 |
| Read home indexer | `:q` 000000 | `:X 00` index 0B |

On the `:X` path, positions are sent and read as 32-bit values. The legacy path uses 24-bit values offset by 0x800000.

A board's firmware version decides which half of the rest of these notes applies to it.

### Which mounts can run newer firmware

Based on Sky-Watcher's [motor-controller firmware downloads](https://skywatcher.com/download/software/motor-control-firmware/) as of September 2026, plus [Sky-Watcher USA](https://www.skywatcherusa.com/pages/firmware-and-software) for the Star Adventurer 2i and Mini. Each row is one hardware revision. The **Revision** column is taken from the download notes, which split some mounts by USB port, Wi-Fi, board generation or build date. A dash means the notes don't split that mount.

**New firmware available (newer than 3.38, uses `:X`)**

| Mount | Revision | Board | Latest |
| --- | --- | --- | --- |
| EQ6 | Built-in USB Type B port | MC015 | 3.47 |
| EQ6-R | Built-in USB Type B port | MC015 | 3.47 |
| AZ-EQ6 | Built-in USB Type B port | MC015 | 3.47 |
| EQ8 | Built-in USB Type B port | MC015 | 3.47 |
| EQ8-R / EQ8-RH | — | MC015 | 3.47 |
| CQ350 | — | MC015 | 3.47 |
| Star Gate Dobsonians | "Newer Star Gate series" | MC015 | 3.47 |
| EQ3, EQ5 | 3.xx board with on-board USB | MC019 | 3.47 |
| EQM-35 | 3.xx board with on-board USB | MC019 | 3.47 |
| HEQ5 | 3.xx board | MC020 | 3.47 |
| AZ-GTi, AZ-GTe, AZ-GTiX, Virtuoso GTi | Built before May 2025 | MC014 | 3.40 |
| AZ-GTi, AZ-GTe, AZ-GTiX, Virtuoso GTi | Built after May 2025 | MC029 | 3.54 |
| Skyliner Dobsonian GoTo | With Wi-Fi | MC014 | 3.40 |
| Star Discovery | With Wi-Fi, built before May 2025 | MC014 | 3.40 |
| Star Discovery | With Wi-Fi, built after May 2025 | MC029 | 3.54 |
| Starliner Dobsonian GoTo, AZ-GO2 | Built after May 2025 | MC029 | 3.54 |
| Fusion-120i | — | MC029 | 3.54 |
| StarGate 16/18/20 | — | MC030 | 3.69 |
| Wave 100i, Wave 150i | — | MC030 | 3.69 |
| EQ-AL55i Pro | — | MC016 | 3.48 |
| Star Adventurer GTi | — | MC021 | 3.48 |

**Old firmware only (3.38 or older, uses the legacy commands)**

| Mount | Revision | Board | Latest |
| --- | --- | --- | --- |
| EQ6 | Older board | — | 2.04 |
| EQ6-R | No built-in USB port | — | 2.15 |
| AZ-EQ6 | No built-in USB port | — | 2.15 |
| EQ8 | No built-in USB port | — | 2.15 |
| EQ3, EQ5 | Older board | — | 2.04 |
| EQ3, EQ5 Pro GoTo | Older board | — | 2.07 |
| EQM-35 | Older board | — | 2.07 |
| HEQ5 | 2.xx board | — | 2.04 |
| Skyliner Dobsonian GoTo | Without Wi-Fi | — | 2.09 |
| Star Discovery | Without Wi-Fi | MC006 | 2.18 |
| AZ-EQ5 | — | — | 3.01 |
| AllView | — | — | 2.14 |
| SynScan AZ GoTo | — | — | 2.09 |
| Star Adventurer 2i | — | MC017 | 3.11 |
| Star Adventurer Mini | — | — | 3.11 |

- What matters is the firmware installed, not the model. MC014 units still on 3.26 or older (AZ-GTi family, Skyliner Wi-Fi, Star Discovery Wi-Fi) get the legacy commands until they're updated to 3.40. `:e` reports the installed version.
- Star Gate Dobsonians appear under both MC015 ("newer Star Gate series") and MC030 ("StarGate 16/18/20"). The notes don't say which models use which board.

## Transport and timing

The library allows one command in flight at a time and gives each one up to three sends of 200 ms.

- **Framing.** `:` + command letter + axis character (`1` or `2`) + upper-case hex data + CR. Classic values are little-endian, as in the published command set. `:X` data after the sub-command is big-endian, two's complement.
- **Replies.** Bytes before the first `=` or `!` are discarded. A reply must end in CR. `=` is success; `!` is an error, with the error code kept. Nothing ties a reply to its request beyond the one-in-flight rule.
- **Serialisation.** A lock covers each full request and reply, so commands from different tasks never interleave on the link.
- **UDP.** Port 11880 at the configured address (default 192.168.4.1). For every command the library opens a fresh socket (so each command uses a new source port), discards any datagrams already waiting, sends, and waits for the reply.
- **Timeout and resend.** Default read timeout 200 ms; on a timeout the identical frame is sent again, up to 2 resends (3 sends in all). Bluetooth uses a 1000 ms timeout.
- **Trail points are never resent.** Resends are turned off while trail points are being queued.

## Connect sequence

Commands below omit the leading `:` and trailing CR; `N` is the axis character.

1. `e1`: firmware version and mount code. Any failure ends the connect.
2. `q1010000`: Status EX (see [Extended settings](#extended-settings-and-feature-flags)). A `!` reply means older firmware: all extended capabilities are taken as absent and the connect carries on. A timeout fails the connect.
3. Mount mode: from the caller when the mount reports switchable alt-az / equatorial mode; otherwise alt-az when the mount code is 0x80 or higher, equatorial below that.
4. For axis 1, then axis 2: `eN` (only if not already read, so in practice `e2`), then counts per revolution (`XN0002` on `:X` boards, `aN` on legacy boards), then status `fN`.
5. `b1`: step-timer frequency. Classic on both paths.
6. `s1`: PEC period. A `!` reply is accepted.
7. Only when Status EX reports dual encoders: `WN050000` (dual encoder off) for each axis.
8. `PNd` for each axis: the stored guide rate, 0.5× (`2`) by default.
9. Only when Status EX reports a polar-scope LED: `V1` and `V2` with the stored brightness.
10. `:X` boards only: `X10006` (mount-clock ticks per second), `X10503` (clear the trail-point queue), then an empty trail-point query whose reply gives the queue's free space.
11. Starting position:
    - For each axis whose `f` reply said "not initialised": set the position (`XN01` on `:X` boards, `EN` on legacy), then `FN`. The position is home, or the stored park position if there is one. This is the only place the library sends `:F`.
    - If no axis needed that: read both positions (`XN0003` or `jN`).

The connect never changes the serial baud rate.

## The `:X` protocol as the library uses it

The library uses ten `:X` sub-commands. It never sends 03, 06, 07, 08, 09 or 0C, and its code for 0D is switched off.

| Sub | Library use | Arguments |
| --- | --- | --- |
| 00 | Read a register by index: 02 counts per rev, 03 position, 06 clock ticks per second, 0B home-indexer reading | Register index |
| 01 | Set axis position | 32-bit position |
| 02 | Move at a rate, and stop (rate 0) | Signed rate, 64-bit |
| 04 | GOTO a position, then continue at a given rate | 32-bit position + signed rate |
| 05 | Operations: 00 / 01 SNAP shutter output, 03 clear the trail-point queue | Operation code |
| 0A | Add an offset to trail-point tracking | Axis 1 offset + axis 2 offset, 32-bit counts each |
| 0B | Set the mount clock | Time, 64-bit ticks |
| 0D | Queue a trail point with rates (present but disabled in this build) | Time + both positions + both rates (32-bit) |
| 0E | Queue an absolute trail point | Time (64-bit) + axis 1 and axis 2 positions (32-bit) |
| 0F | Snapshot of both axis positions plus the mount clock | Sent on axis 1 with a zero argument |

**Frames.** `:X` + axis character (`1` or `2`) + two hex digits of sub-command + data + CR. Commands that cover both axes (05 03, 0A, 0B, 0E, 0F) are sent on axis `1`. There is no axis `3`.

**Units.**

- Position: radians × counts-per-rev ÷ 2π, rounded to the nearest count, as a 32-bit value. On an equatorial mount, axis 1 positions get a quarter-turn offset added before encoding.
- Rate: the same conversion × 1024, rounded, as 64-bit (`02`, `04`) or 32-bit (`0D`). The library applies no range clamp on this path.
- Time: mount-clock ticks, truncated, as 64-bit. The library reads the tick rate from register 06. The mount clock holds Unix-epoch time.

**Clock.** The library sets the mount clock to host time with `:X 0B`, sending it twice, and treats a `!` reply as success. It avoids sending a time whose low 16 bits fall at or above 0xD901, rounding around that band. Its round-trip correction ends up as zero in this build, so the clock is set to host time without allowing for link delay.

**Replies.** `0F` returns both positions and the mount clock (32 hex digits). `0E` returns both positions, the mount clock and the number of free queue slots.

The library never sends `:X 05 02` (busy) or `:X 05 04` (stop one axis). Busy is read with classic `:f`, and stops use `:X 02` (see [Axis motion](#axis-motion)).

## Axis motion

On an `:X` board, every speed change is a single `:X 02` command: no stop, no wait for rest, no mode or direction setup, even when the direction reverses. The legacy path is the familiar stop-and-restart.

**Move at a rate**

- Every rate the library sends, including tracking, guide offsets and stops, is first multiplied by a per-mount fine-adjustment factor (default 1.0, limited to 0.5–1.5).
- `:X` path: encode the signed rate and send `:X 02`. Nothing else is sent before or after, and axis status isn't checked first.
- Legacy path:
    - Rates are limited to ±3.4°/s. At or below 0.001× sidereal the axis is stopped with `:K` instead.
    - Above 128× sidereal the high-speed mode is used and the period is divided by the `:g` ratio.
    - A change that keeps the same direction at low speed while already moving sends only `:I`.
    - Any other change stops with `:K`, polls `:f` every 100 ms until stopped, then sends `:G`, `:I` and `:J`.
    - The step period is never below 6.

**Stop**

- `:X` path: `:X 02` with a rate of exactly 0.
- Legacy path: `:K`. The library has an instant-stop (`:L`) option but never turns it on.
- The stop task sends the stop and returns once the command is accepted, retrying every 200 ms on errors. It doesn't wait for the axis to come to rest or check the final position.

**Initialisation, status and waiting**

These use classic commands on both paths. The library has no `:X` equivalent for them.

- Initialise an axis: set its position, then classic `:F`, only at connect and only for axes that report "not initialised".
- Moving or stopped: classic `:f`, read on `:X` boards too. The library uses the running bit, the mode bit (GOTO vs rate), direction, high speed, and initialised. It ignores the blocked bit.
- Waiting for an axis to stop: poll `:f` every 100 ms until the running bit clears. There is no timeout; only cancelling ends the wait.

**Position reads.** While connected, a background task re-reads both positions whenever the cached ones are 500 ms old. On `:X` boards that is one `:X 0F`.

The library keeps the last commanded rate per axis. Pulse guiding and the return to idle start from that value.

## GOTO

The library's GOTO is a sequence of plain moves, each one landing at rate 0 and checked at the end. Sky-Watcher's own software corrects landing errors by going round again, not inside a single move.

**Single-axis move**

- `:X` path: `:X 04` with the target position and a rate on arrival of **0**, with no stop first. The library has a variant with a non-zero arrival rate, but nothing in it uses that variant.
- Legacy path:
    1. If the axis is moving, `:K`, then poll `:f` until stopped.
    2. Read the position with `:j`. If the move rounds to 0 counts, do nothing more.
    3. `:G` with the high-speed mode for moves over about 2.67°, otherwise low speed; direction from the sign of the move.
    4. `:H` with the distance, `:M` with a brake distance of 3,500 counts, then `:J`.
- The library never sends `:T`, so the board's own GOTO cruise speed is always used.

**Sky-target GOTO** (both paths)

1. For a moving target, aim at where it will be 2 s ahead.
2. Move both axes (axis 2 first, then axis 1), then poll `:f` every 100 ms until both stop.
3. **Final approach always in the positive direction.** If an axis's move is negative, the first pass overshoots the target by 1,200″ on an equatorial mount (2,700″ in alt-az) and the next pass comes back. This happens at most once per axis per GOTO.
4. A pass counts as arrived when the remaining separation is within **120″**.
5. Timing: a pass that finishes in under 3.95 s is padded out to 3.95 s and accepted. A slower pass leads to another approach.
6. Give-up: after five extra approaches the GOTO is reported as done anyway. Other errors retry every 200 ms with no limit, until cancelled.

The 120″ tolerance becomes 5′ in alt-az mode, and 0.1° for mount codes 0x80–0x82 with firmware 1.09 or older.

## Pulse guiding and guide rates

The manufacturer's pulse guide is a rate change, a host-timed wait, and a return to idle. On an equatorial mount's Dec axis, the return to idle is a stop, which on an `:X` board is a single `:X 02` with rate 0.

1. Read the axis's last commanded rate (0 if none).
2. Add the guide rate, signed by the requested axis direction, and send it as one move command (`:X 02` on newer boards). If the send fails, it is resent at once until it succeeds or the pulse is cancelled.
3. Sleep on the host clock for the pulse duration, less the time since step 2 began. There is no maximum duration.
4. Return the axis to idle:
    - RA with tracking on: restart tracking with one `:X 02` at the tracking rate.
    - Dec on an equatorial mount, or any axis with tracking off: the stop task (`:X 02` rate 0, or `:K` on legacy boards). It does not wait for rest or poll status.
    - Alt-az mounts: any pulse stops alt-az tracking on both axes; it is restarted after a stop and wait.

There is no settle wait, position check or correction after the stop.

**Direction.** The library takes a raw axis direction (positive or negative), not N/S/E/W. It doesn't flip for hemisphere, pier side or alt-az mode, so the caller maps compass directions to axis signs. The pulse entry point isn't exported in the public C API.

**Overlapping pulses.** A new pulse on the same axis cancels the running pulse's wait. Pulses aren't merged.

**Guide rates.** The library offers five rates and uses the chosen one for its own pulses. At connect it also sends the same choice to the controller with `:P` (default 0.5×), which sets the rate the ST-4 port uses.

| Rate (× sidereal) | `:P` code |
| --- | --- |
| 1.0 | 0 |
| 0.75 | 1 |
| 0.5 | 2 |
| 0.25 | 3 |
| 0.125 | 4 |

## Extended settings and feature flags

The library reads the tracking-torque support flag but has no way to send the torque setting (IDs 000006 / 000106).

**Status EX** (`:q` ID 000001) is read at connect. The reply is six hex digits; the library uses the first three.

| Digit | Bit 0 | Bit 1 | Bit 2 | Bit 3 |
| --- | --- | --- | --- | --- |
| 1st | PPEC training running | PPEC playback running | — | — |
| 2nd | Dual encoders | PPEC | Auto home | Switchable alt-az / equatorial |
| 3rd | Polar-scope LED | Axes must start separately | Tracking-torque selection | — |

**Extended settings the library sends** (`:W`), the complete list:

| ID | Setting | In the published command set |
| --- | --- | --- |
| 000000 / 000001 | PPEC training start / cancel | Yes |
| 000002 / 000003 | PPEC playback on / off | Yes |
| 000004 / 000005 | Dual encoder on / off (off is sent at connect) | Yes |
| 000008 | Reset the home indexer | Yes |
| xxxx0D | Set the serial baud rate; the upper two bytes are the baud rate ÷ 100 (115200 → 04800D) | No |

**Extended inquiries** (`:q`): 000001 for Status EX, and 000000 for the home indexer on legacy boards.

> [!WARNING]
> The baud-rate setting isn't in the published command set. A wrong value leaves the board on a baud rate the host isn't using. The library contains it but nothing in the library calls it.

## Tracking, trail points and rate precision

Equatorial tracking is one constant-rate move command on RA and nothing on Dec. Alt-az tracking on `:X` boards streams time-stamped trail points, which the changelog credits with less oscillation on newer mounts.

**Equatorial tracking.** The library looks up the rate, flips its sign in the southern hemisphere, records it as the axis's last rate, and sends a single move command on RA (`:X 02` on newer boards), retrying every 100 ms until accepted. Dec isn't commanded, and the rate isn't refreshed while tracking.

| Tracking rate | rad/s | Relative to sidereal |
| --- | --- | --- |
| Sidereal | 7.2921158 × 10⁻⁵ | 1 |
| Solar | 7.2722052 × 10⁻⁵ | 0.99727 |
| Lunar | 7.0234578 × 10⁻⁵ | 0.96316 |

**Alt-az tracking** runs a 5 s cycle.

- On `:X` boards with dual encoders off, each cycle queues `:X 0E` points covering the next 5.5 s. Spacing is 5.5 s divided by the free slots, but never less than 100 ms or the measured link latency. If the mount is more than 3° off target, it re-targets with a GOTO instead.
- Otherwise each cycle sends rate updates with `:X 02` (or the legacy commands).

**Trail points.**

- Each point is a mount-clock time plus both axis positions, queued with `:X 0E`.
- Queue depth isn't fixed: it's the free count the board reports after a clear.
- The sender picks points due later than now plus twice the average round trip (averaged over the last four), sends while slots are free, and polls at 1.5 × the point interval when the queue is full.
- At the end it waits for the queue to drain, then clears it with `:X 05 03`.
- `:X 0A` adds a position offset in counts to both axes during trail-point tracking.
- The SDK manual recommends at least 30 ms between points and speeds around 10× sidereal, and it supports satellite tracking.

**Rate precision.**

- `:X` boards: one rate step is 1/1024 count per second, which the library expresses as a fraction of sidereal: 86164.09 ÷ (counts per rev × 1024).
- Legacy boards: the relative change from one `:I` period step at sidereal.
- Legacy period from rate: round(2π ÷ counts-per-rev × timer frequency ÷ rate), rounding halves away from zero.

## Other capabilities

The rest of the library's hardware access uses documented commands. On `:X` boards the library still sends these classic commands: `:e`, `:F`, `:f`, `:b`, `:P`, `:V`, `:s`, and `:W` / `:q` for extended settings.

| Capability | Legacy boards | `:X` boards | Notes |
| --- | --- | --- | --- |
| Initialise axis | `:E` then `:F` | `:X 01` then `:F` | At connect, only for axes reporting "not initialised" |
| Axis status (moving / stopped) | `:f` | same | Polled every 100 ms while waiting for a stop |
| PEC period | `:s` | same | Read at connect |
| SNAP shutter output | `:O` with 0 / 1 | `:X 05` 00 / 01 | Shutter open or closed |
| Polar-scope LED | `:V` + one brightness byte | same | Brightness given as 0–1, sent as a byte |
| Home indexer | `:q` 000000 to read, `:W` 000008 to reset | `:X 00` index 0B to read, `:W` 000008 to reset | Used by the auto-home task |
| Counts per revolution | `:a` | `:X 00` index 02 | |
| Step timer frequency | `:b` | same | |
| High-speed ratio | `:g` | same | Only used on the legacy fast path |
| Firmware version | `:e` | same | Decides legacy vs `:X` |

**Auto home.** A dedicated task slews toward the index sensor, watches for the reading to latch, backs off to a reliable side, and slews to the latched position. It stops and waits for full rest between phases (*inferred* from function names).

**PPEC and dual encoder** are switched with the `:W` IDs in the previous section, and Status EX reports whether each axis supports them.

## Model differences and defaults

The library has almost no per-model knowledge and no table of mount names. Outside the firmware version, the mount code changes behaviour in only three places.

- Codes 0x80 and above start in alt-az mode (unless the mount reports switchable mode).
- On the legacy path, counts per revolution are fixed at 1,453,975 for 0x80 and 2,118,424 for 0x82 instead of using the `:a` reply.
- Codes 0x80–0x82 with firmware 1.09 or older get a 0.1° GOTO tolerance.

**Altitude limits** are fixed constants shared by every mount:

| Limit | Default | Allowed range |
| --- | --- | --- |
| Minimum altitude | 0° | −10° to 30° |
| Maximum altitude | 75° | 60° to 90° |

### Known mount codes

The mount code is the last byte of the `:e` reply. Neither the SDK nor the published command set lists these codes. The table below combines the driver tables that do: Sky-Watcher's 2013 open-source API, INDI (core `skywatcherAPI` and `indi-eqmod`), and OpenAstro's AlpacaBridge.

| Code | Mount | Listed in |
| --- | --- | --- |
| 0x00 | EQ6 | Sky-Watcher 2013 API, INDI, AlpacaBridge |
| 0x01 | HEQ5 | Sky-Watcher 2013 API, INDI, AlpacaBridge |
| 0x02 | EQ5 | Sky-Watcher 2013 API, INDI, AlpacaBridge |
| 0x03 | EQ3 | Sky-Watcher 2013 API, INDI, AlpacaBridge |
| 0x04 | EQ8 | INDI, AlpacaBridge |
| 0x05 | AZ-EQ6 | INDI, AlpacaBridge |
| 0x06 | AZ-EQ5 | INDI, AlpacaBridge |
| 0x09 | EQ-AL55i Pro | AlpacaBridge only (one owner's report) |
| 0x0A | Star Adventurer | INDI, AlpacaBridge |
| 0x0C | Star Adventurer GTi | indi-eqmod, AlpacaBridge |
| 0x20 | EQ8-R Pro | INDI, AlpacaBridge |
| 0x22 | AZ-EQ6 Pro | INDI, AlpacaBridge |
| 0x23 | EQ6-R Pro | indi-eqmod, AlpacaBridge (INDI core names it "EQ6 Pro") |
| 0x24 | EQ6 Pro | indi-eqmod, AlpacaBridge |
| 0x25 | CQ350 Pro | indi-eqmod, AlpacaBridge |
| 0x31 | EQ5 Pro | INDI, AlpacaBridge |
| 0x32 | EQM-35 Pro | AlpacaBridge only (one owner's report) |
| 0x44 | Wave 100i | INDI core, AlpacaBridge |
| 0x45 | Wave 150i | INDI core, AlpacaBridge |
| 0x80 | GT (alt-az GoTo) | Sky-Watcher 2013 API, INDI |
| 0x81 | MF | Sky-Watcher 2013 API, INDI |
| 0x82 | 114GT | Sky-Watcher 2013 API, INDI |
| 0x90 | Dobsonian (DOB) | Sky-Watcher 2013 API, INDI |
| 0xA2 | AZ-GTe | INDI core, AlpacaBridge |
| 0xA5 | AZ-GTi | INDI, AlpacaBridge |
| 0xF0 | "GEEHALEL" | indi-eqmod only |

- Sky-Watcher's own list, in a code comment in the 2013 API, covers only 0x00–0x03, 0x80–0x82 and 0x90. The rest were matched to mounts by driver authors from user reports.
- The sources disagree on 0x23: INDI core names it "EQ6 Pro", while indi-eqmod and AlpacaBridge name it the EQ6-R Pro and give 0x24 to the EQ6 Pro.
- 0x09 and 0x32 each rest on a single unit, so it isn't known whether they belong to one model or a family.
- The list is likely incomplete. Newer mounts may report codes that no table has yet.

## Notes for driver authors

On MC 3.39 and newer, following the manufacturer's sequence means sending `:X` for every rate change, stop and position read, with one command in flight at a time.

- Keep one command on the link at a time, with a 200 ms reply timeout and up to two resends. On UDP, discard stale datagrams before each send.
- Change the rate, or reverse direction, with one `:X 02`. No stop-and-restart is needed first.
- End a guide pulse by returning to the idle rate with `:X 02`: rate 0 on an equatorial Dec axis, the tracking rate on RA. The manufacturer does nothing after that: no wait, no position check.
- GOTO with `:X 04` and an arrival rate of 0, poll `:f` every 100 ms until stopped, and check the result against the target. Repeat if it's more than 120″ off, finishing each axis in the positive direction.
- Read both positions with one `:X 0F`, or a single axis with `:X 00` index 03.
- Initialise an axis by setting its position and then sending classic `:F`, only when `:f` says it isn't initialised. Read moving / stopped from classic `:f`. The library mixes these classic commands with `:X` on the same link.
- The manufacturer's software never writes `:T`, so GOTOs run at the board's factory cruise speed.
- The manufacturer's software never sends the torque setting (000006 / 000106), so there is no reference for what it does on any board.

## Open questions

None of this was checked on the wire.

- Whether classic `:f` reliably reports "stopped" after `:X 02` and `:X 04` motion on every board. The library relies on it.
- Whether setting a position and sending classic `:F` fully prepares an axis for `:X` motion on every board.
- What happens when a second pulse arrives on an axis while one is running: the call order suggests the first pulse's return to idle can override the new one.
- Which boards accept trail-point tracking (the SDK manual names the AZ-GTi as one), and how the board answers an empty `:X 0E` query.
- Whether the mount-clock rounding band (low 16 bits at or above 0xD901) reflects a real board limit.
- The baud-rate setting (xxxx0D): which rates the board accepts, and whether it survives a power cycle.
