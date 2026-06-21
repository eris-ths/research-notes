# 基盤 A（リメッシュ / 疎ソルバ）実装ログ 📈

> eris-renderer の `scene::mesh` リメッシュ系（基盤 A = 疎 SPD ソルバ + その前哨）の実装進捗を時系列で残す **living doc**。
> 更新は **このファイルの「## 進捗ログ」直下に1ブロック追記するだけ**（新ファイル・新 issue を都度作らない）。新しい節目ほど上。
>
> - 議論の正本: [`discussion/2026-06-21-remesh-roadmap.md`](../discussion/2026-06-21-remesh-roadmap.md)（三層地図 A/A′/B + 背骨 injectivity、D1-D14）
> - 実装状況の権威: [eris-renderer STATUS.md §4.5](https://github.com/eris-ths/eris-renderer/blob/main/docs/reference/STATUS.md)（このログは要約。詳細はあちら）
> - サーベイ: [MESH-EDITING-LANDSCAPE-SURVEY.md §13](https://github.com/eris-ths/eris-renderer/blob/main/docs/design/MESH-EDITING-LANDSCAPE-SURVEY.md)（near-term スタックと段取り）

## 追記テンプレ（コピペして先頭に貼る）

```
### YYYY-MM-DD — <タイトル>（<どの層: A 前哨 / A 本体 / B / QEM 等>）
- **何を**: <1-2行>
- **原典照合**: <論文・式番号。なければ「—」>
- **検証**: <テスト数・実機・計測>
- **次**: <次の一手>
- **ref**: eris-renderer commit `<sha>` / STATUS §4.5
```

---

## 進捗ログ

### 2026-06-21 — Taubin λ|μ smoothing（基盤 A 着工の第一歩 / 前哨）
- **何を**: 非収縮ラプラシアン平滑を **疎ソルバの「前」に**出荷。λ 縮ステップ + μ 膨ステップを交互に打ち、収縮を相殺しつつ平滑。1-ring だけで済む（half-edge に既存）= ソルバ不要。`&self → Mesh` 純粋モディファイア、`/mesh?taubin=N`。境界頂点は固定、等重み `w_ij=1/|i*|`、Jacobi 更新で bit 決定論。
- **原典照合**: Taubin "Geometric Signal Processing on Polygonal Meshes"（EUROGRAPHICS 2000 STAR）を逐語照合。transfer `f(k)=(1−λk)(1−μk)`、pass-band `k_PB=1/λ+1/μ`、推奨値 **λ=0.6307 / μ=−0.6732 / k_PB=0.1**、境界固定（§2）。係数は記憶でなく原典から。
- **検証**: ユニットテスト 7 緑（位相保存 V/F/χ/manifold 不変・非収縮[素朴ラプラシアンの収縮の半分未満]・粗さ半減[ノイズ球]・境界不動・恒等・決定論・spec パース）。全 286 テスト緑・ビルド警告ゼロ。実機（torus）で頂点が動き位相完全保存（genus 1・valence-4×512 不変）、2 回叩いて bit 一致。
  - 途中の誤診訂正: 当初 bbox 縮小を平滑指標にしたら細分立方体で逆に微増 → 不適と判明 → ラプラシアン振幅指標へ差し替え。
- **次**: Jacobi-PCG で **疎ソルバ A 本体**（implicit smoothing → LSCM UV → 段1 凸 cross field）。並走で QEM Decimation。
- **ref**: eris-renderer commit `0e77a5d` / STATUS §4.5（M4.2 taubin 行）
