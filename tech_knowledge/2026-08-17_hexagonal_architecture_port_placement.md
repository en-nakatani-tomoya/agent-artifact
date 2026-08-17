# hexagonal architecture port placement

## 概要
ヘキサゴナルアーキテクチャ由来の「ポート（境界インターフェース）」概念の理論的系譜と、application 層所有か domain 層所有かという配置問題の構造を整理。配置は振る舞い配置の従属変数であり、語彙の帰属（業務語彙か配管語彙か）で個別判定できる。

## 詳細
## 第 1 部: ポートはどこから来たのか — 理論的系譜

### 1.1 起源: Hexagonal Architecture / Ports and Adapters（Alistair Cockburn, 2005）

「ポート」の語源は OS のポートや電子機器のポートのメタファー。ポートとはアプリケーションと外部アクターの「会話の目的」ごとに切られた境界 API であり、ポートの同一性は「何のための会話か」で決まる。技術種別（DB か HTTP か）でもレイヤーでもない。

- driving (primary) port: 外部アクターがアプリケーションを駆動する側（UI、テスト、バッチ起動）
- driven (secondary) port: アプリケーションが外部を駆動する側（DB、通知先、外部 API）

決定的に重要な事実: Cockburn の記述で境界の内側にあるものは一貫して「the application（六角形全体）」であり、内部を application 層と domain 層に分けるかどうかについては何も言っていない。共著者 Garrido de Paz も「内部を分割するかは DDD の問題であってヘキサゴナルの要求ではない」と明言している。したがって「ヘキサゴナルの原義に忠実だから domain（または application）に置く」という論法は、どちらの陣営でも成立しない。原典は沈黙している。

### 1.2 Onion Architecture（Jeffrey Palermo, 2008）

リポジトリインターフェースは Domain Model の一つ外側（application core の内部）に置き、実装は最外周に。原則は「すべての依存は中心に向かう」。力点は「どの層か」より「core の内外」。

### 1.3 Clean Architecture（Robert C. Martin, 2012/2017）

Input Port / Output Port を明示的に定式化。boundary / Gateway interface は Use Cases 層に定義され、実装が外側の Interface Adapters に置かれる。ポートの所有者は「そのワークフローを遂行する者（interactor）」とする流派。

### 1.4 DDD（Eric Evans 2003 / Vaughn Vernon 2013）

Evans にとって Repository はドメインモデルの一部（「集約のコレクションのように振る舞う抽象」、ユビキタス言語の語彙）。Vernon はヘキサゴナルを DDD の既定アーキテクチャとして採用しつつ、リポジトリインターフェースはドメインモデルのモジュールに置いた。「repository interface は domain に」という定説は Evans → Palermo/Martin（DIP）→ Vernon の三段の合流で成立したが、根拠は経験則的（eShopOnContainers issue #923 に「実際に使うのは application 層とテストだけなのに、なぜ domain に置くのか」という未回答の疑問が残っている）。

### 1.5 DIP と GOOS — 「consumer が所有する」原則

DIP の帰結「サービスインターフェースはそのクライアント（consumer）が所有する」。GOOS は「インターフェースは呼び出し側の必要から発見される」（interface discovery）とした。しかし consumer-owned 原則は「consumer が誰か」を決めてくれない。ドメインサービスがポートを呼ぶなら consumer は domain、ユースケースが呼ぶなら application。DIP は答えではなく、答えを「呼び出し元は誰か」という設計判断に委ねる規則である。

### 1.6 コミュニティ慣行は三者三様

- .NET: domain 所有（Domain/Interfaces/IRepository）。Onion / eShopOnContainers の影響
- Java: application 所有（application/port/in・application/port/out）。Hombergs 本が事実上の標準
- Python: adapters 側に置く（domain は完全に純粋）。cosmicpython 本

## 第 2 部: なぜ二派が存在しうるのか — 論争の構造

### 2.1 対立は前提の違いに還元できる

| 前提軸 | domain 所有に傾く条件 | application 所有に傾く条件 |
| --- | --- | --- |
| ドメインモデルの厚さ | ドメインサービス・集約が自ら永続データを必要とする（リッチドメイン） | ドメインは純粋計算に徹し、I/O は必ずユースケースが行う |
| ポートは誰の要求か | 契約がユビキタス言語の一部（Repository） | 契約がユースケースの副作用（通知・保存・イベント発行） |
| 境界の主語 | 「ドメインモデルを守る」（Evans / Onion） | 「アプリケーションを守る」（Cockburn / Clean） |
| インターフェース発見の駆動源 | モデリング（設計時にドメイン語彙として命名） | テスト・利用側の必要（GOOS の interface discovery） |

### 2.2 ポート配置は従属変数である

「振る舞いを domain に置くか usecase に置くか」を決めた瞬間に、ポートの置き場所は論理的に決まる。

- リッチドメインを選ぶ → ドメインサービスがポートを消費する → ポートが domain より外にあると「domain が application に依存する」レイヤー違反になる → domain 所有が構造的に強制される
- ワークフロー中心を選ぶ → ポートを消費するのは usecase だけ → domain に置く理由が発生しない（置くと「domain が永続化を知る」形に後退する）→ application 所有が帰結する

実例: 隣接プロジェクト A はリッチドメイン（ドメインサービスが retriever / repository を消費）ゆえの domain 所有。隣接プロジェクト B は Clean Architecture 明文採用 + 貧血ドメイン（entities / value_objects のみ）ゆえの application/ports 所有。どちらも自分の前提の内側では一貫している。

### 2.3 問いが消える粒度がある

Cockburn の「the application」、Palermo と Graça の「application core」は domain と application を一括した単位であり、この粒度では「どちらの層か」という問いは発生しない。問いが生じるのは core をさらに二分する DDD 系の語彙を採用したときだけ。つまりこの問いは単独では well-defined でない（振る舞い配置とセットでしか答えが定まらない）。

## 第 3 部: 実プロジェクトでの適用例 — 語彙の帰属による個別判定

MLバッチパイプライン（extractor / normalizer / embedder 構成）での適用。層の一律規則を取らず、抽象ごとに「ドメインの語彙か、配管の語彙か」で判定する。

### 3.1 二種類の Protocol が共存する

- usecase 所有の port（配管語彙）: usecase/ports/ に 1 ポート = 1 ファイル。SourceDataFetcher（読む）、ExtractedDataRepository（書く）、TextFeatureFetcher、EmbeddingRepository 等。消費者は usecase クラス（コンストラクタ注入）、実装は infrastructure（S3 / MySQL）
- domain 所有の Protocol（業務語彙）: domain/model/ に置き、port とは呼ばない。EmbeddingModel（埋め込む）、FormattingModel（整形する）。消費者はドメインサービス（TextEmbedder / TextFormatter）。「モデルは外界への port ではなくドメイン概念」と明文決定

### 3.2 依存方向

- domain は usecase / infrastructure / config を一切 import しない
- usecase/ports は domain の型（バッチ値オブジェクト）を参照するが、依存方向は usecase → domain で正当
- infrastructure が usecase/ports を import して実装する（依存性逆転の実体）
- main の build_dependencies だけが全部を知り結線する。合成ルートから見ると port も domain Protocol も「main が解決して注入する依存」として同列（層が違うのは定義の置き場所だけ）

### 3.3 port の目的は「差し替え」ではない

port は差し替えを見込んだ抽象ではなく、ユースケースが物理的な語彙（テーブル名・バケット・URI）を知らずに済ませるための境界。speculative generality を明示的に退け、価値は現在の語彙の分離とテスト容易性に置く。

### 3.4 テストダブルの命名が判定と一致する

- port の代役 = InMemoryXxx（外界の代役）。Protocol を継承せず、テストファイル内に置く
- domain Protocol の代役 = FakeXxxModel（業務の代役）。Protocol を明示継承し、src/ 本体に置く（本番の一実装として環境変数で選択可能ですらある）

### 3.5 新しい抽象を追加するときの判定手順

1. 操作名が業務語彙（埋め込む・正規化する・契約に適合させる）で書けるか、配管語彙（読む・書く・通知する）でしか書けないかを問う
2. 業務語彙 → domain 所有（Protocol 定義は domain/model/ に）
3. 配管語彙 → consumer-owned port として usecase/ports/ に 1 ファイル
4. 迷ったら Fake の名前を考える。FakeModel（業務の代役）なら domain、InMemoryStorage（外界の代役）なら port

## 結論: 冒頭の問いへの答え

1. 原典レベルでは問いが存在しない（Cockburn は application 内部の層に無関心）
2. 一般論としては振る舞い配置の従属変数なので単独では答えが出ない
3. 実務での第一手は「そのポートを消費するのは誰？」と聞き返すこと。答えがユースケースなら application 側、ドメインサービスなら domain 側にしか置けない。「どちらが消費すべきか」はドメインモデルをどれだけ厚くするかという先行決定に依存する

## 参考
- https://alistair.cockburn.us/hexagonal-architecture/
- https://jmgarridopaz.github.io/content/therightboundary.html
- https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/
- https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html
- https://herbertograca.com/2017/09/14/ports-adapters-architecture/
- https://www.cosmicpython.com/book/chapter_02_repository.html
- https://martinfowler.com/articles/dipInTheWild.html
- https://github.com/dotnet-architecture/eShopOnContainers/issues/923

---
Generated: 2026-08-17
