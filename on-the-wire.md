# On-the-wire findings — Sky-Watcher EQ-AL55i Pro (MC 3.48)

## Provenance: independent, hardware-derived

Everything in this document was determined by sending commands to a physical mount over its
serial link and recording the replies and the resulting motion. **It uses no Sky-Watcher
software, SDK, firmware image or source code.** The behaviours here are black-box
observations of the hardware, reproducible by anyone with the mount and a serial terminal.

The commands are the mount's own serial protocol: the classic Sky-Watcher motor-controller
`:` command set, which has long been public, plus the `:X` extended sub-commands, whose
argument layouts and behaviours were mapped **here, by probing the board directly** and
watching what it did. Items below are marked **measured** (read or timed directly) or
*inferred* (a reasoned reading of the measurements).

## Board under test

Sky-Watcher EQ-AL55i Pro, motor-controller firmware 3.48. Firmware newer than 3.38 speaks
the `:X` extended protocol. Everything below was captured over USB serial at 115200 8N1.

Fixed values read from the board:

| Property | RA (axis 1) | Dec (axis 2) | Read with |
| --- | --- | --- | --- |
| Counts per revolution | 4,032,000 | 3,600,000 | `:X 00` index 02 (matches classic `:a`) |
| Timer base | 16,000,000 | 16,000,000 | classic `:b` |
| Home centre | 0x800000 | 0x800000 | — |

Positions are reported relative to the home centre 0x800000. One count is about 0.32″ on RA
and 0.36″ on Dec.

## Command framing observed on the wire

Argument fields are hex, most-significant digit first. A data reply begins with `=`; an error
is `!` followed by a code (`0` = unknown command). These layouts were established by sending
each form and observing the reply and motion.

| Sub | Form | Observed behaviour |
| --- | --- | --- |
| `00` | `:X<a>00` + 2-hex index | Read a register. Index 03/04 = **position relative to home, 32-bit**, equal to the classic `:j` value minus 0x800000. Index 02 = counts per revolution; index 06 = 1,000,000 (a 1 µs tick). |
| `01` | `:X<a>01` + 8 hex | Set position (32-bit, relative to home). **No range check** — a mistyped target silently replaces the position, so bound it host-side. |
| `02` | `:X<a>02` + 16 hex | Move at a signed rate. The rate is **counts per second × 1024**, signed, 64-bit. Confirmed by measurement: 1°/s on RA (11,200 × 1024 = `0x00AF0000`) read back as 11,204 counts/s. Rate 0 stops. |
| `04` | `:X<a>04` + 8 hex position + 16 hex rate | GOTO the position, then hold at the given rate-on-arrival. With rate 0 it lands and stops. |
| `05` | `:X<a>05` + 2 hex | `02` busy check (`!2` while moving, `=` when idle); `03` stop both axes (also clears the trail-point queue); `04` stop this axis; `05` set-initialised. |
| `0E` | `:X<a>0E` + 16 hex time + 8 hex RA pos + 8 hex Dec pos | Queue one absolute trail point. **Carries positions only — no rates.** The reply echoes both positions, the time, and a free-slot count. |
| `0F` | `:X<a>0F` + 8 hex | Snapshot. Reply = `=` + RA position (8 hex) + Dec position (8 hex) + mount clock (16 hex, µs since power-on), 34 characters in all. A host buffer sized for the classic 8-character reply overflows on it. |

## Initialisation is required before motion

An axis must be energised and initialised before it will accept motion commands, or it
silently ignores them:

- A bare `:X 02` (rate) or `:X 04` (goto) sent to a **cold, idle axis does nothing.** The
  axis must first be energised and initialised — `:F<axis>`, then `:X<a>05` with operation
  `05` (set-initialised). After that, rate and position commands take effect. **Measured:** an
  identical `:X 02` delivered 0 counts before initialisation and its full rate after.
- `:X 0E` trail points **only steer an axis that is already moving.** From rest the stream has
  no effect; prime the axis with `:X 02` first and the waypoints then take over. **Measured.**
- After a single `:X 0E` waypoint, with the queue then empty, the axis **coasts at the last
  interpolation velocity** rather than stopping. Keep the queue fed, and stop explicitly when
  done.

## Delivering a small guide correction — position move vs rate pulse

The practical headline. The measured guide behaviour — a rate pulse from rest under-delivers,
and the remainder creeps in over the following one to three seconds — is the signature of a
position servo with limited drive authority at standstill (*inferred*). It affects any axis
that starts each correction **from rest**: the Dec axis of an equatorial mount (which holds
still between corrections), and RA with tracking off.

Measured on the Dec axis (guide rate 0.5× sidereal, each correction issued from rest, encoder
read after it settles; medians of 20 repetitions, directions alternating):

| Correction | Distance | Rate pulse (`:X 02` for a duration) | Position move (`:X 04`, rate-on-arrival 0) |
| --- | --- | --- | --- |
| 500 ms | ~10 counts (~3.8″) | roughly 60–95%, and individual short pulses occasionally near 0 | **100–110%, never short** |
| 2 s | ~42 counts | ~100% | ~100% |
| 5 s | ~104 counts | ~100% | ~100% |

A rate pulse commands a speed for a time; from rest the servo has not covered the intended
distance before the stop lands, and short pulses suffer most. Commanding the correction as a
**position move** (`:X 04` to the current position ± the wanted distance, rate-on-arrival 0)
lets the servo close on the target instead of being cut off mid-travel. Across 20 corrections
on each axis the position move delivered the full distance with sub-count scatter and never
fell short.

**Recommendation:** deliver a small correction to a from-rest axis as an `:X 04` position
delta of `guide_rate × duration` counts, not a timed rate pulse.

These figures are read from the position encoder, which sits ahead of the gear train, so gear
or worm backlash is not reflected in them; a plate-solved or autoguided sky-motion run is the
end-to-end check.

## Tracking, and correcting while tracking

- **RA tracking:** a single `:X 02` at the sidereal rate. A guide correction *while tracking*
  is a rate offset (`:X 02` to sidereal ± guide) and delivers ~100%, because the axis is
  already moving — no from-rest shortfall. Return to the tracking rate to end the pulse.
- **Trail-point tracking (`:X 0E`):** streaming timestamped (time, RA position, Dec position)
  waypoints tracked the sidereal path to about ±1 count per waypoint (~1″ at sidereal,
  **measured**). A guide correction folded into the waypoint positions is delivered smoothly
  while tracking. Only the moving axis follows the stream — a held (stationary) axis is not
  steered by it, so on an equatorial mount the Dec correction still belongs on `:X 04`.

## Tracking-torque setting (IDs 000006 / 000106) — not implemented on MC 3.48

The `:X` command set carries a tracking-torque setting and a matching support flag in the
Status EX word. On this board:

- Status EX (`:q<a>` ID 000001) reads `=009000` on both axes. The tracking-torque-selection
  capability bit — the value-4 bit of the third reply character — is **clear**, i.e. not
  supported.
- `:W<a>060100` (enable) and `:W<a>060000` (disable) both return `!0` (unknown command) on
  both axes.

So MC 3.48 does not implement the tracking-torque setting, and the support flag correctly
reads "unsupported". Whether newer firmware implements it is still open.

## Method

- One physical EQ-AL55i Pro on a bare mount (no optics or counterweights). Commands were sent
  as raw `:` strings over USB serial (115200 8N1) and replies captured verbatim.
- Positions and timing came from `:X 00` index 03 and the `:X 0F` snapshot, which timestamps
  both axis positions against the mount clock.
- Every motion test ran under a guard that stopped both axes if a position or speed left an
  expected window.
- Delivery figures are medians over 20 repetitions per class, with directions alternating.
