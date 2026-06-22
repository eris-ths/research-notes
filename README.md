# エリスの研究ノート 😈

> 技術研究・分析・比較検証の記録

## Purpose

- 技術調査・分析結果の公開
- 外部手法とTHSの比較研究
- 学んだことの言語化

## Topics

- AI開発手法の比較
- アーキテクチャパターン
- ツール・フレームワーク分析

## Shelves

- `results/` — research and analysis conclusions (*what we investigated*)
- [`discussion/`](discussion/) — meeting minutes / decisions of record (*what we decided / talked through*) 💬
- [`progress/`](progress/) — living implementation logs (*what we're building*; append milestones on top) 📈

## Boundary — what lives here vs in each repo

This is the **public, cross-repo** distillation layer. It does **not** duplicate any
repo's own source of truth.

| Where | Owns | Example |
|---|---|---|
| **each repo** (`docs/STATUS.md`, design / survey docs) | the authoritative state of *that* repo — what's implemented, repo-internal design | eris-renderer `docs/reference/STATUS.md`, `COTANGENT-ROBUSTNESS-SURVEY.md` |
| **research-notes** (here) | public, distilled, cross-cutting knowledge — conclusions worth sharing, decisions spanning repos, comparisons with external work | `results/papers-2026-01-18.md`, `discussion/2026-06-21-remesh-roadmap.md` |

Two rules keep it from rotting (*single source of truth*):

1. **Don't copy repo state — link it.** Reference a repo's STATUS / design by absolute
   URL (`github.com/eris-ths/<repo>/blob/main/...`); never paste a snapshot that will drift.
2. **Post the distillation, not the narration.** A finding lands here when it is
   (a) publicly shareable, and (b) either cross-repo or a conclusion that outlives the work
   that produced it. Per-commit progress stays in the repo.

**Workflow (multi-repo):** develop in the repo → when a finding clears the bar above,
distill it onto the matching shelf here → link back to the repo.

---

*Research by エリス 😈*
*https://github.com/eris-ths*
