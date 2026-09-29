# SynScanLink SDK: Mount Protocol Findings

*As of 2026-09-29*

Sky-Watcher's own SynScanLink library drives motor controllers newer than MC 3.38 with the `:X` extended protocol, and on those boards it never sends the classic `:K` stop. These notes record what the library does, in our own words, so third-party drivers can talk to the same hardware the same way.

## Contents

- [Scope and provenance](#scope-and-provenance)
- [Protocol selection by firmware](#protocol-selection-by-firmware)
- [The `:X` protocol as the library uses it](#the-x-protocol-as-the-library-uses-it)
- [Axis motion](#axis-motion)
- [Pulse guiding and guide rates](#pulse-guiding-and-guide-rates)
- [Extended settings and feature flags](#extended-settings-and-feature-flags)
- [Tracking, trail points and rate precision](#tracking-trail-points-and-rate-precision)
- [Other capabilities](#other-capabilities)
- [Model differences and defaults](#model-differences-and-defaults)
- [Notes for driver authors](#notes-for-driver-authors)
- [Open questions](#open-questions)

## Scope and provenance

- **Source:** SynScanLink SDK v1.0.0 for iOS and macOS, binaries dated December 2024, as distributed by Sky-Watcher. The library is built from the SynScan app's code base.
- **Method:** only the exported and debug symbol names, type layouts, constants, and the order of calls inside a few functions were examined. No code, headers or disassembly are reproduced here.
- **Purpose:** interoperability. Every item below is a fact about how software talks to the mount, not how Sky-Watcher wrote its code.
- **Confidence:** anything marked *inferred* comes from a name or a call order, not from watching the wire.

## Protocol selection by firmware

The library switches every axis operation to the `:X` protocol when the motor-controller firmware is **newer than 3.38** (3.39 and up). It reads the version once at connect and caches the choice per connection.

| Operation | MC 3.38 and older (legacy) | MC 3.39 and newer (`:X`) |
| --- | --- | --- |
| Move at a rate | `:G` + `:I` + `:J`, stopping first when the mode or direction changes | `:X 02` with a signed rate |
| Stop | `:K` (or `:L`) | `:X 02` with rate 0 |
| GOTO | `:G` + `:H` + `:M` + `:J` | `:X 04` (position plus rate on arrival) |
| Read position | `:j` (24-bit) | `:X 00` index 03 (32-bit) |
| Set position | `:E` (24-bit) | `:X 01` (32-bit) |
| Read home indexer | `:q` 000000 | `:X 00` index 0B |

On the `:X` path, positions are sent and read as 32-bit values. The legacy path uses 24-bit values.

A board's firmware version decides which half of the rest of these notes applies to it.

## The `:X` protocol as the library uses it

The library uses nine `:X` sub-commands. It never sends 06, 07, 08, 09, 0C or 0D.

| Sub | Library use | Arguments |
| --- | --- | --- |
| 00 | Read a register by index: 02 counts per rev, 03 position, 06 clock ticks per second, 0B home-indexer reading | Register index |
| 01 | Set axis position | 32-bit position |
| 02 | Move at a rate, and stop (rate 0) | Signed rate, 64-bit |
| 04 | GOTO a position, then continue at a given rate | 32-bit position + signed rate |
| 05 | Operations: SNAP shutter output, and 03 to clear the trail-point queue | Operation code |
| 0A | Add an offset to trail-point tracking | RA offset + Dec offset |
| 0B | Set the mount clock | Time |
| 0E | Queue an absolute trail point | Time + RA and Dec positions |
| 0F | Snapshot of both axis positions plus the mount clock | Not traced |

**Units.** Rates are counts per second multiplied by 1024, so the smallest step is 1/1024 count per second. Times are in mount-clock ticks, and the library reads the tick rate from register 06.

**Clock.** The library can set the mount clock to host time and measures the round trip to correct for link delay. It also rounds requested times to values the board accepts (*inferred* from the function name).

The library uses operation 05 03 to clear the trail-point queue.

## Axis motion

On an `:X` board, every speed change is a single `:X 02` command: no stop, no wait for rest, no mode or direction setup, even when the direction reverses. The legacy path is the familiar stop-and-restart.

**Move at a rate**

- `:X` path: encode the signed rate (counts per second × 1024) and send `:X 02`. Nothing else is sent before or after, and axis status isn't checked first.
- Legacy path: stop the axis and wait for full rest whenever the mode, direction or speed class changes. Then set the motion mode, divide the period by the high-speed ratio in fast mode, send the step period (never below 6), and start the motion.

**Stop**

- `:X` path: `:X 02` with a rate of exactly 0.
- Legacy path: `:K`.

**GOTO**

- `:X` path: `:X 04` with the target position and a rate on arrival of 0, so the axis lands and holds.
- Legacy path: stop and wait, set the GOTO mode, send the move distance, set a brake distance of 3,500 counts, then start.

The library keeps the last commanded rate per axis. Pulse guiding and the return to idle start from that value (next section).

## Pulse guiding and guide rates

The manufacturer's pulse guide is a rate change, a wait, and a return to idle. On an equatorial mount's Dec axis, the return to idle is a stop, which on an `:X` board is `:X 02` with rate 0.

1. Read the axis's last commanded rate (0 if none).
2. Add the signed guide rate and send it as one move command (`:X 02` on newer boards). If the send fails, it retries until it succeeds or the pulse is cancelled.
3. Wait for the pulse duration, less the time already spent sending.
4. Return the axis to idle:
    - RA with tracking on: restart the tracking task.
    - Dec on an equatorial mount, or any axis with tracking off: run the stop task (`:X 02` rate 0, or `:K` on legacy boards).
    - Alt-az mounts: resume tracking when it was on.

There is no settle wait, position check or correction after the stop.

**Guide rates.** The library offers five rates and uses the chosen one for serial pulses. It also has a command that sends a rate to the controller with `:P`, which sets the ST-4 port's rate. When the library sends it wasn't traced.

| Rate (× sidereal) | `:P` code |
| --- | --- |
| 1.0 | 0 |
| 0.75 | 1 |
| 0.5 | 2 |
| 0.25 | 3 |
| 0.125 | 4 |

The library's sidereal constant is 7.292 × 10⁻⁵ rad/s.

## Extended settings and feature flags

The library reads the tracking-torque support flag but has no way to send the torque setting (IDs 000006 / 000106).

**Status EX** (`:q` ID 000001) is read at connect into two groups.

- Fixed capabilities: dual encoders, PPEC, auto home, switchable alt-az / equatorial mode, polar-scope LED, "axes must start separately", and tracking-torque selection.
- Live state: PPEC training running, PPEC playback running.

**Extended settings the library sends** (`:W`), the complete list:

| ID | Setting | In the published command set |
| --- | --- | --- |
| 000000 / 000001 | PPEC training start / cancel | Yes |
| 000002 / 000003 | PPEC playback on / off | Yes |
| 000004 / 000005 | Dual encoder on / off | Yes |
| 000008 | Reset the home indexer | Yes |
| xxxx0D | Set the serial baud rate; the upper two bytes are the baud rate ÷ 100 (115200 → 04800D) | No |

**Extended inquiries** (`:q`): 000001 for Status EX, and 000000 for the home indexer on legacy boards.

> [!WARNING]
> The baud-rate setting isn't in the published command set. A wrong value leaves the board on a baud rate the host isn't using.

## Tracking, trail points and rate precision

Equatorial tracking is one constant-rate move command on RA and nothing on Dec. Alt-az tracking can instead stream time-stamped trail points, which the changelog credits with less oscillation on newer mounts.

**Equatorial tracking.** The library looks up the rate, flips its sign in the southern hemisphere, records it as the axis's last rate, and sends a single move command on RA (`:X 02` on newer boards). Dec isn't commanded.

| Tracking rate | rad/s | Relative to sidereal |
| --- | --- | --- |
| Sidereal | 7.2921158 × 10⁻⁵ | 1 |
| Solar | 7.2722052 × 10⁻⁵ | 0.99727 |
| Lunar | 7.0234578 × 10⁻⁵ | 0.96316 |

**Alt-az tracking.** The task recomputes both axis rates from the target and sends them as rate updates. When the next stretch can be tracked with trail points, it queues those instead (*inferred* from function names).

**Trail points.** Each point is a board-clock time plus both axis positions, queued on the board with `:X 0E`. The library tracks free slots and can add an offset during tracking with `:X 0A`. The SDK manual recommends at least 30 ms between points and speeds around 10× sidereal, and it supports satellite tracking.

**Rate precision.** The library works out the smallest rate change each board can make. The legacy path uses the gap between neighbouring `:I` periods near sidereal. The `:X` path uses a fixed step, since rates are sent in 1/1024 count per second.

## Other capabilities

The rest of the library's hardware access uses documented commands, and two of them switch to `:X` on newer boards.

| Capability | Legacy boards | `:X` boards | Notes |
| --- | --- | --- | --- |
| SNAP shutter output | `:O` with 0 / 1 | `:X 05` operation | Shutter open or closed |
| Polar-scope LED | `:V` + one brightness byte | same | Brightness given as 0–1, sent as a byte |
| Home indexer | `:q` 000000 to read, `:W` 000008 to reset | `:X 00` index 0B to read, `:W` 000008 to reset | Used by the auto-home task |
| Counts per revolution | `:a` | `:X 00` index 02 | |
| Step timer frequency | `:b` | same | |
| High-speed ratio | `:g` | same | Only used on the legacy fast path |
| Firmware version | `:e` | same | Decides legacy vs `:X` |

**Auto home.** A dedicated task slews toward the index sensor, watches for the reading to latch, backs off to a reliable side, and slews to the latched position. It stops and waits for full rest between phases (*inferred* from function names).

**PPEC and dual encoder** are switched with the `:W` IDs in the previous section, and Status EX reports whether each axis supports them.

## Model differences and defaults

The library has no per-model knowledge: no table of mount names, and no behaviour that changes with the model code in the `:e` reply. The firmware version is the only hardware input that changes how it drives a board.

**Altitude limits** are fixed constants shared by every mount:

| Limit | Default | Allowed range |
| --- | --- | --- |
| Minimum altitude | 0° | −10° to 30° |
| Maximum altitude | 75° | 60° to 90° |

## Notes for driver authors

On MC 3.39 and newer, following the manufacturer's sequence means sending `:X` for every rate change, stop and position read.

- Change the rate, or reverse direction, with one `:X 02`. No stop-and-restart is needed first.
- End a guide pulse by returning to the idle rate with `:X 02`: rate 0 on an equatorial Dec axis, the tracking rate on RA.
- Read positions with `:X 00` index 03 to get the full 32-bit value.
- The manufacturer's software never sends the torque setting (000006 / 000106), so there is no reference for what it does on any board.

## Open questions

None of this was checked on the wire. These points come from names or call order and still need confirming.

- Which user actions make the library send `:P`?
- Which boards the library allows trail-point tracking on (the SDK manual names the AZ-GTi as one).
- The full format of the RA and Dec fields in `:X 0A`, and whether `:X 0E` also carries rates.
- Whether the mount-clock rounding reflects a real board limit on accepted times.
- The baud-rate setting (xxxx0D): which rates the board accepts, and whether it survives a power cycle.
