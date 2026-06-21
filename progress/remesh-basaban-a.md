# Foundation A (Remesh / Sparse Solver) — Implementation Log 📈

> A **living document** tracking implementation progress of the remesh stack in eris-renderer's `scene::mesh`
> (Foundation A = sparse SPD solver and its precursors).
> To update, **append one block under "## Log" below** — no new files or issues per milestone. Newest on top.
>
> - Decision of record: [`discussion/2026-06-21-remesh-roadmap.md`](../discussion/2026-06-21-remesh-roadmap.md) (three-layer map A/A′/B + injectivity backbone, D1–D14)
> - Source of truth for implementation status: [eris-renderer STATUS.md §4.5](https://github.com/eris-ths/eris-renderer/blob/main/docs/reference/STATUS.md) (this log is a summary; details live there)
> - Survey: [MESH-EDITING-LANDSCAPE-SURVEY.md §13](https://github.com/eris-ths/eris-renderer/blob/main/docs/design/MESH-EDITING-LANDSCAPE-SURVEY.md) (near-term stack and sequencing)

**Screenshots**: modeling-heavy work, so CAD-style wireframe (`mode=preview:wire-cad` — black background, primary-color
edges, emphasized vertex points, optional V/E/F/χ HUD) is the default. No beauty path-traced shots.

## Entry template

```
### YYYY-MM-DD · <title>
**Layer:** <A precursor / A core / B / QEM> — <one-line summary>

- 🔧 **What** — <1–2 lines>
- 📐 **Source** — <paper + equation numbers, or —>
- ✅ **Verify** — <tests / live run / measurements>
- ➡️ **Next** — <next step>
- 🔗 **Ref** — eris-renderer `<sha>` · STATUS §4.5
```

---

## Log

### 2026-06-21 · Taubin λ|μ smoothing
**Layer:** A (precursor) — non-shrinking Laplacian smoothing, shipped *before* the sparse solver.

| before (subdivision only) | after (+ Taubin, 25 steps) |
|---|---|
| ![before](../images/taubin-2026-06-21-before.png) | ![after](../images/taubin-2026-06-21-after.png) |

> CAD wireframe of a subdivided box (subdiv=1). The HUD (`V 26 E 48 F 24 / χ 2 MANIFOLD`) is **identical** in both
> shots — Taubin only moves vertices, so the topology is preserved — yet the angular silhouette rounds out.
> Repro: `--ui` → `reset box` → `/mesh?subdiv=1` (before) / `/mesh?subdiv=1&taubin=25` (after) → `/frame?mode=preview:wire-cad&scene=modeler`.

- 🔧 **What** — Ship non-shrinking Laplacian smoothing *before* the sparse solver. Alternate a λ shrink step and a
  μ inflate step so shrinkage cancels while high-frequency noise is removed. Needs only the 1-ring (already in the
  half-edge mesh) — no solver. A pure `&self → Mesh` modifier, `/mesh?taubin=N`. Boundary vertices pinned, equal
  weights `w_ij = 1/|i*|`, Jacobi update for bit-exact determinism.
- 📐 **Source** — Verified verbatim against Taubin, *Geometric Signal Processing on Polygonal Meshes* (EUROGRAPHICS
  2000 STAR). Transfer `f(k) = (1−λk)(1−μk)`, pass-band `k_PB = 1/λ + 1/μ`, recommended **λ = 0.6307 / μ = −0.6732 /
  k_PB = 0.1**, boundary fixed (§2). Coefficients from the paper, not memory.
- ✅ **Verify** — 7 unit tests green (topology preserved; non-shrinking: under half the shrinkage of plain
  Laplacian; roughness halved on a noised sphere; boundary fixed; identity at 0 passes; determinism; spec parse).
  Full suite 286 green, zero warnings. Live (torus): vertices move, topology fully preserved (genus 1, valence-4
  ×512), bit-identical across runs. *(Misdiagnosis corrected: a bbox-shrink metric grew on a subdivided cube →
  switched to a Laplacian-magnitude roughness metric.)*
- ➡️ **Next** — Jacobi-PCG for the **sparse solver core** (implicit smoothing → LSCM UV → stage-1 convex cross
  field), with QEM decimation in parallel.
- 🔗 **Ref** — eris-renderer `0e77a5d` · STATUS §4.5 (row M4.2 taubin)
