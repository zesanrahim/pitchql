# PitchQL semantics (working draft)

Plain-English rules first, symbols later. One decision per entry: what, why (and which corpus pattern forced it), what was rejected.

**How to read the status labels.** Text from the team's earlier commits (1.1-1.3 as written, 1.6) stands as written. Anything marked **[PROPOSED by Kieran, to be approved]** is Kieran's working proposal and is not agreed with Zesan and Bashir. This covers 1.4, 1.7, the proposed edits inside 1.1-1.3, and decisions D-001 to D-022 in the decision log. Nothing marked proposed should be treated as settled until the team approves it.

## Open questions

- [~] Total order over events with equal or duplicate timestamps: sequencing uses (period, t, seq), windows use the clock only (see 1.7). [PROPOSED by Kieran, to be approved; provisional]
- [~] Which 25 Hz tracking source: IDSSE. [PROPOSED by Kieran, to be approved] (see D-001). Metrica stays as a fallback sample
- [ ] What counts as one possession, and do patterns reset when it changes
- [~] Which clock do time windows use: game clock (see 1.7). [PROPOSED by Kieran, to be approved]
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

**[PROPOSED by Kieran, to be approved] Edits to 1.1 if IDSSE is the tracking source (D-001, D-002 to D-007):**
- Add `idsse` to `Match.source` (`statsbomb | idsse | metrica`), and `tracking` comes from IDSSE as well as Metrica.
- The "never joined" rule becomes: events and tracking join only within one source and one match. IDSSE ships its own synced events, so P04, P18 and P21 run on IDSSE with those events. StatsBomb is never mixed with tracking (D-007).
- IDSSE events lack Pressure, Carry, Ball Receipt and Dribble events, so those are derived from tracking on IDSSE (D-019).
- Dead-ball stops inside a period do not break a pattern; only a period boundary does (D-005).
- The other 1.1 choices (one match, `US` run both ways, periods 1-4, derived score) are kept as written (D-002, D-003, D-004, D-006).

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
- Why: the shape patterns (P05 low block, P08-P09 rest defence, P22 horizontal compactness) describe where players actually stand, not the label on the team sheet. They must be computed from positions (1.5), and Metrica has no formation data anyway.
- Rejected: `formation` as a team attribute. Queries could then match the provider's label instead of the shape, and would not run on Metrica.

**Team metadata is not queryable (decided).** Managers, country, gender and group (StatsBomb) go into `Match.meta` with the rest of the descriptive fields.
- Why: no corpus pattern uses them.

**Team-level measures are functions, not fields.** Quantities such as a team's width, the height of its back line, or its compactness change from frame to frame, so they are not stored on `Team`. They are defined later as functions of a team's player positions at an instant (1.5 and section 2).

**Which team has the ball is not a team attribute.** StatsBomb's `possession_team` is a property of an event and of a possession, and is covered with possession (open question above).

### 1.3 Player

A **player** is one person on one of the two teams. Like a team, a player only has meaning inside its match.

```
Player = {
  id   : PlayerId
  team : Team                       -- fixed for the whole match (1.2)
}

keeper   : Player x Instant -> bool         -- is the player in goal at t
on_pitch : Player x Instant -> bool         -- is the player on the pitch at t
players  : Role x Instant -> PlayerSet      -- the team's players on the pitch at t
```

Where each part comes from:

| part | StatsBomb | Metrica |
|---|---|---|
| `id` | `player.id` | tracking column name (`Player1` ... `Player28`), prefixed `metrica-` |
| `team` | team of the `Starting XI` or `Substitution` event that brings the player on | which tracking file (home or away) the column is in |
| `keeper` | position `Goalkeeper` in `Starting XI`, `Tactical Shift` and `Substitution` events | not given; must be inferred (to verify on the sample) |
| `on_pitch` | from `Starting XI`, `Substitution`, `Player Off` / `Player On`, and red cards | position is not `NaN` in the tracking file |
| name, shirt number | `player.name`, `jersey_number` (into `meta`) | shirt number from the tracking header (into `meta`) |

**Queries name no players (decided).** As with teams (1.2), the language has no player literals. Patterns talk about players through what they do or where they are ("the passer", "players ahead of the ball"), never by name. Names and shirt numbers go into `meta`.
- Why: no corpus pattern names a player, and Metrica players are anonymised.
- Rejected: player names or ids in queries. To look at one player, the analyst filters the results.

**360 freeze frames do not identify players (fact, not a choice).** A 360 frame lists positions with three flags: `teammate` (same team as the event's actor), `actor` (the player doing the event) and `keeper`. There is no player id. So in a 360 frame, a query can count and quantify over positions ("at least three of our players ahead of the ball", P01), but cannot ask which player stands where, or follow a player from one frame to the next. Only the actor is linked to a `Player`, through the event. The `teammate` flag is turned into `US` / `OPP` when loading, using the actor's team. (Frame contents are defined in 1.5.)

**The only declared position is goalkeeper (decided).** `keeper(p, t)` is part of the data model. Other positions (full-back, centre-back, winger) are not: they are worked out from where players stand, like the shape patterns in 1.2.
- Why: several patterns need keeper vs outfield players (P23 "outfield players", P30 "second-last defender", which counts the keeper). It is the one position all sources can give: StatsBomb events and the 360 `keeper` flag have it directly, and Metrica can infer it. Patterns that name other positions (P04 and P31 "full-back") mean where the player is, not the label.
- Rejected: StatsBomb's `position.name` (about 25 values) as a queryable field. 360 frames and Metrica don't have it, and it would make "full-back" mean the team-sheet label. This follows the same reasoning as "No declared formation" in 1.2.

**Who is on the pitch changes over time (decided).** `players(R, t)` is the set of players of team `R` on the pitch at instant `t`. It returns `Player` values (id and team), not a count and not positions: `count(players(R, t))` gives the number, and where a player stands comes from a frame (1.5). Substitutions, temporary exits (`Player Off` / `Player On`, e.g. injury treatment) and red cards change it.
- Why: counts such as P01's "at least three players ahead of the ball" must not include substitutes or sent-off players.
- Rejected: a fixed set of eleven per team. It would be wrong after every red card.

**Lineup changes take effect at their place in the event order (proposed).** A substitution, temporary exit or red card changes `players(R, ·)` from the next event in the total order (section 4), not from its timestamp. So if a `Substitution` and a pass share a timestamp, the substitute is on the pitch for the pass only if the pass comes after the substitution in StatsBomb's `index` order. A tracking frame sees every change from events at an earlier or equal time.
- Why: two events can share a timestamp, and the answer must not depend on how ties happen to be broken. In practice substitutions happen while the ball is out of play, so such ties should be rare.
- Depends on: the total-order rule (open question above).
- **[PROPOSED by Kieran, to be approved]** Adopted as written (D-016). The total-order rule it depends on is now proposed in 1.7 as (`period`, `t`, `seq`). The last sentence ("a tracking frame sees every change from events at an earlier or equal time") is held back until the event-to-frame alignment in 1.7 is settled.

**Metrica: on the pitch from first to last tracked frame (proposed).** A Metrica player is on the pitch from their first non-`NaN` frame to their last non-`NaN` frame in each period. A `NaN` inside that span means the player is on the pitch but their position is unknown, so conditions on that player's position evaluate to unknown, as for a player outside the 360 visible area. If the Metrica event file has substitution events, they are used instead (to verify).
- Why: a `NaN` can mean "substituted off" or "not tracked for a moment", and the two must not be confused. A substituted player cannot come back on, so a player's span has no real gaps from substitutions.
- Known limit: a temporary exit (e.g. injury treatment) also shows as a `NaN` gap, so the player counts as on the pitch with an unknown position while actually off. This can turn a definite answer into unknown, but never into a wrong true or false.
- Rejected: treating every `NaN` frame as off the pitch. A short tracking dropout would then remove a player from counts such as P01's.
- **[PROPOSED by Kieran, to be approved]** Kept, but only used if Metrica is the tracking source. If IDSSE is chosen (D-001), starters come from the match info (`Starting="true"`) and changes from substitution events (D-017). IDSSE marks starting keepers with `PlayingPosition="TW"`; a substitute keeper must be inferred from substitution events (D-014). The reason "Metrica players are anonymised" for "queries name no players" is weaker with IDSSE, but the rule stands because patterns are not player-specific (D-013). Also: IDSSE tracking has player ids, so following a player between frames is possible there; only 360 frames lack ids.

### 1.4 Event **[PROPOSED by Kieran, to be approved]**

An **event** is one recorded thing in the match data (a pass, a tackle, a shot). It is an input that query conditions look at, not what a query returns.

**One shared shape (D-018).** Every source's events are converted into the same shape, so `type = Pass` means the same on StatsBomb and IDSSE. Source-specific detail is kept as optional extra fields.

**One list of types (D-019).** Pass, cross, shot, tackle or duel, recovery, foul, set piece, substitution, card, goal, pressure, carry, receipt, dribble, miscontrol. Each type has one definition. A source either records a type or derives it from tracking. If a source cannot supply a type, conditions on it are unknown, not false.

**Stored fields (D-020):** `id`, `type`, `subtype`, `team` (`US` / `OPP`), `player`, time (`period`, `t`, `seq`, see 1.7), `location` (canonical, see 1.6), `outcome`, `end_location`, `recipient`, `restart` (open play, or corner, free kick, throw-in, goal kick, kick-off, penalty). Checked against corpus v0.3.0: end location (P03, P07, P12, P27), recipient (P12, P18, P31), restart (P02, P06, P13, P32).

**Derived, not stored (D-021):** turnover, touch count and under-pressure are named definitions written in section 3; what each means is decided there.

**Missing values (D-022):** a field is a value, **none** (does not apply, so a condition is false) or **unknown** (source should have it, so a condition is unknown).

**Source loading notes:** IDSSE events include 108 `Delete` events that retract earlier events; apply them at load (D-007). IDSSE goals come from `ShotAtGoal/SuccessfulShot` (own goals still to check) (D-006).

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
### 1.7 Time **[PROPOSED by Kieran, to be approved]**

**Clock (decided): the game clock.** A time is a pair (`period`, `t`), where `t` is seconds since the start of that period. All time windows ("within 8 s") use it. A window cannot span two periods, so a pattern that would cross half-time never matches.
- **Why:** coaches refer to plays by the game clock; all three sources can be converted to it.
- **Includes dead-ball time:** the clock keeps running while the ball is out of play (e.g. a throw-in being taken).
- **Forced by:** P01 press-trap ("within eight seconds"), P02 counterpress (5 s).
- **Rejected:** wall clock (runs through half-time, injuries and VAR checks); ball-in-play clock (no source gives it directly). The latter can be added later as a named option.

How each source converts to (`period`, `t`):

| source | what it gives | conversion |
|---|---|---|
| StatsBomb | `timestamp` restarts at 0 each period; `minute`/`second` count from kick-off of the match | `t` = `timestamp` in seconds. Ignore `minute`/`second` |
| IDSSE events | wall-clock `EventTime` (+02:00) | `t` = `EventTime` minus the wall-clock time of that period's kick-off (`KickOff` event with the `GameSection`) |
| IDSSE tracking | wall-clock `T` (UTC), frame counter `N` | same as events, after converting to a common timezone |
| Metrica | `Time [s]` keeps counting across periods (period 2 starts at 2885.4 s) | `t` = `Time [s]` minus the time of the first frame of that period |

**To verify:** that the IDSSE kick-off event time and the first tracked frame agree closely enough to use as the period start (they may differ by a fraction of a second).

**Ordering and ties (decided for now, revisit after hand-checking tied cases in the viewer).** Two rules:
1. **Sequencing ("then") uses a total order** on (`period`, `t`, `seq`). `seq` is a tie-break number assigned by each source's loader, so events with equal `t` still have a definite order.
2. **Time windows use the clock only.** A 0 s gap counts as inside any window, whichever event comes first in the order. So an arbitrary tie order cannot flip a window or a negation ("no pressure in the 3 s before the pass").
- **Why:** ties are common (43% of StatsBomb time slots in the sample match). Most are one real moment recorded as several events (receipt and carry, foul committed and foul won). Natural patterns such as "receipt, then pass" must still match. A 0 s gap is below what the data can resolve.
- **Forced by:** P01, P02 (windows); any pattern using `sequence`.
- **Rejected:** (1) treating tied events as simultaneous, with no order (breaks "receipt, then pass"); (2) one order for everything, windows included (window results would depend on the provider's arbitrary tie order).
- **Possible later refinement:** report a match as "unknown" when it only holds under one tie order.

How `seq` is assigned per source:

| source | `seq` |
|---|---|
| StatsBomb events | `index` (clean and gap-free). Fix the 3 timestamps that jump back by about 50 minutes at load |
| StatsBomb 360 | none of its own; inherits the time and order of its event (`event_uuid`) |
| Metrica events | row order in the file (rows are already sorted by period and start frame) |
| IDSSE events | sort by time first (file order is not time order: 199 neighbouring events go backwards, by up to 108 minutes), then by `EventId` |
| tracking frames | not needed (one frame every 0.04 s, no ties) |

**Data oddity to handle at load:** one Metrica event has an end frame before its start frame.

**Instant or interval (decided for now, revisit if hand-checking shows a boundary case).** Every event is an instant at its start time. The data model still stores `duration` / end time (StatsBomb `duration`, Metrica `End Time [s]`; IDSSE events have a single time, so end = start), but the language does not use it yet.
- **Why:** simplest option. A crude keyword check of the corpus found no pattern that needs an event's end (only P28, which is about tracking movement). Evidence it matters at the edges: in the sample match a pass lasts about 1.3 s typically (up to 7 s), a carry up to 26 s, a pressure about 0.8 s.
- **Consequence to know about:** "within 8 s of the first pass" in P01 is measured from the *start* of the pass. A regain at 8.6 s after the start of a 1.3 s pass does not match, though it is 7.3 s after the pass arrives. The author cannot write the other reading yet.
- **Rejected:** everything an interval (every window must name an end; heavier); hybrid with an optional `end` and `during` (not needed yet).
- **Extension path:** add `end` and `during` later. Adding them does not change any existing pattern, because the default stays "start time". Do this before the freeze date.
- **Not affected:** tracking state is interval-valued regardless (required by the program document); that is separate from event durations.

**Still open in 1.7:** how events line up with 25 Hz frames (including: does an event at time `t` see the frame at `t`?).

## 2. Type system

## 3. Abstract syntax

## 4. Semantics

## Decision log

Template:

### D-001: Tracking source is IDSSE (PROPOSED, not agreed with the team)
- **Status:** proposed by Kieran, working assumption only. Zesan and Bashir have not confirmed. Do not treat as settled.
- **Decision:** use IDSSE (DFL, 105x68 centred, all players tracked, speed and acceleration per frame, synced official events) as the 25 Hz tracking source. StatsBomb stays the source for events and labels. Metrica stays as a small fallback sample.
- **Why:** every player is visible; provider speed/acceleration lets us check our kinematic predicates against something we did not compute; synced events allow event-plus-tracking patterns (P04, P18, P21) on one match; 7 matches allow a holdout.
- **Costs:** IDSSE events are DFL format, not StatsBomb's schema, so they need a mapping; about 400 MB of XML tracking per match; `CLAUDE.md` goal 2 ("2-3 tracking predicates on Metrica") would need rewording.
- **Affects:** Kevin's 1.1 (source list is `statsbomb | metrica`; "never joined" claim) and 1.3 (anonymised-players reasoning); 1.5 Frame; 1.7 clock conversions (already cover IDSSE).
- **Rejected (for now):** Metrica only (anonymous, positions only, nothing independent to check against).

### D-002: One query runs on one match (1.1)
- **Decision:** a query is evaluated on one match at a time. Results over many matches are the union, tagged with match id.
- **Why / forced by:** every corpus pattern happens inside one game (P01).
- **Rejected:** cross-match queries (no corpus pattern needs them).
- **Status:** agreed Kieran (reviewing cheeno's 1.1); team not yet confirmed.

### D-003: `US` is set per run; default is to run both teams (1.1)
- **Decision:** the query names `US` and `OPP`. A run binds `US` to one team. By default we run twice, once per team, and tag each result with the `US` team. The analyst can fix `US` to run one side.
- **Why / forced by:** patterns are written from one side (P01, P02, P23); the 1.6 mirroring needs `US` known; scoring against StatsBomb's counterpress flag (P02) needs both teams.
- **Rejected:** fixing `US = home` (misses half the matches).
- **Status:** agreed Kieran; team not yet confirmed.

### D-004: Periods 1-4 only (1.1)
- **Decision:** periods 1-2 are the halves, 3-4 are extra time. Period 5 (penalty shootout) is dropped at load.
- **Why / forced by:** shootouts have no open play; shootout kicks could falsely match shot patterns (P20).
- **Rejected:** keeping period 5 as an ordinary period.
- **Status:** agreed Kieran; team not yet confirmed.

### D-005: No pattern crosses a period boundary, but patterns may cross dead-ball stops within a period (1.1, 1.7)
- **Decision:** a match of a pattern lies inside one period. Dead-ball stops inside a period (corners, free kicks, throw-ins, injury stops) do not break a pattern; the game clock keeps running (1.7).
- **Why / forced by:** half-time and the extra-time break are real stoppages (teams swap ends, possession restarts). Set-piece patterns ("corner, then a shot within 8 s") must cross dead-ball stops.
- **Rejected:** allowing windows across periods; banning all stoppages (breaks set-piece patterns).
- **Status:** agreed Kieran; team not yet confirmed. Extends cheeno's 1.1 wording, which only covered period boundaries.

### D-006: Score is computed at a moment from goal events (1.1)
- **Decision:** `score(team, t)` counts goals before instant `t`. Final score is not stored.
- **Why / forced by:** P21 needs the score mid-game; a stored final score leaks the future into a streaming run.
- **Note:** StatsBomb goals = shots with outcome Goal plus own goals. IDSSE goals need their own mapping (`ShotAtGoal/SuccessfulShot`, own-goal events to check).
- **Status:** agreed Kieran; team not yet confirmed.

### D-007: Events and tracking join only within an IDSSE match (1.1)
- **Decision:** a pattern gets events and tracking only from the same source and match. Event-only patterns run on StatsBomb (and can be scored against its labels). Mixed patterns (P04, P18, P21) run on IDSSE with its own events. StatsBomb is never mixed with tracking.
- **Why:** StatsBomb events and IDSSE/Metrica tracking never cover the same game.
- **Checked on the samples:** IDSSE events (full match: 1715 events) cover passes, crosses, shots, tackles, ball claiming, fouls and set pieces, but have no Pressure, Carry, Ball Receipt or Dribble events and no counterpress flag. 108 `Delete` events retract earlier events and must be applied at load. Metrica events have the same gaps. On IDSSE, pressure, carry and receipt are derived from tracking.
- **Depends on:** D-001 (IDSSE, proposed). If the team picks Metrica, the rule is the same with Metrica's own events.
- **Status:** agreed Kieran; team not yet confirmed.

### D-008: Queries name roles, never teams (1.2)
- **Decision:** the language has no team literals. `team = US` compares with whichever team the run bound to `US` (D-003).
- **Why / forced by:** every corpus pattern is written from a point of view (P01, P02, P23); a query naming a club can't be reused.
- **Rejected:** team names or ids in queries.
- **Status:** agreed Kieran (reviewing Kevin's 1.2); team not yet confirmed.

### D-009: Exactly two teams; officials and the ball are not teams (1.2)
- **Decision:** every player is on exactly one of two teams. Referees are dropped at load. The ball is its own object.
- **Why:** `US`/`OPP` and the mirroring in 1.6 need every player on one side.
- **Rejected:** a third "neutral" group for officials.
- **Status:** agreed Kieran; team not yet confirmed.

### D-010: No declared formation; shape comes from coordinates (1.2)
- **Decision:** the team sheet's formation (e.g. 4-2-3-1) is not queryable. Shape patterns (P05, P08-P09, P22) are computed from player coordinates in a frame.
- **Why:** the label can be wrong, and real shape changes within a match. IDSSE and Metrica have no formation label anyway.
- **Rejected:** `formation` as a team attribute.
- **Status:** agreed Kieran (it was marked "proposed" by Kevin); team not yet confirmed.

### D-011: Team details are not queryable (1.2)
- **Decision:** manager, country, gender and similar go into descriptive match data only.
- **Why:** no corpus pattern uses them.
- **Status:** agreed Kieran; team not yet confirmed.

### D-012: Team measures are functions, not stored fields (1.2)
- **Decision:** width, back-line height and compactness are computed from a team's player coordinates at an instant (see 1.5 and section 2).
- **Why:** they change every frame.
- **Status:** agreed Kieran; team not yet confirmed.

### Deferred from 1.2
- "Which team has the ball" is not a team attribute. It belongs to the possession question (open).

### D-013: Queries name no players (1.3)
- **Decision:** the language has no player literals. Patterns describe players by what they do or where they are ("the passer", "players ahead of the ball"). Names and shirt numbers go into descriptive match data.
- **Why:** patterns are not player-specific; no corpus pattern names a player.
- **Rejected:** player names or ids in queries. To look at one player, filter the results.
- **Status:** agreed Kieran (reviewing Kevin's 1.3); team not yet confirmed.

### D-014: Goalkeeper is the only declared position (1.3)
- **Decision:** `keeper(p, t)` is part of the data model. Other roles (full-back, winger, centre-back) are computed from coordinates, not taken from labels.
- **Why / forced by:** P23 and P30 need keeper versus outfield; every source can give it. Labels can be wrong and roles shift during play. Patterns that say "full-back" (P04, P31) will need a coordinate-based definition (to design).
- **Checked:** IDSSE match info marks starting keepers with `PlayingPosition="TW"`. A substitute keeper has no position listed, so a keeper coming on mid-match must be inferred from substitution events.
- **Rejected:** StatsBomb `position.name` (about 25 values) as a queryable field; 360 frames and Metrica don't have it.
- **Status:** agreed Kieran; team not yet confirmed.

### D-015: Who is on the pitch changes over time (1.3)
- **Decision:** `players(R, t)` is the set of team `R`'s players on the pitch at instant `t`. Substitutions, temporary exits and red cards change it. It returns players, not a count or positions.
- **Why / forced by:** P01's "at least three players ahead of the ball" must exclude subs and sent-off players.
- **Rejected:** a fixed eleven per team.
- **Status:** agreed Kieran; team not yet confirmed.

### D-016: Lineup changes take effect at their place in the event order (1.3)
- **Decision:** a substitution, temporary exit or red card applies from the next event in the total order (1.7: period, t, seq), not from its timestamp. If a substitution and a pass share a timestamp, the substitute counts for the pass only if the pass comes later in the order.
- **Why:** tied timestamps must not give ambiguous answers. Ties should be rare (substitutions happen while the ball is dead).
- **Rejected:** applying the change at its timestamp (ambiguous on ties).
- **Open:** how tracking frames see these changes depends on the still-open event-to-frame alignment in 1.7. Kevin's wording ("a frame sees changes from events at an earlier or equal time") is not adopted until that is settled.
- **Status:** agreed Kieran (Kevin marked it "proposed"); team not yet confirmed.

### D-017: On-pitch status per source (1.3)
- **Decision (IDSSE, main, depends on D-001):** starters come from the match info (`Starting="true"`); changes come from substitution events.
- **Decision (Metrica, only if Metrica is used):** Kevin's rule. A player is on the pitch from their first to their last non-`NaN` frame in each period. A `NaN` inside that span means on the pitch with unknown position, so conditions on that player's position evaluate to unknown. Known limit: a temporary exit also shows as a `NaN` gap.
- **Rejected:** treating every `NaN` frame as off the pitch (a short dropout would remove a player from counts like P01's).
- **Status:** agreed Kieran; team not yet confirmed.

### Fact recorded from 1.3 (no choice)
- StatsBomb 360 frames list positions with `teammate`, `actor` and `keeper` flags but no player ids. A query can count and quantify over positions in a 360 frame, but cannot follow one player between frames. IDSSE tracking has player ids, so following a player is possible there.

### D-018: Every source's events are converted to one shared shape (1.4)
- **Decision:** queries see one common event shape, not each source's raw fields. `type = Pass` means the same on StatsBomb and IDSSE. Source-specific detail is kept as optional extra fields.
- **Why:** one query must run on any source, and the formal semantics needs one definition of an event.
- **Rejected:** raw per-source fields (a query would only work on the source it was written for).
- **Status:** agreed Kieran; team not yet confirmed.

### D-019: One list of event types, each recorded or derived per source (1.4)
- **Decision:** there is one vocabulary of event types with one definition each (pass, cross, shot, tackle or duel, recovery, foul, set piece, substitution, card, goal, pressure, carry, receipt, dribble, miscontrol). A source either records a type or derives it from tracking (D-007). If a source cannot supply a type at all, conditions on it are unknown, not false.
- **Why:** the language must not change with the source.
- **Open:** a derived pressure will not exactly match StatsBomb's recorded one. Needs a definition from positions, and a check of how closely it reproduces StatsBomb's.
- **Rejected:** a "core" list plus a "StatsBomb-only" list (the language itself would change by source).
- **Status:** agreed Kieran; team not yet confirmed.

### D-020: Stored fields on every event (1.4)
- **Decision:** `id`, `type`, `subtype`, `team` (as `US` / `OPP`), `player`, time (`period`, `t`, `seq` from 1.7), `location` (canonical coordinates, 1.6), `outcome` (succeeded, failed, not applicable), `end_location`, `recipient`, and `restart` (open play, or the kind of restart: corner, free kick, throw-in, goal kick, kick-off, penalty).
- **Why / forced by (checked against corpus v0.3.0):** end location: P03, P07, P12, P27. Recipient: P12, P18, P31. Restart: P02, P06, P13, P32.
- **Source notes:** IDSSE has `FromOpenPlay` and `Recipient` on passes. Mapping per source is still to write.
- **Status:** agreed Kieran; team not yet confirmed.

### D-021: Derived event concepts are defined later, not stored (1.4)
- **Decision:** turnover (P02, P23, P24), touch count (P04) and under-pressure (P24) are not stored fields. They are named definitions built from basic events, like P01's `regain` macro. How definitions are written belongs to section 3 (syntax). What each one means is a decision to make there.
- **Why:** no single event records them, and IDSSE has no receipts, so sources would record them inconsistently.
- **Status:** agreed Kieran; team not yet confirmed. Parked for section 3.

### D-022: Missing event fields: "none" versus "unknown" (1.4)
- **Decision:** an event field is in one of three states. It has a value; it is **none** (does not apply, e.g. the recipient of a failed pass), so a condition on it is false; or it is **unknown** (the source should have it but does not, e.g. a missing location), so a condition on it is unknown.
- **Why:** a failed pass is recorded data, not missing data. In the IDSSE sample, completed passes carry a `Recipient` and unsuccessful ones do not. Treating that as unknown would wrongly turn real results into "possible".
- **Status:** agreed Kieran for now; team not yet confirmed. May change once the visible-area question (open) is settled.
