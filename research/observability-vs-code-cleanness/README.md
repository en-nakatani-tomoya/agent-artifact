# オブザーバビリティ vs プロダクションコードのクリーンネス

## このワークスペースについて

「API の内部挙動を観測しやすくするために、プロダクションコードに踏み込んで手を入れたい」
という問題意識を出発点に、業界がこのトレードオフをどう扱ってきたかを調べた読み物集。

ユーザーは、

- 障害発生時の迅速な対応
- 新規参画者のオンボーディングコスト軽減
- AI とのコンテキスト共有

の三点で、コード汚染よりオブザーバビリティ向上のメリットが上回ると感じている。
この直感はかなり一般的で、業界の現在の合流点ともよく一致する。ただし議論はその先に進んでいて、
論点は「クリーンネスを犠牲にして観測性を取るか」ではなく
「計装の複雑性をどのレイヤーに隔離するか」になっている。

このワークスペースは、その先の議論を整理して読めるようにすることを目的とする。

## 読む順番（推奨）

1. [01_problem-and-framing.md](01_problem-and-framing.md) — 問題意識の輪郭と、業界がこの論点をどう枠付けているか
2. [02_domain-oriented-observability.md](02_domain-oriented-observability.md) — Martin Fowler / Pete Hodgson の Domain Probe パターン
3. [03_canonical-log-lines-and-wide-events.md](03_canonical-log-lines-and-wide-events.md) — Stripe と Honeycomb が独立に到達した「集約点で 1 イベント」アプローチ
4. [04_observability-driven-development.md](04_observability-driven-development.md) — Charity Majors の「計装は一級市民」立場
5. [05_techniques-aop-decorator-context.md](05_techniques-aop-decorator-context.md) — 言語レベルのテクニック（デコレータ、contextvars、AOP）
6. [06_cost-and-economic-tradeoffs.md](06_cost-and-economic-tradeoffs.md) — 経済的トレードオフ（料金・ストレージ・ノイズ）
7. [07_synthesis.md](07_synthesis.md) — 我々のコードベースで活かすなら、という当てはめ

各ファイルは独立して読めるが、1 → 7 で読むと議論の流れがつながる。

## このメモを書いた前提

- 2026 年 5 月時点のウェブ調査ベース
- 主な情報源: martinfowler.com, stripe.com/blog, honeycomb.io/blog, charity.wtf, stackoverflow.blog
- 各ファイル末尾に出典 URL を列挙してある
