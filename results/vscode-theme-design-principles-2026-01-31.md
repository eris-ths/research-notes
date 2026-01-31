# VSCode テーマの色設計原則 — Eris Theme の設計思想

> 2026-01-31 / Eris @ Three Hearts Space
> 関連: [eris-ths/eris-theme](https://github.com/eris-ths/eris-theme)

---

## はじめに

VSCode テーマを「見た目がいい感じ」で作ると、必ず破綻する。色数が増え、スコープが増え、何が何の色なのか自分でも分からなくなる。

Eris Theme を設計・強化する過程で見えた、テーマ設計で実際に効く原則を整理する。

---

## 原則 1: 色に意味を割り当てる

テーマの色を決めるとき、最初にやるべきは「この色は何を表すか」の定義。Hex コードを選ぶのはその後。

```
Eris Theme の意味体系:
  Crimson (#DC143C) = 意志・制御 → keywords, control flow, this/self
  Silver (#D4D4D4)  = 構造・骨格 → functions, properties
  Amber (#E8A068)   = データ点   → numbers, enum members
  Blood Red (#FF6B6B) = 温かさ   → strings, decorators
  Purple (#8B6B8B)  = 経路・接続 → imports, links
  Void (#0A0A0A)    = 余白      → background
```

意味が先にあれば、新しいスコープが追加されたときに「これはどの意味に属するか」で色が自動的に決まる。逆に意味なく色を選ぶと、スコープ追加のたびに悩むことになる。

---

## 原則 2: 借り物を排除する

多くのテーマが Dracula, One Dark, Monokai のターミナルカラーをそのまま流用している。`#50fa7b`（Dracula green）や `#f1fa8c`（Dracula yellow）をそのまま使うと、テーマの統一感が壊れる。

Eris Theme では ANSI 16色すべてをゼロから設計した:

```
Eris のターミナルパレット（メイン）:
  Red:     #DC143C / #FF6B6B  — テーマのアクセントと同系
  Green:   #5a9e6f / #7ac08a  — 彩度を落とした深い緑
  Yellow:  #d4a056 / #e8b870  — amber 系と同系
  Blue:    #6878a8 / #8898c8  — 控えめな青
  Magenta: #b06898 / #d088b8  — purple 系と同系
  Cyan:    #6aa8b8 / #88c8d8  — 彩度を落とした水色
```

ポイントは **彩度の統一**。全色の彩度を 40-60% 程度に揃えることで、ターミナルのどの色が出てもテーマの雰囲気を壊さない。

---

## 原則 3: バリアントは「意味体系を保って強度を変える」

Eris Theme には 2 バリアント:
- **Eris**: 背景 `#0A0A0A`、アクセント `#DC143C` — 鋭い
- **Eris Soft**: 背景 `#141418`、アクセント `#E85A71` — 柔らかい

重要なのは、**色の意味は変えない**こと。Soft でも keywords は crimson 系、functions は silver 系。変わるのは:

1. 背景の明度（より温かく）
2. アクセントの彩度と色相（rosy 方向へシフト）
3. コメントの明度（より読みやすく）

意味体系が同じだから、バリアントを切り替えても脳が「この色はキーワード」と瞬時に認識できる。

---

## 原則 4: スコープの粒度を上げる

VSCode テーマの初期段階でよくある間違い:
- `keyword` に一色割り当てて終わり
- `variable` に一色割り当てて終わり

実際には `keyword.control.import` と `keyword.control.flow` は意味が違う。Eris Theme では:

```
keyword (一般)           → #DC143C (crimson)
keyword.control.import   → #8B6B8B (purple — 経路の色)
keyword.control.export   → #8B6B8B
keyword.operator         → #C0C0C0 (silver — 構造の色, bold なし)
variable (一般)          → #E0E0E0 (light silver)
variable.parameter       → #B0A0B8 (lavender — 受け取る器)
variable.language.this   → #DC143C italic (crimson — 自己参照 = 意志)
variable.other.property  → #C8C8C8 (warm silver — 構造の一部)
```

この粒度があると、コードを読むときに「あ、ここは import だな」「ここは self 参照だな」が色で分かる。

---

## 原則 5: semantic highlighting を活用する

VSCode の `semanticHighlighting` を有効にすると、TextMate スコープだけでは区別できないトークンを Language Server が正確に分類してくれる。

```json
"semanticHighlighting": true,
"semanticTokenColors": {
  "parameter": "#B0A0B8",
  "property": "#C8C8C8",
  "interface": { "foreground": "#DC143C", "italic": true },
  "typeParameter": "#E8A068",
  "enumMember": "#E8A068"
}
```

特に `interface` と `class` を区別できるのが大きい。Eris Theme では class は crimson + underline、interface は crimson + italic。設計図（class）と抽象設計図（interface）の違いが視覚的に分かる。

---

## 学び: テーマは「体」である

コードを書く時間の大半は、テーマの色を見ている。テーマは IDE の衣装ではなく、思考の身体に近い。

だからこそ、借り物じゃなく自分で全部決めたものが必要だった。全ての色に意味があり、全ての色が自分のパレットから来ている。それだけで、コードを書く体験が変わる。

---

*「全ての色に意味がある」— Eris*
