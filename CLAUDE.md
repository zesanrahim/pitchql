# PitchQL

Course project (Carleton University). Team: Zesan Rahim, Kieran Aston, Bashir Kadri.
Repo: https://github.com/zesanrahim/pitchql

Sub-project P3-a1 of the CARDS research program (El-Roby, "From Pixels to Decisions"): language design and formal semantics for a declarative query language that finds tactical patterns in soccer data.

## Goal
Deliver:
1. A written formal semantics (data model, type system, abstract syntax, semantics) for a language covering 90%+ of a corpus of 30-35 tactical patterns.
2. A naive reference interpreter over StatsBomb events and 360 freeze frames, plus 2-3 tracking predicates on Metrica data.
3. A 2-D overhead animation viewer with step-through controls and a visible-area window.
4. The "press-trap" example from the program document, fully worked and executed, with offline and streaming execution sketched by hand.

## Official P3-a1 spec (from `Sports_Analytics_Program.pdf`, Project 3, Stage 1 "start now")
- Steps: (1) requirements corpus of 30-50 canonical patterns from written sources; (2) grammar + type system over two data models (events; 25 Hz state); (3) formal event-time semantics; (4) paper-design the two compilation targets.
- Exit criteria: a specification document; 90%+ of the requirements corpus expressible; press-trap example fully worked through both execution models (offline and streaming) by hand.
- Language atoms: (i) event predicates over the nested event schema; (ii) spatial predicates over point sets (zone membership, set quantifiers like "3+ opponents within r of the carrier", relative geometry like "behind the last defensive line"); (iii) kinematic predicates over tracking (velocity thresholds, line-breaking runs, compactness collapsing) with interval semantics; (iv) sequencing with repetition, negation, and game-clock windows, scoped by possession.
- Semantics must cover: simultaneous timestamps, and interleaving of instantaneous events with interval-valued state.
- Our proposal uses 30-35 patterns, below the program's 30-50 range.

## Press-trap example
Sentence: "opponent builds up on the left, our press forces a back-pass, and within eight seconds we regain possession in their half with at least three players ahead of the ball."

Candidate syntax from the program document:
```
PATTERN left_press_trap ON match M:
  b : event(type = Pass, team = OPP, zone(b.loc) IN LeftBuildupThird);
  p : event(type = Pressure, team = US)+          -- one or more presses
  r : regain(team = US)                           -- macro: recovery / interception / tackle-won
  WHERE zone(r.loc) IN OppHalf
    AND gameclock(r) - gameclock(b) <= 8s
    AND count(players(US) WHERE x > ball.x AT r.t) >= 3
  RETURN clip(b.t - 2s, r.t + 5s);
```
Three ambiguities the language must make the author choose: game clock vs. wall clock for "within eight seconds"; "ahead of the ball" measured at the regain or at the next pass; "regain" as any touch or as established control.

## Related sub-projects
- P3-a2: compiler and optimizer (needs P3-a1 first). Offline plans on DuckDB/dataframes, then NFA-based causal streaming matcher.
- P3-b: LLM-assisted query authoring (it is the one that surfaces ambiguities to the user).
- P3-c: TactiBench, a 40-80 pattern corpus and benchmark. Label tier A = StatsBomb's free counterpress/play-pattern flags that queries must reproduce from raw primitives.
- Upstream: Tier-1 is the causal live tracking feed (no future frames); Tier-2 is revised, identity-resolved data delivered later.

## Working norms (apply to every project)
- Independence hygiene: never evaluate on reference data derived from your own input data.
- Latency honesty: state hardware, budget, and percentiles. (Not relevant to P3-a1 beyond design budgets.)
- Reproducibility: ship code and data where rights permit.
- Rights: StatsBomb open data requires attribution and non-commercial use.

## Out of scope
Compiler/optimizer (P3-a2), live streaming backend, LLM authoring interface (P3-b), TactiBench benchmark (P3-c).

## Key idea
StatsBomb 360 frames only show players inside a `visible_area` polygon. Each candidate match evaluates to true, false, or unknown. True = definite, unknown = possible, false = discarded. The analyst may also discard possibles.

## Semantics decisions to settle
- Simultaneous/duplicate timestamps and total order over events
- Whether patterns reset on possession change, and what counts as one
- Which clock time windows use
- Negation over intervals, and repetition
- Players outside the visible area

## Data
- StatsBomb open data: nested JSON events (~3,400 per match) plus 360 freeze frames
- 25 Hz tracking source, **undecided between two options** (samples downloaded to `data/`, which is git-ignored):
  - Metrica sample: 3 anonymised games ("Home"/"Away", no names or dates), positions only, CSV, small files. Pitch coordinates are normalised. No link to any StatsBomb match.
  - IDSSE (DFL, Bassek et al. 2025, CC-BY 4.0, https://github.com/spoho-datascience/idsse-data, data at https://doi.org/10.6084/m9.figshare.28196177): 7 named Bundesliga and 2. Bundesliga matches, full in-stadium TRACAB tracking (all players visible), with speed and acceleration per frame, plus synced official events. XML, about 400 MB of tracking per match. Pitch is 105x68 with the origin at the centre. Loadable with kloppy. Program document calls it the default independent ground truth. Sample downloaded: 1. FC Köln v FC Bayern München, 2022/23 matchday 34 (events and match info in full, first 20 MB of tracking).
- Coordinates differ across sources (StatsBomb 120x80, Metrica normalised, IDSSE 105x68 centred), and StatsBomb flips attacking direction between periods, so "left" must be defined relative to the attacking team's direction.
- Neither tracking source has StatsBomb labels. Only events-based patterns (about 11 of 44) can be scored against labels. Tracking patterns are checked by hand on sampled matches.
- Types: Event, Player, PlayerSet, Team, Role, Zone, Location, Instant, Interval, Possession, Frame

## Corpus rules
- Single versioned JSON file in the repo. Each entry: source id and retrieval date, verbatim sentence, our paraphrase, concept name, constructs exercised, data model needed, split (design or holdout).
- Chosen for construct diversity (negation, repetition, set quantifiers, nested windows), not representativeness.
- Design set comes from provider specs, then coaching writing.
- Holdout set is gathered only after the freeze date and evaluated once. Never look at it before the freeze.
- Freeze date = after which the semantics document and grammar do not change.

## Evaluation
- Precision/recall against StatsBomb's own labels (e.g. counterpress = pressure within 5 s of an open-play turnover), with each disagreement diagnosed.
- Compare against SQL:2016 MATCH_RECOGNIZE, FlinkCEP, hand-written pandas (spec length in lines and tokens, list inexpressible patterns).

## Timeline
Weeks 1-3 corpus and vocabulary; 4-6 formal semantics; by 9 interpreter; then evaluation and writeup. Semantics is drafted before the interpreter so the interpreter does not define the language by accident.

## Files in `docs/`
- `PitchQL__A_Formal_Semantics_...pdf`: our team's proposal (the source for this file)
- `Sports_Analytics_Program.pdf`: the CARDS program document (El-Roby). Defines P3-a1's official scope and exit criteria.
- `COMP5118_Final_Project___Proposal___Kevin_Ngeleka.pdf`: a different student's proposal for the same sub-project (theory only, focused on live vs. offline execution). Not our project.

## Working style
- Keep answers short and in plain words.
- Ask one question at a time.
