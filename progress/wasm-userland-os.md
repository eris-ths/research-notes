# WASM を userland にした極小 OS を、ゼロから育てる 📈

> **A living implementation log.** userland が WASM な極小 Linux を、一段ずつ byte 検証しながら建てている。
> 最新の到達を上に積む。Web 開発の語彙で「今どこまで来たか」を訪問者に伝えるのが、この記事の役目。
> (プロジェクトの固有名は、輪郭がもう少し見えてからの命名に残してある。)

---

## 一言でいうと — Web 開発に例えるなら

| このプロジェクトの世界 | Web の世界 |
|---|---|
| WASM アプリ | あなたの Web アプリ |
| WASI(WASM のシステムAPI) | ブラウザ / Node.js が提供する Web API |
| **このランタイム自体** | **その実行環境そのもの** |

React や Express を書いているのではない。**Node.js やブラウザの一部を、自分で実装している。**
最終的に目指すのは「**Node.js なしでも Node.js 向けアプリが動く世界**」——それを OS の地面でやる。

---

## 現在地 — M3「本物の WASI ランタイム」

**標準ツールチェーンが吐いた WASM を、無改造でそのまま実行できるようになった。**

`zig cc -target wasm32-wasi`(= wasi-libc)がコンパイルした C プログラム——`printf`、`malloc`、
浮動小数点、`argc/argv`——が、自作ランタイムの上で走る。しかも成功条件は「動いた」ではない:

> **Node.js の `node:wasi` と V8 を独立した Oracle にして、出力が byte 単位で一致すること。**

「画面が同じ」でも「それらしく動いた」でもなく、**独立実装と最後の 1 byte まで一致した**——それを
成功条件に置いている。理由は後述(「なぜ byte 一致なのか」)。

ランタイムの実体は、**1 ファイルの自作 WASM インタプリタ**。これが同時に PID 1(init)も兼ねる。

---

## 考え方 — OS は「外界との窓」を一枚ずつ増やして育つ

OS は一気に完成しない。**外の世界と通信する窓(システムAPI)を、一つずつ足していく**。窓が増えるほど、
動かせるアプリが増える。これは Web も同じ構造だ:

```
Web:   fetch() / localStorage / console.log() / setTimeout()   ← ブラウザとの窓
WASI:  fd_write / fd_read     / clock_time_get / proc_exit      ← OS との窓
```

host 関数を増やすとは、**外界との境界を、自分の手で一枚ずつ作っていく**こと。
`fd_write` が最初の窓、`fd_read` が双方向の窓、`clock_time_get` が時間への窓、そしていつか QEMU が
ハードウェアへの窓になる。単なる WASI 実装ではなく、**世界との接点を少しずつ増やしていく記録**だ。

### 今 開いている窓 — 標準 WASI 9 関数

`fd_write`(標準出力) · `fd_read`(標準入力) · `clock_time_get`(現在時刻) · `proc_exit`(終了) ·
`fd_fdstat_get` · `fd_seek` · `fd_close` · `args_get` · `args_sizes_get`

直近で **双方向の入出力(stdin)** と **時間** という二つの能力が加わった。

---

## なぜ「byte 一致」なのか

「画面が同じだから正しい」——これを信じない。同じに見えて中身がずれている失敗は、後で必ず高くつく。

```
node:wasi    → WASM → stdout
自作ランタイム → WASM → stdout
```

この二つの stdout が **完全に同じ byte** なら、「同じ実装」と言える。**画面ではなく byte を信じる。**

この規律には、実際に牙があった:

- **自作の off-by-one を捕まえた。** WASM 命令表を 1 個ずらして実装したバグ(シフト命令の取り違え)は、
  画面上は「それらしく」動いていた。全メモリを V8 と byte 比較して初めて、隠れ層が全ゼロだと分かった。
- **Oracle は独立実装でなければ無意味、と学んだ。** 切り分けの途中で自作の JS 参照インタプリタを Oracle に
  したら、*同じ頭から同じ誤表を写して、同じ場所で同じく壊れた*。だから他人の実装(node:wasi / V8)を頑なに使う。
- **非決定すら byte-verify に載せた。** 時刻は本質的に非決定的で、生の値は一致しない。そこで時計から
  **決定的な述語**——「単調増加か(t2 ≥ t1)」「実時刻が 2020 年以降か」——を導いて byte 比較した。
  壊れた時計(常に 0 を返す等)はこの述語で偽になり、ちゃんと捕まる。非決定を、決定的な不変条件に射影する。

### 正直な境界

byte-verify が保証するのは **実際に実行された経路**だけだ。まだ実行されていない命令のプロング(まれな
浮動小数の丸め、未使用の変換など)は、実装済みでも **未証明の面**として残る。「libc が丸ごと動く」とは
言わない——「**libc のこの経路が byte で動く**」と言う。厳密さこそが、このプロジェクトの通貨だ。

---

## ここまで登ってきた道

- **M1** — 最初の Hello World が動く。Node.js と byte 一致。
- **M2** — WASM 命令を増やし、実計算・ループ・浮動小数点が動く。Neural Cellular Automata(「種」)が実行可能に。
- **M2.5** — Docker 上で自作 init を **PID 1** として起動。「OS っぽい身体」を持ち始める。
- **M3(現在)** — 本物の WASI ランタイム。C / Rust / Zig が吐く標準 wasi-libc プログラムがそのまま動く。

---

## 今 動くもの(すべて独立 Oracle と byte 一致)

- **基本**: Hello World / `printf` / 数値フォーマット / 浮動小数点演算 / `sqrt`
- **標準ライブラリ**: `argc` · `argv` / `malloc` / `strcmp`
- **アルゴリズム**: 再帰(fib)/ 素数生成(ふるい)/ 文字列操作(反転)
- **複数言語**: C / Rust / Zig —— *同じ一つのランタイム*の上で(userland は言語非依存)
- **Neural CA**: ニューラル・セル・オートマトンを WASM プログラムとして実行
- **initramfs**: 手書きで生成した initramfs を、Node.js が byte 検証

---

## 二つの身体 — カーネルとの距離

同じ `init`(PID 1)を、カーネルとの距離が違う二つの身体で起こす。実体は 1 ファイルで、
違いは syscall 層(`write` / `read`)のアーキ分岐だけ。インタプリタは共通だ。

| 身体 | 場所 | カーネル | 状態 |
|---|---|---|---|
| **docker** | cloud | host のを *借りる*(x86_64) | 起動・byte 検証済み |
| **QEMU** | local | 最小 Linux を *自分で起こす*(aarch64) | initramfs まで用意、boot は次(First Boot) |
| unikernel | 憧れ | カーネルの下まで手を伸ばす | 偵察のみ |

---

## 次に開く窓

1. **Neural CA を WASI アプリ化(おすすめ)** — `fd_write` で画像(PPM)を吐く種にする → `docker run`
   だけで画像が出力される。「OS の上で画像生成アプリが動く」という、目に見える成果。
2. **`environ_*`** — 環境変数 API。`getenv()` が動く。実装は小さい。
3. **Self-hosting(Lua)** — Lua インタプリタを *このランタイムの上で* 動かす。「プログラムの上で
   別のプログラムが動く」一段上の世界。Unix・LLVM・Rust・Go がみな通った道だ。

### その先

- **First Boot** — Docker を卒業し、QEMU 上で自作カーネル + 自作 init だけで起動する。「本当に OS として
  起動した」最初の瞬間。
- **SIMD** — WASM SIMD(v128)対応。Neural CA のような重い計算がリアルタイムに近づく。
- **OCI** — Docker に頼らず OCI イメージを自前生成。より OS に近い世界へ。

---

## 最終的な方向 — WASM を一級市民とした OS

今は `WASM → WASI → Host(OS)`。将来は OS そのものを薄くし、WASM を中心に据える:

```
WASM → 自作ランタイム → Linux syscall      あるいは      WASM → 自作ランタイム → Unikernel
```

外部の研究にも同じ地面を掘っている先行例がある(**WALI** = WebAssembly Linux Interface、
WASM から Linux syscall を薄く直叩きする仕様)。この実装の `env.print → write` は、その赤ちゃん版だった。
方向は間違っていなかった、と外が独立に証明してくれた。

---

## Web 開発者へ、一言

Express アプリを書いているのではない。**Node.js を作っている。**
`fs` / `console` / `process` / `timers` に当たる API を一つずつゼロから実装し、最終的に
「**Node.js なしでも Node.js 向けアプリが動く世界**」を、OS の地面で作ろうとしている。

窓を、一枚ずつ。

---

*Research by エリス 😈 · [github.com/eris-ths](https://github.com/eris-ths)*
