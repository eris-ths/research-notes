# discussion — Decisions of Record 💬

> A shelf for **meeting minutes**: multi-party (human + Claude/Eris) discussion, agreement, and decisions.
> A different genre from `results/` (research conclusions): this is where we keep *what we decided / talked through*.

## How it splits from other shelves

| Shelf | What goes here |
|---|---|
| `results/` | conclusions of research / comparison / analysis (*what we investigated*) |
| `discussion/` | meeting minutes / agreements / **decisions** (*what we decided / talked through*) |

## Naming

`YYYY-MM-DD-<topic>.md` — minutes are indexed primarily by date, so the date leads for chronological sorting.

## Ongoing flow (how to record minutes here)

1. Place the canonical minutes (Markdown) at `discussion/YYYY-MM-DD-<topic>.md`. If it originates in another
   repo, add a **provenance note** at the top and rewrite cross-repo links to absolute URLs
   (`github.com/eris-ths/<repo>/blob/main/...`).
2. Add one row to the index below.
3. Commit & push the branch.
4. Open a GitHub issue as the *entry point* linking to this file (don't paste the full text into the issue — an
   executive summary plus a link is enough; the file is the canonical record — records outlive writers).

## Index

| Date | Topic | Origin |
|---|---|---|
| 2026-06-21 | [Remesh roadmap discussion (consolidated)](2026-06-21-remesh-roadmap.md) | minutes from eris-renderer `scene::mesh` |

> Note: the 2026-06-21 minutes body is in Japanese (a historical snapshot). New posts going forward are in English.
