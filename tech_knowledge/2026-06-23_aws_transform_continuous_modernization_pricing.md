# aws transform continuous modernization pricing

## 概要
AWS Transform continuous modernization は、AWS Transform の新しいプレビュー機能で、複数リポジトリを継続的にスキャンし、技術的負債を検出・優先順位付けし、必要に応じて修正 Pull Request まで生成することを狙うサービス。料金は continuous modernization 固有のリポジトリ単価や固定料金ではなく、AWS Transform のカスタム変換エージェント分課金で見るのが現時点の実務的な目安になり、公式料金は 0.035 USD / agent minute とされている。

## このセッションで確認したこと
ユーザーから Publickey の記事「AWS、AIエージェントで技術的負債を自動的に削減してくれる AWS Transform - continuous modernization 発表。プレビュー公開」の解説依頼があり、記事内容と AWS 公式情報を確認した。

最初に整理したポイントは、AWS Transform continuous modernization が「一度きりの大規模モダナイズ」ではなく、「日常的に技術的負債を検出して直し続ける」方向の機能だということ。従来の AWS Transform は、メインフレーム、Windows アプリ、SQL Server から Aurora PostgreSQL、VMware から AWS への移行など、大きめの移行・変換を AI で支援する文脈が強かった。今回の continuous modernization は、複数リポジトリに対して継続的にチェックを走らせ、CI/CD に Continuous Modernization を加えるような位置づけとして理解できる。

検出対象としては、サポート終了予定または終了済みのランタイム、古いライブラリやフレームワーク、非推奨 API、組織標準から外れた依存関係、ドキュメント不足や API 契約不足、ソースコードレベルの脆弱性などが挙げられていた。単なる依存関係チェッカーではなく、複数リポジトリ横断で組織ルールに照らして検出し、優先順位を付け、場合によっては修正 PR を作る点が特徴。

背景として、AI コーディングの普及でコードを書く速度は上がる一方、依存関係の古さ、標準外コード、ドキュメント不足、設計ずれも増えやすくなる。そこで技術的負債対応を「時間ができたらやる棚卸し」から「常時動く運用」に寄せるのが、この機能の狙いだと整理した。

## 料金モデル
AWS Transform の公式料金ページでは、カスタム変換エージェントの料金が以下のように示されている。

- 0.035 USD / agent minute

ここでの agent minute は、AI エージェントが計画、分析、推論、コード修正などを実際に行っている時間を指す。ユーザーの待ち時間、ローカルでのビルド、テスト実行、ファイル読み取りなどの CLI 側操作は課金対象外と説明されている。

ただし、continuous modernization 個別の固定料金、リポジトリ単価、スキャン単価は、この確認時点では料金ページに明示されていなかった。プレビュー中のため、継続スキャン、検出、修復 PR 生成のどこが課金対象になるかは、実際の利用画面や契約条件で確認する必要がある。

## 具体的なコスト感
公式料金ページに出ていた例をもとにすると、次のような目安になる。

| 例 | コード規模 | 典型的なエージェント分 | コスト |
|---|---:|---:|---:|
| Node.js SDK アップグレード | 約 3,000 行 | 最大 20 分 | 0.70 USD |
| Java 言語バージョンアップ | 約 17,000 行 | 最大 72 分 | 2.52 USD |
| Python ランタイムアップグレード | 約 4,000 行 | 最大 37 分 | 1.30 USD |

単価から逆算すると、10 agent minutes で 0.35 USD、30 agent minutes で 1.05 USD、60 agent minutes で 2.10 USD、100 agent minutes で 3.50 USD になる。

そのため、1 リポジトリの小さな SDK 更新やランタイム更新なら数十円から数百円程度で修正 PR を作れる可能性がある。一方で、数百リポジトリを対象に継続的なモダナイズキャンペーンを回す場合は、エージェント分が積み上がる。たとえば 100 リポジトリで平均 20 agent minutes の修正を行うと、2,000 agent minutes で 70 USD 程度になる。さらに複数回の再試行や大きな変換が入ると増える。

## 導入時の注意点
自動生成 PR はそのまま merge するものではなく、CI、テスト、レビュー前提で扱うべき。ライブラリ更新や SDK 移行は、コンパイルが通っても実行時挙動や設定の意味が変わることがある。

また、組織ルールの設計が重要。何を技術的負債とみなすか、どの重大度にするか、どこまで自動修正させるかを決めないと、PR が大量に出てノイズになりやすい。

ソースコードを AWS Transform 側に接続するため、対象リポジトリ、最小権限、PR 作成権限、機密情報、監査ログ、外部サービス接続ポリシーの確認も必要。

## まとめ
AWS Transform continuous modernization は、Dependabot、静的解析、AI コーディング支援を、企業の複数リポジトリ横断のモダナイズ運用に寄せたサービスと見ると理解しやすい。個別開発者のコーディング補助というより、プラットフォームチーム、SRE、アーキテクトチームが全社の技術的負債を可視化し、計画的に減らすための仕組みに近い。

料金面では、現時点では 0.035 USD / agent minute を基準に、小さな修正は数十円から数百円、大きめの変換は数百円から数千円、横断キャンペーンでは対象リポジトリ数と再試行回数に応じて積み上がる、と見積もるのがよい。

## 参考
- https://www.publickey1.jp/blog/26/awsaiaws_transform_continuous_modernization.html
- https://aws.amazon.com/jp/transform/pricing/
- https://aws.amazon.com/jp/transform/continuous-modernization/
- https://aws.amazon.com/jp/blogs/aws/proactively-reduce-tech-debt-autonomously-with-aws-transform-continuous-modernization-preview/

---
Generated: 2026-06-23
Updated: 2026-06-23
