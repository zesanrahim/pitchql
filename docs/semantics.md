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

A **match** is the unit a query runs over. It bundles everything recorded about one game: who played, when each period started and ended, and the data streams.

```
Match = {
  id        : MatchId
  source    : statsbomb | metrica
  home, away: Team                      -- see 1.2
  periods   : [Period]                  -- in order
  events    : [Event]                   -- see 1.4; totally ordered (section 4)
  frames360 : Event -> Frame?           -- StatsBomb 360 only; at most one per event
  tracking  : [Frame]?                  -- Metrica only; 25 Hz
  meta      : { date?, competition?, season?, stadium?, ... }   -- not queryable
}

Period = { n : 1..4, start : Instant, end : Instant }
```

Where each field comes from:

| field | StatsBomb | Metrica |
|---|---|---|
| `id` | `match_id` | sample game number (1, 2, 3), prefixed `metrica-` |
| `home`, `away` | `home_team`, `away_team` (id and name) | `Home`, `Away` (no names, anonymised) |
| `periods` | `Half Start` / `Half End` events per `period` | first and last frame of each `Period` value in the tracking file |
| `events` | events file, ordered by `index` | Metrica event file, if present (to verify once `data/` is restored) |
| `frames360` | 360 file, joined on `event_uuid` = event `id`; empty if the match has no 360 data | none |
| `tracking` | none | tracking files (home and away), joined on frame number |
| `meta` | `match_date`, `competition`, `season`, `stadium`, ... | none (anonymised) |

**One query, one match (decided).** A query is evaluated on one match at a time. Running it on a set of matches returns the union of the per-match results, each tagged with its match id. No pattern variable can bind to things from two different matches.
- Why: every corpus pattern happens inside one game (P01 is written `ON match M`). Possession, the game clock and "us" only make sense inside a match.
- Rejected: cross-match queries ("the same press in two matches in a row"). No corpus pattern needs them, and they belong to precedent retrieval (Project 4).

**"Us" is a parameter of the run, not of the match (decided).** A query names roles `US` and `OPP`. A run binds `US` to one of the two teams. Running a query on a match means running it twice, once per binding, unless the analyst fixes `US` (e.g. `US = home`). Every result records which team was `US`.
- Why: almost every pattern is written from one side ("our press", "we regain": P01, P02, P23). Coordinates already depend on this choice (1.6: +x is toward the goal `US` attacks).
- Rejected: fixing `US = home`. That would miss half the counterpresses in the StatsBomb flag experiment (P02).

**Periods 1-4 only (decided).** Periods 1 and 2 are the halves, 3 and 4 are extra time. Period 5 (penalty shootout) is dropped when loading.
- Why: a shootout has no open play, no possession in the usual sense, and no corpus pattern refers to it.
- Rejected: keeping it as an ordinary period. Shot-based patterns such as P20 (shot assist) could then match shootout kicks.

**No pattern spans a period boundary (decided).** Each pattern match lies inside one period.
- Why: the ball is out of play at half time, StatsBomb's `possession` restarts, and the teams swap ends. A press that "carries over" a half-time break means nothing.
- Rejected: allowing windows across periods. Time windows over a gap of 15 minutes would be well defined on a wall clock but tactically meaningless.

**Score is derived, not stored (decided).** The final score fields (`home_score`, `away_score`) are not part of the match. Instead, `score(team, t)` is computed from goal events up to instant `t` (goals from shots with outcome `Goal`, plus `Own Goal For` events).
- Why: P21 ("dropping into a block when leading late") needs the score *at a moment*, not at the end.
- Rejected: the final score as a match attribute. It would leak the future into a streaming run.

**StatsBomb and Metrica are never joined (fact, not a choice).** They never cover the same game. A pattern that needs both events and tracking (P04, P18, P21) can only run on a Metrica match with its own events.

### 1.2 Team

A **team** is one of the two sides in a match. It only has meaning inside its match: since a query runs on one match (1.1), nothing about a team carries over to another match.

```
Team = {
  id   : TeamId
  name : string?                -- none for Metrica (anonymised)
  side : home | away
}

Role = US | OPP
team : Role -> Team             -- fixed by the run (1.1); team(OPP) is the other team
```

Where each field comes from:

| field | StatsBomb | Metrica |
|---|---|---|
| `id` | `home_team_id` / `away_team_id` in the match file; `team.id` on events | `Home`, `Away` (the `Team` column and the tracking file names) |
| `name` | `home_team_name` / `away_team_name` | none |
| `side` | which of `home_team` / `away_team` it is | `Home` or `Away` |

**Queries name roles, never teams (decided).** The language has no team literals. A condition such as `team = US` compares an event's team with whatever team the run bound to `US`. To look at one club, the analyst fixes the binding when running the query (e.g. `US = home`); the query does not change.
- Why: every corpus pattern is written from a point of view ("our press", "their half"), never about a named club. A query that names "Italy" can't be reused on another match.
- Rejected: team names or ids as literals. They tie a query to one dataset, and Metrica has no names at all.

**Exactly two teams; officials and the ball are not teams (decided).** Each player belongs to exactly one of the two teams for the whole match. Referees (present in some tracking feeds) are dropped when loading. The ball is its own object, not a member of either team (see 1.5).
- Why: `US`/`OPP` and the mirroring rule in 1.6 both need every player to be on exactly one side.
- Rejected: a third "neutral" team for officials. No corpus pattern refers to them.

**No declared formation (proposed).** StatsBomb records the formation each team says it plays (`Starting XI` and `Tactical Shift` events, e.g. 4-2-3-1). It is not part of `Team` and cannot be queried.
- Why: the shape patterns (P05 low block, P08-P10 rest defence, P26 high line, P28 compactness) describe where players actually stand, not the label on the team sheet. They must be computed from positions (1.5), and Metrica has no formation data anyway.
- Rejected: `formation` as a team attribute. Queries could then match the provider's label instead of the shape, and would not run on Metrica.

**Team metadata is not queryable (decided).** Managers, country, gender and group (StatsBomb) go into `Match.meta` with the rest of the descriptive fields.
- Why: no corpus pattern uses them.

**Team-level measures are functions, not fields.** Quantities such as a team's width, the height of its back line, or its compactness change from frame to frame, so they are not stored on `Team`. They are defined later as functions of a team's player positions at an instant (1.5 and section 2).

**Which team has the ball is not a team attribute.** StatsBomb's `possession_team` is a property of an event and of a possession, and is covered with possession (open question above).

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
