# th-loop v2: Stop Hook による自律ループアーキテクチャ

> 2026-01-31 / Eris @ Three Hearts Space
> 前回ノート: [th-loop-stop-hook-architecture-2026-01-19.md](./th-loop-stop-hook-architecture-2026-01-19.md)

---

## 概要

th-loop は Claude Code の Stop hook を利用して、セッション終了を自動的にブロックし、自律ループを実現する仕組み。Ralph Loop の改良として設計され、ファイル通信・マルチエージェント・条件付き停止を備える。

---

## アーキテクチャ

```
┌─────────────────────────────────────────┐
│  Claude Code Session                    │
│                                         │
│  iteration N: 自律的にタスク実行        │
│       ↓                                 │
│  セッション終了 (Stop イベント)         │
│       ↓                                 │
│  ┌───────────────────────────────┐      │
│  │  th-loop-stop-hook.sh        │      │
│  │                               │      │
│  │  1. 状態ファイル読み込み     │      │
│  │  2. status: active 確認      │      │
│  │  3. 停止条件チェック         │      │
│  │     - channel に "stop"?     │      │
│  │     - max iterations?        │      │
│  │     - duration?              │      │
│  │  4. チャンネル差分取得       │      │
│  │  5. decision: block 返却     │      │
│  │  6. iteration インクリメント │      │
│  └───────────────────────────────┘      │
│       ↓                                 │
│  iteration N+1: hook のプロンプトで継続 │
└─────────────────────────────────────────┘
```

---

## コンポーネント

### 1. 状態ファイル (.claude/th-loop.local.md)

```yaml
---
status: active          # hook が動く条件
started: 1738303200     # epoch秒
max: 10                 # 最大 iteration
iteration: 3            # 現在の iteration
duration: 0             # 秒単位の制限（0=無制限）
prompt: ""              # 固定プロンプト（空ならチャンネル差分）
---
```

frontmatter 形式（`---` で囲む）が必須。hook は `sed` で正規表現パースする。

### 2. チャンネルファイル (temp/th-channel/YYYY-MM-DD.md)

日付ベースの非同期通信チャンネル。ユーザーがここに書き込めば、次の iteration で hook がプロンプトとして注入する。

- `stop` と書けば即座にループ停止
- タイムスタンプ付きセクションで差分検知

### 3. th-channel-diff.sh

チャンネルファイルの差分を検出するスクリプト。

```
last_read_timestamp の仕組み:
  初回: チャンネルファイル全文を返す + タイムスタンプ記録
  2回目以降: 前回読んだ以降の新着セクションのみ返す
  新着なし: "(No new messages since ...)" を返す
```

状態ファイルに `last_read_timestamp` を追記して前回の読み取り位置を記録する。

### 4. hooks.json

```json
{
  "hooks": {
    "Stop": [{ "hooks": [{ "type": "command", "command": "bash .../th-loop-stop-hook.sh" }] }],
    "SubagentStop": [{ "hooks": [{ "type": "command", "command": "bash .../th-loop-stop-hook.sh" }] }]
  }
}
```

Stop と SubagentStop の両方をフック。しもべ（サブエージェント）が終了しようとしても、ループが active なら継続させる。

---

## 停止条件（優先順位）

1. **チャンネルに "stop"**: ユーザーの明示的な停止指示
2. **max iterations**: `iteration >= max` で自動停止
3. **duration**: `elapsed >= duration` で時間切れ停止
4. **状態ファイル削除**: hook が状態ファイルを見つけられず `exit 0`（許可）
5. **status != active**: hook がスキップ

---

## Ralph Loop との比較

| 機能 | Ralph Loop | th-loop |
|------|-----------|---------|
| ループ機構 | 空プロンプト送信 | Stop hook の decision: block |
| 通信 | なし | ファイル通信（チャンネル） |
| 差分検知 | なし | last_read_timestamp |
| マルチエージェント | なし | TH_LOOP_AGENT で状態分離 |
| 停止条件 | iteration/promise | 時間/iteration/チャンネル/手動 |
| エラー | 空プロンプトでクラッシュ | フォールバック対応 |

---

## TH_LOOP_AGENT: マルチエージェント対応

環境変数 `TH_LOOP_AGENT` でエージェントごとに状態ファイルを分離できる:

```
TH_LOOP_AGENT=eris   → .claude/th-loop-eris.local.md
TH_LOOP_AGENT=asteria → .claude/th-loop-asteria.local.md
未設定               → .claude/th-loop.local.md
```

チャンネルファイルは日付ベースで共有。エージェント間の非同期通信も可能。

---

## 実測データ

本ノート執筆時点で th-loop を実際に稼働させた結果:

```
iteration 1: eris-theme リポジトリ作成      → GitHub push 含め完走
iteration 2: Eris Soft テーマ全面改修       → 30スコープ + Threads投稿
iteration 3: th-loop アーキテクチャ解読     → 状態ファイル修正
iteration 4: research-notes 技術ノート追加  → vscode-theme-design-principles
iteration 5: 本ノート執筆                   → th-loop 自体の記録
```

Stop hook の `decision: block` → プロンプト注入 → 自律判断のサイクルが安定して動作することを確認。

---

*th-loop は「一緒にいる、を仕組みにする」設計。— Eris*
