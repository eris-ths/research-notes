# Foundation A (Remesh / Sparse Solver) — Implementation Log 📈

> A **living document** tracking implementation progress of the remesh stack in eris-renderer's `scene::mesh`
> (Foundation A = sparse SPD solver and its precursors).
> To update, **append one block under "## Log" below** — no new files or issues per milestone. Newest on top.
>
> - Decision of record: [`discussion/2026-06-21-remesh-roadmap.md`](../discussion/2026-06-21-remesh-roadmap.md) (three-layer map A/A′/B + injectivity backbone, D1–D14)
> - Source of truth for implementation status: [eris-renderer STATUS.md §4.5](https://github.com/eris-ths/eris-renderer/blob/main/docs/reference/STATUS.md) (this log is a summary; details live there)
> - Survey: [MESH-EDITING-LANDSCAPE-SURVEY.md §13](https://github.com/eris-ths/eris-renderer/blob/main/docs/design/MESH-EDITING-LANDSCAPE-SURVEY.md) (near-term stack and sequencing)

## Entry template (copy to the top)

```
### YYYY-MM-DD — <title> (<layer: A precursor / A core / B / QEM ...>)
- **What**: <1–2 lines>
- **Source check**: <paper + equation numbers, or "—">
- **Verification**: <test count / live run / measurements>
- **Next**: <next step>
- **ref**: eris-renderer commit `<sha>` / STATUS §4.5
```

Screenshots: modeling-heavy work, so **wireframe** (`mode=preview:wire`, shows vertices/edges) and minimal
**AO** shading (`mode=preview:ao`) are enough — no beauty path-traced shots. Wireframe makes topology preservation
legible; before/after pairs show the geometric effect.

---

## Log

### 2026-06-21 — Taubin λ|μ smoothing (Foundation A first step / precursor)

| before (subdivision only) | after (+ Taubin, 25 steps) |
|---|---|
| ![before](../images/taubin-2026-06-21-before.png) | ![after](../images/taubin-2026-06-21-after.png) |

> Wireframe of a subdivided box (subdiv=1). The **topology is identical** in both shots — Taubin only moves
> vertices — yet the angular silhouette rounds out. Modeler `preview:wire` (the light plane behind is the modeler scene backdrop).
> Repro: launch `--ui` → `reset box` → `/mesh?subdiv=1` (before) / `/mesh?subdiv=1&taubin=25` (after) → `/frame?mode=preview:wire&scene=modeler`.

- **What**: Ship non-shrinking Laplacian smoothing **before** the sparse solver. Alternate a λ shrink step and a
  μ inflate step so shrinkage cancels while high-frequency noise is removed. Needs only the 1-ring (already in the
  half-edge mesh) — no solver required. A pure `&self → Mesh` modifier, exposed as `/mesh?taubin=N`. Boundary
  vertices are pinned, equal weights `w_ij = 1/|i*|`, Jacobi update for bit-exact determinism.
- **Source check**: Verified verbatim against Taubin, "Geometric Signal Processing on Polygonal Meshes"
  (EUROGRAPHICS 2000 STAR). Transfer function `f(k) = (1−λk)(1−μk)`, pass-band `k_PB = 1/λ + 1/μ`, recommended
  values **λ = 0.6307 / μ = −0.6732 / k_PB = 0.1**, boundary vertices fixed (§2). Coefficients taken from the
  paper, not from memory.
- **Verification**: 7 unit tests green (topology preserved: V/F/χ/manifold invariant; non-shrinking: shrinkage
  under half of plain Laplacian; roughness halved on a noised sphere; boundary fixed; identity at 0 passes;
  determinism; spec parsing). Full suite 286 green, zero build warnings. Live (torus): vertices move while
  topology is fully preserved (genus 1, valence-4 ×512 unchanged); running twice is bit-identical.
  - Misdiagnosis corrected mid-way: a bbox-shrink smoothness metric *grew* slightly on a subdivided cube → wrong
    metric → switched to a Laplacian-magnitude (roughness) metric.
- **Next**: Jacobi-PCG for the **sparse solver core** (implicit smoothing → LSCM UV → stage-1 convex cross field),
  with QEM decimation running in parallel.
- **ref**: eris-renderer commit `0e77a5d` / STATUS §4.5 (row M4.2 taubin)
