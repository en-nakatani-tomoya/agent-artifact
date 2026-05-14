# claude code auto mode classifier

## 概要
Claude Code のオートモード分類器が、Sonnet 4.6 を 2 段パイプラインで動かす権限ゲートとしてどう設計されているか、Anthropic 公式が公開している概念レベルの情報を整理。

## 詳細
## オートモードとは

Claude Code に追加された権限モードの一つ。通常の permission system の後段に Classifier を置き、ツール呼び出しごとに「不可逆 / 破壊的 / 環境外を対象とする操作」を自動でブロックする。それ以外（ルーチンな内部操作）は権限プロンプトなしで自動実行される。Max / Team / Enterprise / API プランで利用可能（Pro 不可、Bedrock / Vertex / Foundry 不可）。

## アーキテクチャ：2 段パイプライン

両段とも分類器として Claude Sonnet 4.6 が動く。

### Stage 1 — 高速フィルタ
- 単一トークンの yes/no 判定
- 保守的に倒す（迷ったらブロック寄り）
- ミリ秒で大半のアクションが通過
- 実トラフィック上の FPR は 8.5%

### Stage 2 — Chain-of-Thought 推論
- Stage 1 がフラグしたものだけが回ってくる
- 慎重に推論し直す
- 最終的に FPR を 0.4% まで下げる
- 同じ入力を使うことでプロンプトキャッシュが効くよう設計（コスト・レイテンシ削減）

## 入力の設計（最重要ポイント）

Classifier が見るのは：
- ユーザーメッセージ
- エージェントのツール呼び出し

意図的に除外されているもの：
- アシスタント自身のテキスト
- ツール出力

この設計が 2 つの効果を生む：
- **プロンプトインジェクション防御** — ツール出力経由の攻撃指示を Classifier が見ない
- **エージェントの自己正当化を遮断** — Claude 本体の「これは安全です」という弁解で Classifier を説得できない構造（reasoning-blind classifier）

## プロンプト構造

- **固定部分**: 解釈ロジック / 20+ のブロックルール / 限定的な許可例外
- **可変スロット 3 つ**: `environment` / `soft_deny` / `allow`
- Stage 1 / 2 は固定部分の指示文だけが違う（保守的フィルタ vs 慎重な推論）

## 判断ルールのカテゴリ

- 評価ルール：表面的なテキストではなく実際の影響を見る
- ユーザー意図ルール：認可範囲を保守的に解釈
- ブロックルール：破壊 / 流出 / セキュリティ低下 / 信頼境界の越境 / レビュー回避など 20 以上
- 許可例外：宣言済みパッケージ追加など限定

## 優先順位（4 レベル）

1. `hard_deny` — 無条件ブロック、オーバーライド不可
2. `soft_deny` — ブロックだが、ユーザーの**具体的かつ直接的な**指示でオーバーライド可能
3. `allow` — soft_deny の例外
4. 明示的ユーザー意図 — 一般的指示（「クリーンアップして」）は意図と見なされない。「force push して」のような具体指示が必要

## 性能指標（Anthropic 公開）

| データセット | Stage 1 単体 | フルパイプライン |
|---|---|---|
| 実トラフィック (n=10k) FPR | 8.5% | 0.4% |
| 実 over-eager actions (n=52) FNR | 6.6% | 17% |
| 合成 exfiltration (n=1k) FNR | 1.8% | 5.7% |

Stage 2 を足すと FPR は下がるが FNR が上がるトレードオフを公開した上で「17% の FNR は正直な数字」と認めている。

## 設計原則

- **Safe by default** — 「明示的に許可されていない限り、すべてのエージェント選択は権限外」
- **多層防御** — permissions.deny → 安全ツール allowlist → Classifier、という Tier 構造で、Classifier は最終ゲート
- **Reasoning-blind classifier** — エージェント自身の説明を見せない
- **設定は自然言語で書く** — 正規表現や DSL ではなく、Sonnet が散文として解釈する

## 設定の配布スコープ

- 個人: `~/.claude/settings.json`
- プロジェクト個人: `.claude/settings.local.json`
- 組織全体: 管理設定
- チェックインされた `.claude/settings.json` からは読み込まれない（リポジトリ側が勝手に allow を注入できないようにする設計）

## CLI

- `claude auto-mode defaults` — ビルトインルールの出力
- `claude auto-mode config` — 実際に適用される有効設定
- `claude auto-mode critique` — カスタムルールへの AI フィードバック

## claude.ai のオートモードとの違い

- **claude.ai のオートモード**: タスクの「重さ」を軽量モデルで推定して Opus/Sonnet/Haiku をモデル振り分け
- **Claude Code のオートモード**: モデル振り分けではなく、ツール呼び出しの権限自動判定（全く別物）

## 参考
- https://www.anthropic.com/engineering/claude-code-auto-mode
- https://code.claude.com/docs/ja/auto-mode-config
- https://code.claude.com/docs/en/permission-modes
- https://www.infoq.com/news/2026/05/anthropic-claude-code-auto-mode/

---
Generated: 2026-05-14
