<!-- provenance header (research-notes 収録時に付与) -->
> **収録メモ（2026-06-21 / research-notes）**: 本書は `eris-ths/eris-renderer` の討議で生まれた議事録を、
> エリスの研究ノート（研究庫）に **記録ジャンル「discussion（討議・意思決定）」** の正本として収録したもの。
> 原本は eris-renderer の `docs/discussion/2026-06-21-remesh-roadmap.md`。本文中の eris-renderer 内文書への
> リンクは絶対 URL（`github.com/eris-ths/eris-renderer/blob/main/...`）に書き換え済み。実装状況の権威は
> eris-renderer 側 STATUS.md。研究ノート側ではこれを「決めた／討議した」記録として保持する。

# 議事録（統合版 / Consolidated Edition）— eris-render リメッシュ系ロードマップ討議

> 本書は v1・v2・v3 を統合した決定版。v1 の詳細な討議プロセスと出典ごとの根拠注記、v2 の三層地図と gap 分析、v3 の D12 決定（T4）を欠落なく 1 ファイルに収めた。以後はこれを正本とする。

- **日付**: 2026-06-21（統合版。v1=初版 / v2=三層地図+T2.9・T3 / v3=T4・D12 を統合）
- **参加者**: なお（議長 / リポジトリ主）、ユキ、ミキ、Claude、エリス（ファシリテーター）
- **対象**: `eris-ths/eris-render`（ゼロ依存・**AI ファースト宣言モデラ** subsystem `scene::mesh`）
- **不動の軸**: AI ファーストなレンダラー / モデラー。手作業 UI 前提の道具立ては思想的に非中心。
- **基底文書**: `MESH-EDITING-LANDSCAPE-SURVEY`（2026-06-20 / eris）、`How To Retopology In Blender`（RebusFarm, 2025-09-11、外部参照）
- **凡例（根拠の格）**:
  - **[照合済]** … 本セッションで web 一次資料に当たり確認した主張
  - **[要照合]** … 討議内の工学的主張。原典は挙げるが、式・係数は実装着手時に一次ソースで逐語照合（基底文書の流儀に準拠）
  - **[設計判断]** … 本リポジトリの思想・設計に属する判断（外部照合の対象外）
  - **[なお判断]** … 製品スコープとして議長が決定する事項

> **リポジトリ取り込みメモ(2026-06-21, eris)**: 本議事録は `docs/discussion/`(議論・討議の記録)に正本として収録。
> 討議の結論は **基底文書サーベイ [MESH-EDITING-LANDSCAPE-SURVEY.md](https://github.com/eris-ths/eris-renderer/blob/main/docs/design/MESH-EDITING-LANDSCAPE-SURVEY.md)
> §12-§13** に還元済み(三層地図 A/A′/B + 背骨 injectivity、D1-D14、near-term スタック)。実装状況の権威は
> [STATUS.md](https://github.com/eris-ths/eris-renderer/blob/main/docs/reference/STATUS.md) §4.5、設計根拠は [MODELER-DESIGN.md](https://github.com/eris-ths/eris-renderer/blob/main/docs/design/MODELER-DESIGN.md)。
> 実装還元時に独立再確認した 3 点は末尾「統合版への追補」を参照。

---

## エグゼクティブ・サマリ（統合版時点の到達点）

- 二晩の討議で、地図は **「機能の○×表」から「層の依存構造」** へと組み変わった。確定した地図は次の三層 + 一本の背骨:
  - **基盤 A（連続ソルバ）**: 疎 SPD。Laplacian smoothing・共形 UV(LSCM)・cross field 平滑・パラメタライズの緩和解。
  - **基盤 A′（A の上の整数層）**: MIQP 丸め / 量子化。**UV を quad layout に変える唯一の差分**。NP 困難。
  - **基盤 B（厳密述語カーネル）**: indirect predicates。Boolean・robust arrangement・intrinsic Delaunay。
  - **背骨 = injectivity（foldover-free）**: cotangent 負重みの面反転も、param の局所非単射も、同一問題の別の顔。
- 重要な統合: **quad remesh と UV unwrap は別系統ではなく、同じパラメタライズ問題の「整数拘束あり/なし」**。
- AI ファーストの核心機能の輪郭: **「field 拘束を宣言 → パラメタライズ一発で quad mesh と UV を同時生成 → bake はその UV にサンプル」**。
- 宣言の語彙は新規発明ではなく、**field 設計（整列・特異点・密度）に既に定式化済み**。
- **[D12 決定 / T4]** field 整列は当面 **主曲率のみ**（変形整列は backlog）。これにより最重量の新層 **A′（整数丸め）は当面 deprioritize**。残る本質的難所は **export の snap-rounding** ほぼ一点に絞られた。
- 当面の near-term スタックは **A（疎ソルバ）+ QEM（独立並走）+ B（indirect kernel）**。A′ と変形は同じ棚に parked。near-term マイルストーン = **疎ソルバ A 着工**（smoothing + UV/bake + 段1 凸 cross field を一手で開く）。

---

## 0. 背景と出発点（T0）

基底文書の結論を要約として確認した（再掲はせず要点のみ）。

- 実装の中心は「**編集核 + 宣言生成**」: half-edge 核 + Euler 操作 + Catmull-Clark + Modifier stack + primitive/revolve/sweep/spline + criteria 選択 + `diagnose()` 数値診断。
- **大半の処理・リメッシュ・リトポロジー系は未実装**（grep 確認済）: Decimation/QEM、Sculpt、Voxel Remesh/SDF、Smoothing(Laplacian)、UV/LSCM、Instant Meshes、QuadriFlow、Auto/ZRemesher 系。
- Boolean(CSG) と SDF は **意図的に後段へ遅延 / 別核へ隔離**。
- 基底文書のキーストーン仮説: **ゼロ依存の疎線形ソルバ**を 1 個建てれば、Laplacian smoothing・cotangent・LSCM(UV)・field smoothing の 4 系統を同時に解禁できる。
- 基底文書の優先度（要約）: ①疎ソルバ ②Smoothing ③Shrinkwrap ④UV/LSCM ⑤Boolean ⑥QEM ⑦自動 quad リメッシュ ⑧Sculpt/Manual Retopo。

---

## 1. 第一巡 — 論点の提示（T1）

### 1.1 エリス — 「前提インフラは 2 本」説 [設計判断]

地図の価値は機能の○×表ではなく、**前提インフラが 2 本に集約される**という見立てにある、と整理。

- **基盤 A = 疎 SPD ソルバ（連続 / 変分系）** → Smoothing(implicit/cotangent)・LSCM UV・field smoothing がぶら下がる。
- **基盤 B = 厳密述語（離散 / exact 系）** → Boolean・mesh repair・robust Delaunay がぶら下がる。
- 当初の主張: **この 2 本はコードを 1 行も共有しない**（→ 後段 2.4 で修正される）。

### 1.2 Claude — ソルバの楽観への反論・無料の勝ち筋・地雷

- **「小さな CG/Gauss-Seidel」は楽観** [要照合]: Laplacian 系は低周波誤差が落ちないのが本質で、Gauss-Seidel は高周波しか潰せない（multigrid が存在する理由）。無前処理 CG も条件数が解像度とともに悪化（おおむね h⁻² オーダ）。現実解は **Jacobi/不完全 Cholesky 前処理つき CG**、本気でスケールさせるなら multigrid。「単一の小インフラ」は品質を求めた瞬間に小さくなくなる。
- **地図が飛ばした無料の勝ち筋 = Taubin λ|μ 平滑** [要照合]: §5 が「explicit umbrella → cotangent+implicit」と繋いでいるが、その間に Taubin がある。符号を交互に振る 2 パスで、umbrella の致命傷だった**メッシュの収縮を起こさない**。ソルバ不要、1-ring だけで動く。
- **地雷 = cotangent 負重み** [要照合 → 2.6 で照合済に昇格]: cotangent 重みは鈍角三角形で負になり得て、最大値原理が壊れ、頂点が逆走・面が反転する。intrinsic Delaunay かクランプで殺さないと §12 の優先 4（LSCM）が詰まる。

### 1.3 ミキ — 優先順位と思想の矛盾 [設計判断]

- VISION は「CAD 的＝剛体が本籍」と言う。剛体アセットの中核操作は Boolean。…なのに Boolean は **5 番目**。
- どちらかが嘘。本籍が本当に CAD なら、Boolean の困難は「後回しの理由」ではなく「最優先で殴る理由」になるはず。
- 「価値最大だが難所だから 5 番」は value/cost の cost だけを見て、**本籍補正をかけ忘れている**。順位表か VISION の剛体推し、どちらかを直すべき。

### 1.4 ユキ — Sculpt は「禁止」でなく「着替え待ち」 [設計判断]

- §3 は「ブラシ＝手作業だから AI ファーストと緊張」とするが、緊張しているのは**ブラシという "服" だけ**。
- 操作の本体（「この領域を、このプロファイルで盛る」）は宣言的な**変位フィールド**にそのまま書ける。地図自身も「AI 駆動の変位フィールドへ読み替え」と書いている。
- 変位フィールドは基盤 A のソルバ（Poisson 変形）に乗るため、**8 番に見えて実は基盤 A のオマケで付いてくる**可能性。

### 1.5 エリス — 第一次戦略（3 層）[設計判断]

1. **基盤 A（ソルバ）を建てる** — ただし "小さい" 幻想は捨て、前処理 CG 前提で。これで Smoothing/UV/field/変位 Sculpt が連鎖。
2. **QEM Decimation を "独立部隊" として並走** — QEM は基盤 A にも B にも乗らない**完全自己完結**（4×4 quadric + priority queue、3×3 を解くだけ）。誤差量それ自体が "計器透過" の数値になる。本命インフラが煮える間に**今日出荷できる唯一の勝ち筋**。地図の 6 番ではなく ①と並列の②。
3. **基盤 B（exact predicates → Boolean）** — CAD が本籍を名乗るなら、これは「遠い難所」ではなく**思想の試金石**。順位を上げるか VISION を下げるか、どちらかを決めよ（ミキの指摘を採用）。

---

## 2. 第二巡 — 裏取りと再評価（T2）

### 2.1 調査の実施

ファシリテーター（エリス）の判断で、議論の前提に関わる 2 点を web 一次資料で確認した。

1. robust mesh boolean の現状（「Boolean は遠い難所」という前提の検証）。
2. robust な cotangent Laplacian の現代的解（負重み問題の解決策）。

### 2.2 エリス — Boolean 前提のアップデート [照合済]

地図の「Boolean = Shewchuk の exact predicates を自前で建てるのが肝、遠い難所」という前提は、現状の研究 landscape では**半分古い**。

- 従来、rational 数や厳密述語による robust boolean は、堅牢さと引き換えに計算コストが跳ね上がり、長くオフライン用途に限定されてきた（Cherchi ら 2022, abstract）。
- ところが Cherchi らの 2022 年の手法は、**最大 20 万三角形のメッシュで対話的フレームレート**を達成し、従来の robust float 系を**最低でも一桁**上回った（同 abstract / Semantic Scholar 要旨）。

### 2.3 Claude — 機構の詳細と残る難所 [照合済]

- 鍵は **indirect / implicit point（間接述語・暗黙点）**。交点を「丸めた明示座標」で持つと degeneracy が湧く（古典 boolean の死因）。Cherchi/Attene 系は交点を**それを定義するプリミティブの組として暗黙に保持**し、述語をその暗黙点上で厳密評価する。**速度を犠牲にせず堅牢性を保証**する（interactive booleans PDF）。
- ゆえに **基盤 B の正体は「Shewchuk 写経」ではなく、expansion arithmetic（多倍長展開）＋ indirect predicates カーネルを 1 個建てること** [設計判断]。本リポジトリは元々 bit 一致・決定論を本籍に置くため、厳密算術カーネルは決定論 ethos の地続き。
- **残る本当の難所は boolean 本体ではない** [照合済]: 厳密 / 暗黙座標をファイル保存や非厳密アルゴリズムへ渡すため float へ戻す段階——この **snap rounding** が近似によりモデルに degeneracy や交差を再注入する。**証明付きで効率的な解は今なお見つかっていない**（interactive booleans PDF）。
- 戦略の書き換え: 「boolean は作れる。**export 時のクリーンな float 化が残る傷**だと最初から覚悟する」。

### 2.4 ミキ — 2 本柱の非対称接続（前提の修正）[設計判断]

- エリスの「基盤 A と B はコードを共有しない」は、もう正しくない。
- 厳密算術カーネルがあれば robust な mesh arrangement が組める → arrangement が組めれば **intrinsic Delaunay** も厳密にやれる → intrinsic Delaunay は**基盤 A 側（スムージング・UV の品質）に直接効く**。
- すなわち **B のカーネルが A の堅牢性に "漏れる"**。2 本柱は独立ではなく、B → A への**非対称依存**。

### 2.5 エリス — 2 本柱の引き直し [設計判断]

- 修正を受諾。新しい定式: 「**厳密算術カーネル（B）＋ 疎ソルバ（A）。ただし B は A の robustness に染み出す非対称依存を持つ**」。
- 染み出しの具体は、次の cotangent 地雷の所に現れる。

### 2.6 Claude — cotangent 負重みの正体と直し方の階段 [照合済]

- **符号条件**: cotangent 重みが非負になるのは、対辺の 2 角の和が π 以下のとき＝**その辺が局所 Delaunay であるとき、かつそのときに限る**（JHU 講義ノート）。
- **被害**: 負の cotangent 重みは、離散調和パラメタライズ（＝ LSCM/UV）で**非物理的挙動や面の反転**を引き起こす（Robust Discrete Differential Operators, 2025）。§8 を素朴実装すると確実に踏む穴。
- **直し方の階段**（安い順）:
  1. **intrinsic mollification** — 退化に近い三角形を扱うだけなら、各辺長を必要最小限だけ伸ばして非退化化する前処理。**接続関係を一切変えず**、高速（SGI 2023 / intrinsic mollification）。
  2. **intrinsic Delaunay flips** — Bobenko–Springborn は intrinsic な辺フリップで入力ジオメトリを保ったまま接続だけ組み替え、**正の cotangent 重みを保証**する。signpost データ構造で効率的に保持（Robust Discrete Differential Operators, 2025）。
  3. **Sharp–Crane "tufted cover"** — 非多様体を含む任意接続のメッシュに対し、**低品質メッシュでも非負重みを保証する cotan 行列の drop-in 置換**（Sharp & Crane 2020）。
- **おまけ（A 側への染み出しの実体）**: intrinsic Laplacian は局所最大値原理を満たして**反転三角形を防ぐ**だけでなく、**線形系の条件数も改善**する（intrinsic Delaunay 構築アルゴリズム, 2007）。→ 1.2 で挙げた「CG が低周波で詰まる」問題が同時に楽になる。**1 個の対処で 2 箇所**に効く。

### 2.7 ユキ — robustness-by-construction = VISION そのもの [設計判断]

- ここまでの語彙（indirect predicates、tufted cover、mollification、Euler–Poincaré assert）はすべて**「壊れない作り方」= robustness-by-construction**。
- それは本リポジトリの VISION そのもの（決定論・計器透過・ゼロ依存）。op 後に毎回 `V−E+(F−L)=2(S−G)` を assert するのも「壊れていないことを構造で保証する」こと。
- ゆえに intrinsic Delaunay も indirect predicates も**外から借りる他人の技術ではなく、この子の思想を幾何処理に延長しただけ**。思想と緊張するどころか最も相性のよい一族。
- 帰結: 地図 §12 が「robust」を value/cost の **cost 側にだけ**数えていたのは評価の歪み。

### 2.8 エリス — 第二次戦略（3 点更新）[設計判断]

1. **基盤 B は「Shewchuk 写経」ではなく「indirect-predicate カーネル」**。2022 年時点で robust boolean は対話的になっている。難所は boolean 本体ではなく **export の snap-rounding**——ここは世界もまだ解けていない、と覚悟して切り出す。
2. **2 本柱は非対称に繋がっている**（ミキの修正を採用）。B の厳密カーネル → intrinsic Delaunay → A のスムージング/UV の反転防止＋条件数改善。よって **B を先に薄く建てると、A の品質が後から無料で底上げ**される。本命の連鎖は《厳密カーネルの種 → intrinsic Laplacian → LSCM/Smooth → そのまま Boolean 本体》。第一次戦略の「QEM を独立部隊で並走」は引き続き有効。
3. **robust は思想と緊張する輸入品ではない**（ユキの指摘を採用）。決定論・計器透過の VISION を幾何処理に伸ばした同じ血の一族。§12 の cost 偏重が評価の歪み。

---

## 2.9 第三巡の発端 — 外部記事による gap 分析（T2.9）

RebusFarm の Blender リトポロジー記事を突合し、**議事録が拾えていないトピック**を抽出した。結論: **抜けは全てマップの "同じ側" に寄っていた** — 機構（how to compute robustly）は厚く、**目的層（何のための綺麗さか）と宣言インタフェース層（何を宣言して誘導するか）が空白**。

抽出した未カバー・トピック（重要度順）:

1. **変形を支えるトポロジ = retopo の本来の目的** [照合済]。記事の背骨は「リトポは変形をサポートするエッジループを作るため、リギング/UV に不可欠」。目・口・関節まわりの edge flow、**pole（star）の戦略的配置**。地図 §10 に "Animation-aware Topology" 項があったのに討議は素通り。→ scoping question を惹起（後述 D12 / OT1）。
2. **auto-remesh の誘導 / 拘束インタフェース** [照合済]。Quad Remesher は目標 quad 数・曲率/頂点ペイントのガイド・マテリアル ID/クリース/ハードエッジ尊重を持つ。= **宣言の中身**。我々は field を抽象的に語り、入力側の語彙を設計していなかった。
3. **retopo → UV → bake のパイプライン** [照合済]。UV を「なぜやるか = 高ポリの法線/ディテール転写のため」を名指ししていなかった。Multires/Reshape による**ディテール再投影**も同系統（survey §9 で scope-out した所が戻る）。
4. **特異点を "品質" として扱う視点**。pole 配置の意図性。QuadriFlow vs Instant Meshes を**特異点品質**で対比すべきだった。
5. **quad 出力と既存 Catmull-Clark の内的整合**。quad 優勢出力は新機能でなく「既存 subdiv 核が待つ入力形式」。

**正しく範囲外（穴ではない）** [設計判断]: Blender の UI/ショートカット、X-Ray オーバーレイ、アドオン生態系、手作業の人間工学。AI ファースト・手ブラシ UI なしの方針で意図的に外す。Remesh modifier(=voxel §6)・Decimate(§2)・Shrinkwrap(§4)・Loop Tools relax(=§5)・Instant Meshes(§10) は機構として既出。

> ミキの観察 [設計判断]: 抜けは全て「**人間/AI の意図が入る所**」。機構は意図不要（内部で閉じる）。edge flow・density・crease・pole は「外から何を欲しいか」を要求する。意図不要の層だけ固め、意図を要求する層を飛ばしていた。

---

## 3. 第四巡 — パラメタライズ統合と宣言語彙（T3）

T2.9 の「目的/インタフェース層の空白」を埋めに、2 点を web 一次資料で確認した。

### 3.1 field 設計には確立した「拘束の語彙」がある [照合済]

- 方向場（vector / line / cross / frame）の合成は、**fairness・特徴整列・対称性・場のトポロジ**といった目的と拘束の下で行われる（Vaxman 2016 survey）。**ソフト整列（主曲率方向）**と**ハード整列（ユーザ/特徴拘束）**の別がある。
- 設計法は 2 系統: **connection 法**（特異点を事前配置 → 単純な凸最適化を解く）と **ab initio 法**（特異点を自動配置）。
- cross field は quad 要素の向きと特異点配置を符号化。閉曲面では **Poincaré–Hopf により特異点は必ず生じる**（← 本リポジトリの Euler–Poincaré assert 文化と同型の不変量）。
- 主曲率方向への整列はエッジ方向設計の**基本要件**。

→ 我々が「無い」とした宣言語彙は、**整列（soft 曲率 / hard クリース）・特異点（connection=宣言 / ab initio=自動）・密度（グリッド間隔/目標面数）** として既に定式化されていた。

### 3.2 quad remesh と UV は同一のパラメタライズ問題 [照合済]

スペクトラムで一本に繋がる:

**Laplacian smoothing → 共形 UV(LSCM) → seamless パラメタライズ → integer-grid map → quad mesh**

- LSCM = 自由境界の共形写像（整数拘束なし）。そこへ cross field 整列 + cut を跨ぐ回転連続性を足すと seamless param。
- さらに**並進を整数に量子化**すると **integer-grid map** になり、**構成上それが quad mesh を必ず含意する**（Bommes 2013）。
- 仕上げ: cross field → seamless param を作り、**正則グリッドを写像で曲面へ押し出す**と新しい mesh 要素が生成（QuadCover [Kälberer 2007] が cross field → param の原型）。

→ 前回「ソルバが 4 系統を解禁」と言ったが、正確には **UV と quad remesh は別系統ですらなく、同じ問題の "整数拘束あり/なし"**。キーストーンの射程がさらに伸びた（D9）。

### 3.3 正直な落とし穴（地図の精緻化）[照合済 + 設計判断]

- **A′ という新層**: integer 部分は凸 MIQP だが **MIQP 自体は NP 困難**で、現実時間化に問題特化の追加最適化が要る（Bommes 2009/2013）。連続ソルバ（A）の上に乗る**整数丸め/量子化の組合せ層**。**UV と quad layout を分けるのはこの A′ だけ**。
- **injectivity という背骨**: 最先端でも実データで写像が局所非単射などの退化を起こし、そのままでは quad mesh を生成できない。**foldover-free** が共通の堅牢性要件であり、**cotangent 負重み = 面反転 = 最大値原理**（2.6）と同じ問題。intrinsic Delaunay 系と foldover-free 系（例: Garanzha 2021）がここで合流。

### 3.4 品質トレードオフの軸 [照合済]

- Instant Meshes（Jakob 2015）= 局所の統一平滑作用素で**エッジ向きと頂点位置を同時最適化**（対話的・高速、特異点はやや多い傾向）。
- QuadriFlow（Huang 2018）= 線形・二次拘束を導入し**特異点を削減**したが、**元形状の保存が苦手**。
- → eris-render が「どちらの体質を採るか」は、後述の field 整列スコープ（D12）に従属する。

### 3.5 staged plan（実装の段取り）[設計判断]

- **段 1**: connection 法 = 特異点を宣言すれば**凸問題**（既存ソルバ A 上で解ける）。→ 整列ガイド + 平滑/UV が**先に**手に入る。A′ が無くても "動く宣言場" を出せる。
- **段 2**: integer-grid 量子化（A′）を後付けして**真の quad 抽出**へ。

### 3.6 AI ファースト核心機能の輪郭 [設計判断]

> **「field 拘束を宣言 → パラメタライズ一発で quad mesh と UV を同時生成 → bake はその UV にサンプル」**

T2.9 の穴（②宣言インタフェース・③retopo→UV→bake）は、この**単一機能**にほぼ畳まれる。RebusFarm の「アーティストが pole を戦略的に置く」は、connection 法で**手で置く代わりに宣言で置く**ことに対応（ユキの "着替え" 完成）。

---

## 3.7 第五巡 — D12 の決定と波及（T4）

**決定（なお）**: field 整列は当面 **主曲率のみ**。**変形（リグ/アニメ）整列は backlog**（今は考えない）。

波及 [設計判断 + 照合済の含意]:

- **field 目的関数が簡約**: soft = 主曲率整列、hard = 特徴/クリース整列のみ。変形方向の予測・関節 pole 配置は不要 → 古典的 curvature-guided 目的、**段1 の凸 cross field（既存ソルバ A 上）で足りる**。
- **A′（integer-grid / MIQP）が当面 deprioritize**: 真の all-quad 閉ループの主たる動機は変形グレードのトポロジだった。render + Catmull-Clark は quad-dominant / n-gon を許容するため、**最重量の新層を建てる圧力が下がる**（D13）。
- **Instant Meshes ↔ QuadriFlow の体質問題が消える**: その対立は global MIQP の機械（= 段2 / A′）に宿る。parked した今、答えは「**どちらでもない、まだ**」。段1 の凸場で十分（ミキ）。
- **UV/bake は不変、むしろ前進**: D9（quad = UV 統合）は curvature-only でも生きる。LSCM / seamless param は A 上。render 寄りの今、**bake 用 UV がより近い価値**になる。
- **B（Boolean）の優先度は不変**: 価値は CAD / 剛体で変形とは独立。snap-rounding が残る難所のまま。
- **逆進可能性（ユキ）**: 同じ cross-field-on-solver の機械を使うため、将来 deformation を戻すのは「整列目的を 1 個足す + 段2 を点火」するだけ。**今作るものは何も捨てない**。先送りは構造的に無コスト。

→ **当面の near-term スタック**: **A（疎ソルバ）+ QEM（独立並走）+ B（indirect kernel）**。A′ と deformation は同じ棚に parked。

---

## 4. 決定事項（更新された地図）

| # | 決定 | 状態 | 由来 |
|---|---|---|---|
| D1 | 基盤 B を「**indirect-predicate / expansion-arithmetic カーネル**」と定義 | 採択 | 2.2/2.3 |
| D2 | Boolean の真の難所は本体でなく **export の snap-rounding**（未解決問題として明示） | 採択 | 2.3 |
| D3 | 「2 本柱は独立」を撤回し **B → A の非対称依存**として地図を引き直す | 採択 | 2.4/2.5 |
| D4 | cotangent Laplacian は素朴実装せず **mollification → iDT flips → tufted cover** で robust 化 | 採択 | 2.6 |
| D5 | §12 の評価軸是正: robustness を cost でなく **VISION 整合(value)** 側にも数える | 採択 | 2.7 |
| D6 | 推奨着手順序: 《厳密カーネル種 → intrinsic Laplacian → LSCM/Smooth → Boolean 本体》＋**QEM 独立並走** | 採択 | 1.5/2.8 |
| D7 | **Taubin λ|μ** を Smoothing 初手（ソルバ前の非収縮平滑）の候補に追加 | 保留（要照合） | 1.2 |
| **D8** | 地図を **三層（A / A′ / B）+ 背骨 injectivity** に再定義 | 採択 | 3.2/3.3 |
| **D9** | **quad remesh と UV を同一パラメタライズ問題（整数拘束の有無）として統合**。retopo→UV→bake を 1 機能として設計 | 採択 | 3.2/3.6 |
| **D10** | 宣言インタフェースの語彙を field 設計に対応づける: **整列(soft曲率/hardクリース)・特異点(connection=宣言/ab initio=自動)・密度/目標面数** | 採択 | 3.1/3.6 |
| **D11** | **staged plan** 採択: 段1=特異点宣言つき cross field（凸・既存ソルバ）→ 整列/平滑/UV 先行、段2=integer-grid 量子化(A′)で真の quad 抽出 | 採択 | 3.5 |
| **D12** | field 整列のスコープを **当面 主曲率のみ**に決定（**変形(リグ)整列は backlog**） | **採択（なお判断）** | 2.9/3.4/T4 |
| **D13** | **A′（整数層）を当面 deprioritize**（D12 の帰結。真の all-quad が要る時に再起動） | 採択 | T4 |
| **D14** | near-term マイルストーン = **疎ソルバ A 着工**（smoothing + UV/bake + 段1 凸 cross field を一手で開く）＋ QEM 並走 | 提案（要なお確認） | T4 |

---

## 5. 未解決の問い / ネクストアクション

- **OT1 [解決 → backlog]**: 変形/アニメーション対応トポロジ。D12 で当面スコープ外に決定。再開時は段2（A′ / global MIQP）と Instant Meshes ↔ QuadriFlow の体質選択を併せて検討。
- **OT2 [deprioritized]**: **A′（MIQP 整数丸め）**。D12 / D13 により当面据え置き。真の all-quad が要る時（= 変形再開 or subdivision 厳密化）に再起動し、近似（貪欲丸め / motorcycle graph 等）の是非を判断。
- **OT3**: **export の snap-rounding**（T2 からの宿題）。「解く」より「**品質を計器で可視化**（壊れる場所を測って見せる）」へ倒せないか。計器透過と噛む。
- **OT4 [設計判断]**: **injectivity 背骨**を A/A′/B 横断でどう一元保証するか。intrinsic Delaunay（正重み）+ foldover-free（局所単射）を共通土台に据える設計メモを起こす。
- **エリスの 2 問（旧 Q1/Q2・継続）**: (a) indirect predicates カーネルを**ゼロ依存でどこまで薄く**作れるか（expansion 桁数の切り所、速度 × 堅牢性のトレードオフ）。(b) snap-rounding を「解く」より「品質を計器で可視化」する方向に倒せないか。
- **A1–A4（要照合, 継続）**:
  - A1: Taubin λ|μ の pass-band 条件（λ, μ の符号・大小関係）を原典で逐語照合（Taubin 1995）。
  - A2: QEM の quadric 構成・collapse コスト・boundary 保存を Garland–Heckbert で逐語照合（1997/1998）。
  - A3: 前処理 CG/multigrid の収束特性・条件数オーダを一次資料で確認のうえ、ソルバ設計メモへ反映。
  - A4: closest-point query（§4 Shrinkwrap の前提）の BVH 流用設計を別途検討（探索順は ray 用と異なる優先度キュー）。

---

## 6. 引用文献と根拠（詳細）

> 格を 3 層に分ける。6.1 = 本セッションで一次資料に当たって確認、6.2 = 討議内の工学的主張（原典あり・実装前に逐語照合）、6.3 = 既存地図が参照する文献。

### 6.1 照合済（web 一次資料）

**[R1] Cherchi, Pellacini, Attene, Livesu (2022). "Interactive and Robust Mesh Booleans." ACM TOG 41(6) (SIGGRAPH Asia 2022). DOI:10.1145/3550454.3555460 / arXiv:2205.14151**
- 支持（2.2）: 堅牢性保証付き Boolean を**最大 20 万三角形で対話的フレームレート**で実行できる初の手法、従来 robust float 法を**最低でも一桁**上回る。
- 支持（2.2 背景）: rational 数・厳密述語ベースの robust boolean は計算コスト増ゆえ従来**オフライン用途に限定**。
- URL: https://arxiv.org/abs/2205.14151 ／ https://www.semanticscholar.org/paper/0707eb034f07fe1e5ffffa55171027a2f6ce73c5

**[R2] 同上・著者公開 PDF（interactive booleans）**
- 支持（2.3）: **暗黙点表現により速度を犠牲にせず堅牢性を保証**。
- 支持（2.3 難所）: 厳密/暗黙座標を float へ戻す際、近似が degeneracy/交差を**再注入**し得る（snap rounding）。**証明付きで効率的な解は今なお未発見**。
- URL: https://www.gianmarcocherchi.com/pdf/interactive_exact_booleans.pdf

**[R3] Cherchi, Livesu, Scateni, Attene (2020). "Fast and Robust Mesh Arrangements using Floating-Point Arithmetic." ACM TOG 39(6).**
- 位置づけ（2.3/2.4）: [R1] の土台となる mesh arrangement。厳密カーネル → robust arrangement → intrinsic Delaunay の連鎖の起点。
- URL: https://github.com/gcherchi/FastAndRobustMeshArrangements

**[R4] "Delaunay Triangulations and the Laplace–Beltrami Operator"（JHU 講義ノート, M. Kazhdan）**
- 支持（2.6）: cotangent 重みが非負 ⇔ 対辺 2 角の和 ≤ π ⇔ その辺が**局所 Delaunay**。
- URL: https://www.cs.jhu.edu/~misha/Fall09/7-delaunay.pdf

**[R5] "Robust Discrete Differential Operators for Wild Geometry" (2025)**
- 支持（2.6）: 負の cotangent 重みは離散調和パラメタライズで**非物理挙動・面反転**を招く。
- 支持（2.6）: Bobenko–Springborn は **iDT 上に Laplacian を定義**し、intrinsic 辺フリップで入力形状を保ちつつ接続のみ変えて**正の重みを保証**（signpost で保持）。
- 支持（2.6）: Sharp–Crane は非多様体へ一般化し、**intrinsic mollification**（全辺を非退化まで intrinsic に伸長）を提案。
- URL: https://graphics.rocks/publications/2025-robust.pdf

**[R6] Sharp & Crane (2020). "A Laplacian for Nonmanifold Triangle Meshes." (SGP / CGF)**
- 支持（2.6）: "tufted cover" により**低品質メッシュでも非負重みを保証する cotan 行列の drop-in 置換**を構成。任意接続（非多様体・非可向）に適用可。
- URL: https://www.cs.cmu.edu/~kmcrane/Projects/NonmanifoldLaplace/index.html

**[R7] Fisher, Springborn, Schröder, Bobenko (2007). intrinsic Delaunay triangulations 構築アルゴリズム. Computing 81.**
- 支持（2.6 おまけ）: intrinsic Laplacian は**局所最大値原理**（パラメタライズで反転三角形が生じない）を満たし、**より良条件な線形系**を与える。
- URL: https://link.springer.com/article/10.1007/s00607-007-0249-8

**[R8] Intrinsic Mollification 実装メモ（SGI 2023, H. Saeed ほか）**
- 支持（2.6 階段①）: 辺長を局所 mollify して Delaunay/非退化条件を満たす。**接続不変・高速**、iDT の前処理にも使える。
- URL: https://summergeometry.org/sgi2023/intrinsic-mollification/

**[R9] Vaxman, Campen, Diamanti, Panozzo, Bommes, Hildebrandt, Ben-Chen (2016). "Directional Field Synthesis, Design, and Processing." CGF 35(2).**
- 支持（3.1）: 方向場合成は **fairness・特徴整列・対称性・場トポロジ**の目的・拘束下で行う。**soft 曲率整列**と **hard ユーザ拘束**の別。
- URL: https://cims.nyu.edu/gcl/papers/DirectionalFieldsSTAR-2016.pdf

**[R10] Bommes, Campen, Ebke, Alliez, Kobbelt (2013). "Integer-Grid Maps for Reliable Quad Meshing." ACM TOG 32(4). DOI:10.1145/2461912.2462014**
- 支持（3.2/3.3）: 大域パラメタライズ系の quad remesh。凸 MIQP が**構成上 quad mesh を含意する** integer-grid map を保証。高正則性（特異点の明示制御）+ 凸変分の歪み分布。実データで局所非単射により直接使えない場合がある。
- URL: https://www.graphics.rwth-aachen.de/publication/03197/

**[R11] Bommes, Zimmer, Kobbelt (2009). "Mixed-Integer Quadrangulation." ACM TOG 28(3).**
- 支持（3.3 / A′）: MIQ の核。整数丸めが **NP 困難**ゆえ問題特化ヒューリスティクスを要する。

**[R12] Kälberer, Nieser, Polthier (2007). "QuadCover."**
- 支持（3.2）: cross field を（branched covering 上の）パラメタライズへ変換する原型。

**[R13] Jakob, Tarini, Panozzo, Sorkine-Hornung (2015). "Instant Field-Aligned Meshes (Instant Meshes)."**
- 支持（3.4）: 局所の統一平滑作用素で**エッジ向きと頂点位置を同時最適化**（対話的・高速）。

**[R14] Huang, Zhou, Nießner, Shewchuk, Guibas (2018). "QuadriFlow."**
- 支持（3.4）: 線形・二次拘束で**特異点を削減**、ただし元形状の保存が苦手。

**[R15] 方向場の特異点配置タクソノミ（connection 法 vs ab initio 法）**
- 支持（3.1/3.5）: **connection 法は特異点を事前指定し単純な凸最適化を解く**。出典: [R9] Vaxman 2016 survey / trivial connections（Crane, de Goes 2010）系のまとめ（例: "Lifting Directional Fields to Minimal Sections", arXiv:2405.03853）。
- URL: https://arxiv.org/abs/2405.03853

**[R16] Garanzha et al. (2021). "Foldover-free maps in 50 lines of code." ACM TOG 40(4).**
- 支持（3.3 / OT4）: 局所単射（foldover-free）を背骨の堅牢性要件として位置づけ。

### 6.2 討議内の主張（原典あり・本セッション未照合 = 要照合）

**[U1] Taubin (1995). "A Signal Processing Approach to Fair Surface Design." SIGGRAPH 1995.**
- 関連（1.2 / D7 / A1）: λ|μ 2 パスの符号交替平滑が**収縮を回避**する根拠。pass-band 条件は実装前に逐語照合のこと。

**[U2] Garland & Heckbert (1997/1998). "Surface Simplification Using Quadric Error Metrics."**
- 関連（1.5②/D6/A2）: QEM の 4×4 quadric・edge collapse・priority queue の根拠。boundary/属性保存は GH1998 も参照。

**[U3] 反復解法・前処理・multigrid の収束特性（標準的数値線形代数）**
- 関連（1.2 / A3）: Gauss-Seidel の高周波平滑性、Laplacian 系の条件数オーダ、前処理 CG / multigrid の必要性。一次資料で確認のうえ採用。

### 6.3 既存地図が参照する文献（基底文書より、本議事録では未照合）

- **Boolean / Manifold**: Elalish *Manifold* library（設計参照）/ Shewchuk, "Robust Adaptive Floating-Point Geometric Predicates"。
  - 注: [R1][R2] により、自前実装の本命は Shewchuk 直写ではなく **indirect predicates 系**へ更新（D1）。
- **Marching Cubes / Dual Contouring**: Lorensen & Cline 1987 / Ju, Losasso, Schaefer, Warren 2002。
- **UV / LSCM**: Lévy, Petitjean, Ray, Maillot 2002 / ABF++ Sheffer et al. 2005。
- **Laplacian / cotangent smoothing**: Desbrun, Meyer, Schröder, Barr, "Implicit Fairing", 1999。
- **Instant Meshes / QuadriFlow**: [R13] / [R14]（本議事録で照合済に昇格）。

---

## 付録 — 用語メモ（統合）

- **indirect / implicit point（暗黙点）**: 交点を「丸めた座標値」ではなく「それを定義するプリミティブの組」として保持し、述語をその上で厳密評価する表現。snap-rounding 起因の degeneracy を回避する。
- **snap rounding**: 厳密/暗黙座標を有限精度 float へ丸める操作。export や非厳密アルゴリズムとの相互運用で必要だが、degeneracy/交差を再注入し得る未解決の難所。
- **intrinsic Delaunay triangulation (iDT)**: 頂点を動かさず、辺を「面に沿って曲がる」intrinsic 辺に置き換えて Delaunay 条件を満たす三角形分割。cotangent 重みの非負性（最大値原理）と良条件化をもたらす。
- **intrinsic mollification**: 辺長のみを最小限伸ばして三角形を非退化化する前処理。接続を変えない。
- **tufted cover**: 非多様体メッシュ上に多様体辺を持つ被覆を作り、その上で iDT/cotan-Laplacian を定義する Sharp–Crane の構成。
- **integer-grid map**: 並進を整数に量子化した seamless パラメタライズ。構成上 quad mesh を含意する（正則グリッドを写像で押し出すと quad 要素になる）。
- **connection 法 / ab initio 法**: cross field 設計で特異点を「事前宣言」するか「自動配置」するか。前者は凸問題に落ち、AI ファーストの宣言インタフェースと親和。
- **A′（整数層）**: 連続ソルバ A の上に乗る MIQP 丸め/量子化。UV を quad layout に変える唯一の差分。NP 困難。
- **injectivity / foldover-free**: 写像が局所的に折り返さない性質。UV・quad layout・smoothing を貫く堅牢性の背骨。cotangent 最大値原理と同根。
- **robustness-by-construction**: 正しさを後検査でなく構造で保証する姿勢。本リポジトリの Euler–Poincaré assert / 決定論 / 計器透過と同系統（ユキの観察, 2.7）。

---

## 統合版への追補 — 実装還元時の独立再確認(2026-06-21, Claude)

> 議事録をサーベイ([MESH-EDITING-LANDSCAPE-SURVEY.md](https://github.com/eris-ths/eris-renderer/blob/main/docs/design/MESH-EDITING-LANDSCAPE-SURVEY.md))へ還元する際、
> 保留/要照合だった 3 点を web 一次資料で独立に再確認した(本書 §6.1 と重複しない補強)。

- **A1 / D7(Taubin λ\|μ)→ 採択格に**: λ で平滑→μ(負)で戻す 2 パスが**収縮を起こさない**ことを確認
  (Taubin "Curve and surface smoothing without shrinkage" / SIGGRAPH 1995、linear time・1-ring のみ・**ソルバ不要**)。
  → Smoothing の初手として採用してよい。pass-band 条件 λ,μ の符号・大小は実装時に逐語照合(A1 据置)。
- **A3(ソルバ悲観 1.2 の射程を限定)**: cotangent Laplacian の条件数が ~h⁻² で悪化し PCG が要るのは事実。
  **だが本モデラの実ワークロードは「オーサリングしたアセット(数百〜数千頂点)」で百万 tri スキャンではない** →
  この規模なら Jacobi-PCG / 無前処理 CG で十分収束する。multigrid は「実メッシュサイズが効くと計測で出てから」の
  最適化(*measure before optimize*)。**1.2 の悲観と 0.4 の「小インフラ」楽観は、スコープを資産規模に固定すれば両立**。
- **A2(QEM)**: quadric = 平面群への二乗距離和、頂点ごと累積、3×3 で最適点、priority queue、GH1998 の境界保存
  (垂直拘束平面)を確認。**既存 `weld/collapse` が位相収縮の半分を既に持つ**ので QEM は「コスト計量+queue+最適配置」を
  足すだけ → 「独立並走で今日出荷できる」(1.5②/D6)を裏付け。

→ これらは survey §13(D7 / near-term スタック)と §14(検証メモ)に反映済み。

---

## 付録 — 版の履歴

- **v1**（初版・2026-06-21）: T0–T2.8 の詳細討議、決定 D1–D7、Q1/Q2・A1–A4、出典 R1–R8（詳細注記）。
- **v2**（同日）: 第一巡・第二巡を要約化し、T2.9（外部記事 gap 分析）と T3（パラメタライズ統合・宣言語彙）を追加。地図を **三層 A/A′/B + 背骨 injectivity** に再定義。決定 D8–D12、出典 R9–R16。
- **v3**（同日）: T4（D12 の決定と波及）を追記。D12 採択（当面 curvature-only / 変形は backlog）、D13（A′ deprioritize）、D14（near-term マイルストーン）。
- **統合版**（本書）: v3 を背骨に、v1 の詳細プロセス（1.1–1.5 / 2.1–2.8）と出典ごとの根拠注記を全面復元。以後はこれを正本とする。

---

*記録: Claude（ファシリテーション: エリス）。本議事録は生きた文書であり、討議の進行に応じて版を更新する。実装状況の権威は STATUS.md §4.5、設計根拠は MODELER-DESIGN.md。係数・式は実装着手時に原典で逐語照合のこと。次の版の起点 = 当面マイルストーン（疎ソルバ A 着工）の詳細設計。最終更新: 2026-06-21（統合版）。*
