# 04. Observability-Driven Development — 計装は一級市民

## 立場の輪郭

Charity Majors（Honeycomb 共同創業者・CTO）が体系化した立場。
著書 *Observability Engineering*（O'Reilly, 2nd ed.）の第 11 章が ODD に充てられている。

主張は強い: 「計装はコードの汚染ではない、コードの一部だ」。

ユーザーが感じている「コード汚染よりオブザーバビリティのメリットが上回る」という直感を、
さらに一歩進めて「そもそもトレードオフではない」と再定義する立場。

## TDD との対比

ODD は TDD の発想を観測性に拡張したもの。

TDD:
1. テストを先に書く（期待する挙動を宣言する）
2. 実装する
3. テストを通して、リファクタする

ODD:
1. 計装を先に考える（本番でこの機能の何を観測したいかを宣言する）
2. 実装する（計装込みで書く）
3. 本番にデプロイして、計装を通して挙動を観察する

> developers write code with the intention of declaring the outputs and specification
> limits required to infer the internal state of the system

TDD は「正しさ」をユニットテスト粒度で保証する。ODD は「本番での挙動」をテレメトリ粒度で
保証する。TDD のフィードバックループは数秒、ODD は数分〜数十分。両者は補完関係にあり、
排他関係ではない。

## Shift Left の意味

「Shift Left」は元々セキュリティ業界の用語で「テストやセキュリティ検証を開発工程の早い段階に
持ってこい」という主張。ODD はこれを観測性に適用する。

従来のフロー:
- 開発 → デプロイ → 障害発生 → 計装を追加 → 再デプロイ → 観察

ODD のフロー:
- 計装を考える → 開発（計装込み）→ デプロイ → 観察 → 仮説検証 → 改善

決定的に違うのは、最後のステップが「障害対応のため」ではなく「仮説検証のため」になっている点。
本番が継続的な実験環境になる。

## "Test in Production" という挑発的な主張

Charity Majors の有名な主張のひとつが "Test in Production" あるいは
"You can't fully test before production"。

ステージング環境はどれだけ整えても本番の完全な複製にはならない。負荷、データ分布、ユーザー行動、
外部 API の挙動、これらすべてが本番固有。だから「本番で観測しながら検証する」ことを正面から
受け入れろ、というのが彼女の立場。

これを成立させる前提が、

- フィーチャーフラグ
- カナリアリリース
- 即時ロールバック
- そして高品質な観測性

の組み合わせ。観測性が貧弱だと「テスト・イン・プロダクション」はただの賭けになる。
逆に観測性が十分なら、本番で起きていることをほぼリアルタイムで把握できる。

## "Every Log is Sacred" の否定

ODD と相補的に、Charity Majors は「すべてのログを大事に残す」立場を否定する。

> reject the 'every log is sacred' mentality in favor of strategic sampling

戦略は次の通り:

- 戦略的サンプリング: 興味のあるイベント（エラー、遅延、特定ユーザー）は 100% 残し、
  普通のリクエストは 0.1% などに落とす
- 高カーディナリティを優先: 詳細なフィールドを持つ少数のイベントは、薄い大量のログより価値が高い
- 集約は後でやれ: 生イベントを残しておけば後から好きに集計できる

これは「コードを汚してでもログを増やす」のではなく、「コードに書く計装の質を上げて、量は減らす」
方向の主張。

## デバッグとオンボーディングへの影響

ODD が成熟したチームでは、デバッグの体験が本質的に変わる。

従来:
1. 障害発生
2. ログを grep する
3. 仮説を立てる
4. 関連箇所のコードを読む
5. 「ここのログが足りない」とコードを足す
6. デプロイし直す
7. 再現を待つ

ODD のチーム:
1. 障害発生
2. wide events を多次元でスライス（例: 「過去 5 分の P99 latency 上昇に共通する次元」を機械的に探す）
3. 高カーディナリティの軸で異常を特定（特定 SKU、特定リージョン、特定実験グループ）
4. 解像度の高い state がそこにあるので、修正に直行

新規参画者にとっての価値が大きい。コードベースを読まなくても「現在この API で何が起きているか」
が wide events を通じて見える。Charity Majors 自身が
"newcomers can understand systems through their telemetry, not just their source code"
と書いている。

## AI / LLM との接点

これがユーザーが挙げた三つ目の理由「AI とのコンテキスト共有」と直接つながる。

Charity Majors は最近、観測性と AI の関係について三つの軸を示している。

1. モデル訓練 (AI 自体の挙動を観測する)
2. LLM 駆動の開発支援
3. AI 生成コードの本番観測

特に 3 番目について、彼女ははっきり書いている。

> good AI observability can't exist in isolation; it must be embedded in good software
> observability. As generative AI produces more code, comprehensive instrumentation
> becomes essential for understanding behavior in production.

AI がコードを書く速度が上がる → 本番で何が起きているかを問い直す帯域が必要 → 観測性が前提条件。

我々が AI と作業するとき、AI に渡すコンテキストの質がアウトプットの質を決める。
ソースコードだけ渡すより、「このリクエストで実際に何が起きたかの wide event」を渡せた方が、
AI は遥かに具体的な仮説を立てられる。これはユーザーの直感と完全に一致する。

## ODD が成り立つ条件

ODD はマインドセットだけでは成立しない。前提となるインフラがある。

- 構造化イベント基盤（Honeycomb、Datadog、New Relic、自前 ELK + α など）
- 高カーディナリティに耐える保管・検索
- リクエスト単位で wide event を吐く仕組み（03 章の canonical log lines や wide events）
- リリース・ロールバック・フラグの仕組み

これらが揃わない段階で「ODD やるぞ」と言っても空回りする。
逆に言うと、ODD は段階的にしか到達できない理想であり、一気に切り替えるものではない。

## ODD が向かない場面・批判

- 強い規制下のシステム（金融、医療など）: 「本番で試す」が許されない
- 観測コストが本質的に高い領域（モバイル、IoT、エッジ）
- スタートアップ初期: そもそも本番トラフィックがなく、ODD のフィードバックが回らない

批判としてよく聞くのは、

- 「テストを書かない言い訳」になりがち（ODD ≠ テスト不要、両立する）
- 「サンプリングで重要な情報を取りこぼす」（戦略次第）
- 「観測性ツールへのベンダーロックイン」（OpenTelemetry がここを緩和する）

## 我々のコードベースへの含意

ユーザーの問題意識はほぼ ODD の文化的前提と一致している。

- 障害対応の迅速さ → ODD の主要なメリット
- オンボーディング → ODD が掲げる "テレメトリで理解する" の効用
- AI とのコンテキスト共有 → ODD の最新トピックそのもの

なので、計装を「汚染」と見なさず、「コードの一部」として書き換えるマインドセットの転換は、
ユーザーの方向性と矛盾しない。ただし、ODD は計装を増やせと言っているのではなく、
「計装の質を上げて、後から好きに切り出せる形にしろ」と言っている点が肝心。

実際の手段は 02（Domain Probe）と 03（Wide Events）にある。ODD は方向性、02・03 はその実装。

## 出典

- [Observability Engineering, Ch. 11 Observability-Driven Development — O'Reilly](https://www.oreilly.com/library/view/observability-engineering/9781492076438/ch11.html)
- [How observability-driven development creates elite performers — Stack Overflow Blog](https://stackoverflow.blog/2022/10/12/how-observability-driven-development-creates-elite-performers/)
- [Observability: the present and future, with Charity Majors — Pragmatic Engineer](https://newsletter.pragmaticengineer.com/p/observability-the-present-and-future)
- [charity.wtf — observability category](https://charity.wtf/category/observability/)
- [Charity Majors on Observability-Driven Development and Honeycomb — Ambassador](https://www.getambassador.io/podcasts/charity-majors-observability)
