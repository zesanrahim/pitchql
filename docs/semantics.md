# PitchQL semantics (working draft)

Plain-English rules first, symbols later. One decision per entry: what, why (and which corpus pattern forced it), what was rejected.

## Open questions

- [ ] Total order over events with equal or duplicate timestamps
- [ ] What counts as one possession, and do patterns reset when it changes
- [ ] Which clock do time windows use (game clock vs. wall clock)
- [ ] Negation over intervals; repetition
- [ ] Players outside the 360 `visible_area` (true / false / unknown)
- [ ] Press-trap: "ahead of the ball" measured at the regain or at the next pass
- [ ] Press-trap: "regain" as any touch or as established control

## 1. Data model

### 1.1 Match
### 1.2 Team
### 1.3 Player
### 1.4 Event
### 1.5 Frame (360 freeze frame; tracking frame at 25 Hz)
### 1.6 Coordinates and pitch

Source systems:

| source | units | origin | direction |
|---|---|---|---|
| StatsBomb events and 360 | abstract 120 x 80 grid | corner | per team: each team's events are written as if it attacks toward x = 120 (checked: both goalkeepers sit at x of about 10 in every period) |
| IDSSE events | metres, 105 x 68 | corner | absolute (check against `KickOff` `TeamLeft`/`TeamRight` per period) |
| IDSSE tracking | metres, 105 x 68 | centre spot | absolute |

**Canonical system (decided):** metres, origin at the centre spot, x from -52.5 to +52.5, y from -34 to +34.

Conversions into canonical (before any direction handling):
- StatsBomb: `x_m = (x - 60) * 105 / 120`, `y_m = (40 - y) * 68 / 80` (y flipped, see below)
- IDSSE events: `x_m = x - 52.5`, `y_m = y - 34` (no flip, verified below)
- IDSSE tracking: none (no flip, verified below)

**+y is the attacker's left (decided).** Reason: the number matches the word, so "on the left side" is `y > 0`. Rejected: StatsBomb's raw convention, where +y is the attacker's right.

**Attacking direction (decided):** every position is expressed relative to a reference team, with +x toward the goal that team attacks. Whichever team a pattern names as "us" (the subject team) is the reference team. This is about pitch orientation, not possession: our own goal is at -x and the opponent's goal at +x for the whole half, whoever has the ball. So "their half" is `x > 0`.
- StatsBomb events by the other team must be mirrored (x and y).
- IDSSE needs mirroring by period and team.
- `period` is kept as its own field on every event and frame, so half-based queries use it, not the sign of x.
- Rejected: storing the raw (source) pitch orientation. No corpus pattern needs it (the only half-related one, "before half-time", uses the clock). Add it later as an extra field if one does.

**IDSSE y direction (verified on match J03WMX):**
- Absolute direction: the team in `KickOff` `TeamLeft` attacks +x in that half (Bayern in the first half, Köln in the second). Teams swap ends at half time.
- Events: in the first half Bayern attacks +x, and its left-sided players (LV, OLM, IVL, DML) average y of 42-62 while its right-sided players (RV, ORM, IVR, DMR) average 9-31. So +y is already the attacker's left. In the second half the sides flip (e.g. RV Pavard goes from y 9 to 60), as expected when attacking -x.
- Tracking: Sommer's event positions (x - 52.5, y - 34) match his tracked X, Y within 1-3 m, with the same sign in both axes. So tracking and events share one orientation.
- Consequence: for IDSSE, mirror both x and y for the team attacking -x. For StatsBomb, mirror the other team's events (x and y).
### 1.7 Time
- Clocks (game clock, wall clock), periods, stoppage

## 2. Type system

## 3. Abstract syntax

## 4. Semantics

## Decision log

Template:

### D-001: <title>
- **Decision:**
- **Why / forced by:** <corpus pattern id>
- **Rejected:**
