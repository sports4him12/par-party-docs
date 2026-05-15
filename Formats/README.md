# Formats

Design docs for golf tournament formats that GolfSync supports or is
considering. Each doc captures: what the format actually is (with
quotes from the organizer's published rules where available), how it
maps to the existing data model, the gaps that would need new
engineering, an implementation-options analysis with a recommendation,
a migration sketch, scope-of-work estimate, and open questions for
the organizer.

Use these when planning a new format build or when an organizer asks
"can your app run X format?" — the doc tells you (a) yes/no, (b) what
it takes to make it yes, and (c) what's left ambiguous in the
published rules.

## Index

- [BSG Sixes](bsg-sixes.md) — 4 players, 3 rotating pairs across 6-hole
  blocks, per-hole 3/1/0 points. From the BSG Summer Series 2026.
- [Three-Headed Monster](three-headed-monster.md) — 2-man team game,
  three different formats (Scramble / Alt Shot / Greensome) across 6-hole
  phases. BSG Major #3 (July 18, 2026).
- [Wolf](wolf.md) — 4-player rotating-Wolf game with PICK/PASS/Lone Wolf
  per hole. BSG Majors #1 + #4 at Sycamore Creek (May 2 + Aug 15, 2026).

## What's missing across all three

The BSG Majors series counts **"3 best of 5 events"** for the season
standings. The 5 Majors are in different formats — Wolf, BSG 4-3-2,
Three-Headed Monster, Wolf again, Finale. Their per-event scores are
in different units (Wolf points, BSG net team total, Sixes points,
etc.) and don't sum cleanly. **A cross-format normalization rule is a
prerequisite** for the season aggregation, and that decision isn't in
any of the format-specific docs above — it's a shared design
conversation worth having before the season finale (Oct 3 at
Viniterra) but after the first event ships.

See each doc's "Cross-references" section for the existing engine
files + sibling formats they relate to.
