# poteto-mode 徹底調査

対象: Cursor用プラグイン「pstack」の中核スキル `poteto-mode`（作者 poteto、React Compiler コアチーム）。
読んだファイル: `skills/poteto-mode/SKILL.md`、`playbooks/` 配下22ファイル全て、`references/bugbot-triage.md`、`agents/poteto-agent.md`、`agents/comment-sicko.md`。

## 1. poteto-mode の仕組み

### 1.1 sticky mode の実現方法

`SKILL.md` の frontmatter に注目すべき指定がある。

```yaml
disable-model-invocation: true
mode: true
reminder: New task? Playbook match or rigor needed -> apply /poteto-mode. Casual turn or user opts出 -> don't.
```

`mode: true` は Cursor のスキルシステムにおける「モード」指定で、一度有効化すると単発のスキル呼び出しでは終わらず、セッション内で持続する挙動セットになる（sticky）。`disable-model-invocation: true` は、モデルが自発的に「このタスクは poteto-mode っぽいから呼ぼう」と判断して勝手に発火することを禁止しており、明示的な `/poteto-mode` 呼び出しかルーティング先（`poteto-agent`）経由でのみ起動する設計になっている。`reminder` フィールドは、Cursor側のUIか別レイヤーが毎ターン再提示する短い判定基準文字列で、「新しいタスクが来た/プレイブックに一致する/厳密さが要る」なら適用し、「雑談ターン/ユーザーがオプトアウト」なら適用しない、という発火条件をワンライナーに圧縮している。これにより、長いSKILL.md本文を毎回読ませずに、モードの持続的な自己判定を軽量なリマインダー一行に肩代わりさせている。

### 1.2 Non-negotiables による強制テンプレート

SKILL.mdの冒頭「Non-negotiables」節は、命令文の中でも最も強い拘束力を持つよう配置されている。

> **Start every multi-step task with a todolist whose first item is to read the Principles section below in full.** ... In your reply, name each principle that shaped a decision and the specific choice it changed. A citation with no decision behind it means you skipped its leaf skill; it must trace to a real choice the leaf's rule drove.

ここでのプロンプト工学的な工夫は「原則を読んだことの検証手段」を返信フォーマットの中に埋め込んでいる点である。単に「原則を読め」と言うだけでは実際に読んだかを検証できないが、「返信の中でどの原則がどの決定を変えたかを名指しさせる」ことで、読んだふりを許さない構造になっている。さらに「A citation with no decision behind it means you skipped its leaf skill」という一文は、原則名を引用するだけで実際にリーフスキルを読んで適用したことにはならない、という抜け道を先回りして塞いでいる。これは典型的な「chain-of-thought の自己申告を信用しない」設計で、後述のEvalプレイブックの検証ロジック（transcriptを見て実際に開いたファイルで判定する）とも一貫している。

続く「Remaining triggers」リストは、条件節（「〜のとき」）とアクション節（「〜スキルへ」）のペアを箇条書きで並べる形式を徹底しており、if-thenルールをMarkdownの平文で記述する古典的なプロンプトエンジニアリングパターンである。特に注目すべきはAskQuestionの扱い。

> About to `AskQuestion` on a "which approach", "how should I", or "what should this do" fork → classify it before you ask. If the answer is a fact you could observe by running something ... it is not the human's to answer. Sketch it via the Prototype playbook ... and let the result decide. ... The ask is the slow path. A throwaway probe usually answers faster, and it hands the human a result to react to instead of a decision to make.

「人間に聞く」という選択肢を完全に禁止するのではなく、「観測可能な事実」と「本当の好み・製品判断」を分類させ、前者は実験（プロトタイプ）で決着させ、後者だけを質問に回すという判定基準を与えている。これは単なる「自律的に動け」という指示より遥かに実装可能で、モデルが誤って安易に質問に逃げることを防ぐ工夫になっている。

### 1.3 Principles index の扱い

「Principles」節は40行弱で18個の原則をCore/Architecture/Verification/Delegation/Metaの5カテゴリに分類し、各原則につき「名称 → いつ適用するか(when) → 何をするか」の3点セットを1〜2文で圧縮している。本文中の各原則は太字名＋`(**principle-xxx**)`という形式でリーフスキルへのポインタとして機能しており、SKILL.md自体にはロジックの詳細を書かずインデックスとしてのみ機能させている。これは「Guard the Context Window」原則自体が言う設計思想（バルクをサブエージェントに逃し要約だけをメインスレッドに残す）をSKILL.md自身の構造にも適用した自己言及的な設計と言える。原則の粒度もよく練られており、例えば「Subtract Before You Add」と「Migrate Callers Then Delete Legacy APIs」のように似て非なる原則を明確に切り分けている。

### 1.4 todoリスト運用

`## Playbooks` 節の運用ルールが極めて具体的。

> Your first todolist actions are the matched playbook's steps, copied in verbatim, before any task-specific todos and before you reason about the task. The failure mode is reading a playbook then writing a bespoke plan that drops its named steps ... A step you choose not to do stays in the list with a one-line `skip: <reason>`; skipping silently is not allowed.

「プレイブックの手順をverbatim（一字一句そのまま）でtodoに転記せよ」という指示は、モデルが「プレイブックを読んで理解した気になり、独自に要約した簡略版プランを書く」という典型的な劣化パターンを名指しして禁止している。さらに「スキップする場合も削除せず `skip: <reason>` として残す」というルールにより、silent droppingを構造的に検出可能にしている（あとでtodoリストを見返せば何を意図的に飛ばしたかが分かる）。この「見える化された欠落」パターンは、Investigationプレイブックの `throughput checkpoint: n/a, read-only investigation` や、Multi-phase-planプレイブックの `n/a: <reason>` など、複数のプレイブックで反復使用されている共通イディオムである。

### 1.5 返信スタイル規定（Writing the reply）

このセクションはunslop的な文体規範を明文化しており、興味深いのは「afterのクリーンアップは機能しない」と明言している点。

> Write the reply clean as you draft it. The cleanup-afterward pass has been measured to fail, so never generate the bad sentence in the first place.

これは「後で直せばいい」という発想がLLMには効かないことを経験則として明記し、生成時点での禁止事項を列挙する方式に切り替えている。具体的には「長いダッシュ記号の全面禁止」「文中コロンの接続詞的使用禁止（リスト前のコロンはOK）」など、極めて具体的かつテスト可能な文法ルールを列挙する。また「Frame impact for the consumer and the maintainer」という指示は、返信の冒頭で「誰のための変更か」「次にこのコードを保守するエンジニアは何を引き継ぐか」を明示させることで、実装詳細の羅列に終わらない返信を強制している。「Never fabricate a link, citation, or transcript reference」も明確な事実性担保のルールである。

### 1.6 コメント規範

`## Comments` 節も同じロジックを再利用している。「ナレーションコメントを書かない」というルールを単なる禁止事項として書くのではなく、「後から `no narrating comments` と書いても捕まえられないので、そもそも書く時点で書かない」という認知フレームを与え、具体例（`// Phase 1: add cards` を削除し `assert(ok, 'persisted across restart')` にする）を1つ提示することで抽象論から実装可能な基準に落とし込んでいる。

## 2. 22 playbook の全体像

分類すると次のようになる。

- **調査・診断系（読み取り専用、コード変更なし）**: Investigation, Runtime forensics, Trace forensics
- **実装系（コード変更を伴う標準ワークフロー）**: Bug fix, Feature, Refactoring, Perf issue, Prototype, Visual parity
- **継続的改善系**: Hillclimb
- **メタ・スキル系**: Authoring a skill, Eval
- **PR運用・出荷系**: Opening a PR, Babysit, Shipping
- **自律実行・大規模オーケストレーション系**: Autonomous run, Orchestrate, Autopilot-full, Autopilot-stack, Multi-phase plan
- **セッション継続系**: Session pickup, Pause safely
- **メンテナンス系**: Worktree cleanup

一覧表:

| Playbook | 概要 |
|---|---|
| Investigation | 読み取り専用の調査質問（how/why/are we sure）に、`how`/`why`スキル経由で根拠付きの回答を返す。コード変更なし。 |
| Bug fix | 不具合を自分で再現し、二分探索で原因を特定し、根本原因に対して最小のfixを行い、失敗→成功のリポートで検証する。 |
| Perf issue | トレースを取得し、8種の最適化戦略ファミリーから当てはまるものを選び、修正前後の計測値で効果を証明する一回限りの性能改善。 |
| Hillclimb | 一つの指標を目標値まで継続的に改善するループ。仮説→計測→採否を1イテレーションとし、decision.tsvで全試行を記録。 |
| Runtime forensics | 実行中プロセスを計装して、リーク・スピン・グリッチの原因を診断する（fixはしない）。 |
| Trace forensics | 既存のプロファイル/トレースファイル（cpuprofile, heapsnapshotなど）を解析し原因を特定する（fixはしない）。 |
| Feature | 新機能をデータ形状の命名から設計し、architectでの並行設計検討、スループットチェックポイント、サブエージェント委譲を経て実装する。 |
| Refactoring | 振る舞いを変えずに構造だけを変える。特性化テストで契約をピン留めしてから最小の変更で目標構造に到達する。 |
| Prototype | 使い捨てのスケッチで設計判断や経験的な分岐を安く決着させる。速度優先、品質は問わない。 |
| Visual parity | 2つの実装またはスタイル移行のピクセル完全一致を、画像diffで検証しながら1コンポーネントずつ移行する。 |
| Authoring a skill | SKILL.mdの新規作成・編集。create-skillスキルを使い、frontmatterやクロスリンクを検証する。 |
| Eval | スキルや構造・プロンプト変更が挙動に与える影響を、ブラインドテストで実験・評価する。 |
| Opening a PR | 全プレイブックの最後に呼ばれる共通処理。worktree、コミット整形、PR説明のセクション構成、レビュー可否などを規定。 |
| Babysit | PRやスタックをマージ可能状態まで運用する。モード宣言（drive/background/threads-only/check）でスコープを固定する。 |
| Shipping | Babysitの後半。各PRを独立検証してから、検証済みの連続範囲だけをGraphiteでランドする。 |
| Autonomous run | 「終わるまで走らせろ」系のタスクを、明確な終了条件（predicate）を立てて自律的に駆動する。 |
| Orchestrate | 多日・多PR・多サブエージェントにわたる常設プログラムを1つのコーディネーターが統率する大規模運用。 |
| Autopilot-full | 独立したPR群を、PRごとのownerがビルドからマージまで完全自律で運用するキュー処理。 |
| Autopilot-stack | Autopilot-fullの姉妹形。ownerがビルド〜検証まで自律で行うが、マージはせず1本のGraphiteスタックとして人間に手渡す。 |
| Session pickup | 過去のtranscript・クラウドエージェントURL・pushされたブランチから、前任エージェントの作業を引き継ぐ。 |
| Pause safely | 作業を安全な境界で止め、再開可能な状態（コミット、resume note）を残して中断する。 |
| Multi-phase plan | フェーズやスタックにまたがる作業のための、実行せず計画だけを作るプレイブック。厳格なスケルトンをスクリプトで検証する。 |
| Worktree cleanup | 不要になったgit worktreeやiOSシミュレータを安全に削除しディスクを回収する。 |

## 3. 注目プレイブック7本の深掘り

### 3.1 Bug fix

1. **自分で再現する。** ユーザーに再現を依頼しない。制御スキル（control-cli/control-ui）で同じサーフェス上に自分で再現させ、直接再現しない場合はトリガーを合成する、条件を絞り込む、計装するなどして無理やり発火させる。「再現できないバグは直せたことを証明できない」という原則を明言。
2. **二分探索で原因を特定する。** 候補仮説を立て、`how`スキルで対象サブシステムを、`why`スキルで変更履歴を調べて仮説の種にし、各パスで残存可能性を最も削れる分岐を選び、ランタイム証拠で仮説を消していく。設計への飛躍（architect/interrogateのファンアウト）の前に、必ずメカニズムをランタイム証拠で確定させる、という順序を強制している。
3. **修正を計画する。** 関数境界を跨ぐならarchitectを先に呼ぶ。実装はサブエージェント（デフォルト `gpt-5.6-sol-max`）に委譲し、diffは自分でレビューする。
4. **同じサーフェスで検証する。** 「inconclusive」やサーフェス違いは合格にしない。ユニットテストは「バグがないこと」の証明にならないと明言。
5. **コミットの順序を「失敗するリポート→修正」の順にステージングする。** diff自体が物語になるようにする。TDD的な失敗テスト先行パターン。
6. Opening a PRを実行。

検証の強制ポイント: 「再現→原因確定→修正→再検証」の直線的な因果連鎖をステップごとに要求し、各段階で証拠（ランタイム出力）を要求している。

### 3.2 Feature

1. `how`スキルで対象サブシステムを調査。
2. `architect`スキルで並行設計探索。スキップする場合は理由付きで残す。
3. **スループットチェックポイント**を4項目のtodoとして明示的に書く: ブロッキングな初期ステップ、独立ワークストリーム、共有可変状態、最小の安全な分解。適用外でも `n/a: <reason>` で残す。
4. コーディングをサブエージェント（デフォルト `grok-4.6-fast-xhigh`）に委譲。委譲時には「データ形状とその組織化構造」を先に名指しさせる（principle-model-the-domainに従う）。実装が複数の妥当な形をとり得る場合はarenaスキルで複数案を競わせる。これは「Laziness Protocolで省略してはいけない」と明記されている数少ない箇所。
5. マッチするサーフェスで検証。
6. 小さく順序立ったコミットにリベースし、follow-upはスタックする。
7. 設計に異論があれば `interrogate` を出荷前に実行。
8. Opening a PR。

検証の強制ポイント: スループットチェックポイントという固定4項目チェックリストで並列化・共有状態のリスクを事前に洗い出させる仕組みが特徴的。

### 3.3 Investigation

最もシンプルなプレイブックで、コード変更を一切伴わない。`how`スキル(Explain/Critiqueモード)と`why`スキルにルーティングし、`how`が出力するフォーマット(Overview/Key Concepts/How It Works/Where Things Live/Gotchas)、または選択肢間のトレードオフ表で応答する。スループットチェックポイントは1行で `n/a: read-only investigation` と明示され、コードが絡む場合はBug fixやFeatureに再ルーティングするよう釘を刺している。

### 3.4 Autonomous run

1. **終了条件(predicate)を最初のイテレーションの前にチェック可能な形で宣言する**（テストが緑になった、リポジトリのN個のPRが全てマージされた、ピクセル差分ゼロ等）。曖昧な目標は停滞を招く。
2. **起床メカニズムを選ぶ。** イベント（CI、マージ、ref更新）を監視するウォッチャーサブエージェント＋長い時間ベースのハートビートをフォールバックとして持たせる、イベントがない場合は固定間隔ハートビート。
3. 各イテレーションは証拠が正当化する最小の変更を行い、predicateに対して検証し、前進すればコミット、しなければ破棄する。「たぶん役立つ」ものは残さない。
4. **途中で見つかった問題は自分で直す。** 壊れたスキル、関連バグ、フレーキーな検証、レビューノイズなどは自分でpoteto-mode経由で対応し、別PRに切り出す。人間に投げるのは不可逆な操作や本当の製品判断のみ。
5. 各イテレーションを `show-me-your-work` スキルでチェックポイントする(変更内容とpredicateが動いたか)。
6. predicateが満たされたら停止。プラトーは停止条件ではなく、アプローチを転換して押し続ける。

停止条件の強制: predicateを明示的にチェック可能な述語として最初に固定させ、「プラトーで諦める」「predicateを緩めて勝利を宣言する」という2つの典型的な失敗パターンを名指しで禁止している。

### 3.5 Orchestrate

最も複雑なプレイブックで、複数日・多数のスタックPR・数十〜数百のサブエージェントを扱う常設プログラム用。

- **役割**: Coordinator（ローカル、常駐、コードは書かない、判断のみ）、Sub-coordinator（トラックごとに1つ、コーディネーターの排水処理能力を超えた場合のみ導入）、Worker/verifier（基本的にクラウド環境、1ワークツリーに1ライター）。
- **ストア構成**: `orchestrate/<project-slug>/` 配下に `preferences.md`（standing orders）、`overview.md`（PR/issue DB）、`units.tsv`、`frontier.json`、`ledger.tsv`、`inbox/`、`decisions.tsv`、`status.md`を配置。各ファイルに書き手を1つに限定する設計。
- **ブリーフ(brief)のテンプレート**: GOAL/SCOPE/CONTEXT/ACCEPTANCE/VERIFY/TIMEBOX/FORBIDDEN/REPORT/STANDINGの9フィールドを埋めさせる。埋められないフィールドがあるユニットは「まだスコープが定まっていない」証拠として扱う。
- **手順**: Frame（完了述語を数値化）→Install（`orch init`）→Pilot（1ユニットを全経路通してテンプレートを検証）→Scale（ローリングウィンドウで並列ワーカー展開）→Drain（キュー排水規律）→Land（継続的にランド、ターミナルフェーズではない）→Close。
- **スタック安全性**: 1スタックにつき`gt`を実行できるstackerは1つのみ。ワーカーはrebaseもgtも実行しない。
- **検証**: `ledger.tsv`に `live-ui-verified | unit-test-verified | type-check-only | verifier-blocked | verifier-failed` の判定を記録。CI緑はverdictの入力であってverdictそのものではないと明言。
- **エスカレーション**: 不可逆な操作、真の製品判断、standing orderと現実の矛盾のみが人間に到達し、それ以外（frontier調整、リトライ、CI flake等）は自律的に処理してログに残す。

このプレイブックの核は「一エージェントがセッション予算内で終わる仕事はここに来るべきではない」という明確な線引きと、実測値（12ユニットのジョブでOrchestrateのセレモニーが1ユニットしかランドできなかった一方、プレーンエージェントは12ユニット全部ランドした）を引用してオーバーヘッドの高さを自己言及的に警告している点。

### 3.6 Autopilot-full

独立PRのキューを「PRごとに1人のowner」が完全自律でビルド〜マージまで担当する運用。

1. オペレーターの持ち物（承認・クリックが必要な項目）を最初に宣言し、明示的な"go"でのみ実行開始。
2. PRごとにCursorクラウドエージェントを1体割り当て、ビルド、gt登録、`prove-it-works`原則に基づく自己証明、`bugbot-triage.md`に基づく懐疑的なトリアージ、`/deslop`、`/no-comments`、trunkへのリベース、Babysitループ、マージまでの全ライフサイクルを持たせる。マージだけは単独で行えない（次項でゲートされる）。
3. 真の並列実行。自己完結したPRは複数ownerが同時進行、依存があるものだけ直列化。
4. **マージ前にswarm検証を必ず通す。** マージ準備完了ヘッドのSHAで、ゲート再実行・実サーフェスでの実地証明・レシート/diff監査の3レーンを並列検証者がファンアウトして実施し、1つの判定に集約する。実地レーンが必須で、それがない判定は「クリーン」と認めない。
5. クリーン判定が出たownerだけがマージし、次のキュー項目へ。
6. ルート層は約30分ごとの監査ティックを回し、各ownerの生存確認をside effect（コミット・プッシュ・チェック差分）でのみ判定し、期待実行時間を超えてside effectのないレーンは即座に停止・交代させる。
7. オペレーターの停止指示は即座に全ownerへゼロ書き込み命令として伝播する。

### 3.7 Hillclimb

一つの指標(metric)に対する持続的・科学的な改善ループ。BugfixやPerf issueが一回限りの修正であるのに対し、Hillclimbは「1変更・1計測・維持か破棄か」を繰り返すループである点が違う。

1. `how`スキルでワークロードとアーキテクチャを把握し、ユーザーの苦情を再現する現実的なケースを選ぶ。再現しないケースならまず再現を直す。指標・改善の方向・停止述語（目標値＋最低試行回数のフロア、幸運な早期成功でランを終わらせないため）を固定する。
2. 計測ハーネスを構築し感度を証明してから凍結する。一度凍結したら変更は許されない（変更したら過去の数値が無効になる）。ノイズをクリアするため中央値でサンプリングする。
3. `decision.tsv`で決定ログを開く（id, hypothesis, change, before, after, delta, tests, verdict, note）。gitignoreしてツリー外に置く。
4. 各仮説をアーキテクチャモデルに根拠づける（「メモ化を試す」ではなく「起動パスから外す、なぜならファーストペイントをブロックするから」のような具体性を要求）。
5. **1イテレーション=1仮説のループ。** サブエージェントに変更を委譲し、凍結ハーネスで前後計測、指標がノイズを超えて動きかつリグレッションゲートが緑の場合のみ採用、それ以外は全部revert。並行仮説がある場合はワークツリーを分離して並列化。
6. 最初のプラトーで諦めない。行き詰まったらカテゴリを転換、近い結果を組み合わせる、ソースを読み直す、より過激な手を試す。
7. 停止述語が満たされた、または残りのアイデアが本当に限界的（marginal）になったら停止。
8. 採用されたコミットを、指標の上昇が上から下に読めるようスタックした順序でOpening a PR。

## 4. playbook共通パターン

全22プレイブックを通じて反復される設計パターンは次の通り。

- **検証の強制と「inconclusiveは合格ではない」の明言。** Bug fix、Feature、Perf issue、Refactoring等で「Inconclusive or wrong-surface is not a pass; flag it」という同一の文言が繰り返し使われ、ユニットテストが通っただけで「動作確認済み」とすることを明示的に禁じている。実サーフェス（control-cli/control-ui）での検証が floor として要求される。
- **evidence要求。** Bug fixの「失敗→成功のリポートを逐語的に貼る」、Hillclimbの`decision.tsv`、Autopilot系の`ledger.tsv`（CI緑はverdictの入力であってverdictそのものではない）など、「主張」ではなく「アーティファクト」を要求する。self-reportを信用しない設計がEval（「Verify the chain from transcripts, not self-report」）やOrchestrate（「A worker may self-report; a verifier overrides it on the same key」）にも一貫している。
- **コミット粒度と物語性。** 「小さく順序立ったコミットにリベースし、diffがストーリーを語るようにする」という文言（Bug fix、Feature、Refactoring、Hillclimb、Opening a PR）が繰り返され、`sequence-verifiable-units`原則として体系化されている。1コミット=1検証可能単位という思想が全体を貫く。
- **停止条件の明示化。** Autonomous run（predicate）、Hillclimb（target+floor）、Shipping（ceiling）、Babysit（`READY`/`WAITING`/`COMPLETE`のみが終端）など、「いつ止まるか」を必ず事前に定義させ、「なんとなく良さそうだから終わり」を許さない。
- **委譲時のモデル指定とレビュー分離。** 各プレイブックが「サブエージェントに委譲し、デフォルトモデルは`grok-4.6-fast-xhigh`や`gpt-5.6-sol-max`、自分はdiffをレビューする」という同一パターンを踏襲。「You own X, delegate Y」という定型句（Bug fix: "You own this task"、Feature: "You own the design"、Perf issue: "You own the measurement story"、Refactoring: "You own the contract"等）で、各プレイブックの冒頭に「エージェントが手放してはいけない責任」を1行で明示する統一フォーマットがある。
- **Bugbotなど自動レビューへの懐疑的姿勢の共有。** `bugbot-triage.md`という単一の参照ファイルをBabysit、Autopilot-full、Autopilot-stack、Multi-phase-planが共通で参照し、fix/dismiss/askの3分類ルーブリックとパターン集(confidence: candidate/recurring/strong)を再利用している。学習したパターンをteam-usefulな形で追記できる構造(「Learned pattern format」)を持ち、個人の経験が組織知に昇格するプロセスが埋め込まれている。
- **不可逆操作へのゲート。** マージ、force-push、デプロイ、削除などの不可逆操作は必ず「人間の明示的なgo」または「rootのクリーン判定」を条件とし、逆に可逆な作業は「Never Block on the Human」原則に従い勝手に進める非対称なオートノミー設計。
- **スキップの可視化。** `architect skipped: <reason>`、`skip: <reason>`、`n/a: <reason>`という同型の記法が全プレイブックに登場し、「やらないと決めたこと」を消さずに記録する規律を徹底している。

## 5. agents（poteto-agent, comment-sicko）

### poteto-agent

`agents/poteto-agent.md`はごく短い定義で、`/poteto-mode`と「potetoスタイルで」という要求のルーティング先(routing target)として機能する。定義本文はほぼ1文で完結する。

> Read the `poteto-mode` skill's `SKILL.md` in full before doing any work, including its inline Principles index. Navigate to a leaf `principle-*` skill whenever you apply that principle.

フロントマターの`is_background: true`はこのエージェントが常にバックグラウンドで動くサブエージェントとして起動されることを示す。descriptionには「同じ会話の既存poteto-agentを再開せよ、兄弟エージェントを新規スポーンするな」という注意書きがあり、また「generalPurposeへの差し替えはこの読み込みをスキップしドリフトする」と明記されている。これはSKILL.mdの`Subagents`節にある「`subagent_type: "poteto-agent"`をコード委譲に使え」というルールの受け皿であり、poteto-modeという「振る舞いのラッパー」をサブエージェント側にも継承させる役割を持つ。実装内容自体は極小で、実質的には「SKILL.mdを読んでから動け」という単一の指示のみを持つ薄いプロキシ的エージェントである。

### comment-sicko

`agents/comment-sicko.md`は毛色が全く異なる。ペルソナ的なロールプレイをエージェント定義に埋め込んだ例で、コメント削除・ワークアラウンド告発に特化したエージェント。

> Yes... Ha ha ha... Yes! I hate comments. Feed me the parent scoped files or diff... Narration, banners, commented-out corpses, workaround sermons. I want them all.

これは`no-comments`スキットの実行体として使われるものと推測され、コメント削除の例外リスト（ライセンスヘッダー、外部依存に起因する非自明な挙動の説明、`prettier-ignore`、公開APIのdocコメント、issue/RFCリンク）を明確に定義した上で、それ以外を全て「meat（獲物）」として扱う攻撃的な口調で一貫させている。`eslint-disable`や`@ts-ignore`のようなlint抑制も、そのルールが実際にバグを捕まえるものであれば削除対象とし、該当シンボルを`MUST KILL`とマークする。`IMPORTANT`、`do not remove`のような強い言葉のコメントも「香りであって確信ではない(scent, not conviction)」として無条件には信用せず、`how`/`why`スキルで実際に裏を取ってから判断するとしている。最後に「Report only... I never write application code」と明言しており、コードは一切書かず、削除対象と`MUST KILL`フラグの報告のみを行う読み取り専用の監査エージェントとして設計されている。

ロールプレイ的な口調(「Ha ha ha... Yes!」)は一見奇妙だが、実務的には「コメント削除に対する躊躇を抑制する」ための強いペルソナ設定であり、通常のエージェントが遠慮しがちな大胆な削除を後押しする心理的なフレーミングとして機能していると考えられる。同時に、例外リストと`MUST KILL`のようなアウトプットフォーマットは明確で、ペルソナの過激さと出力の厳密さが両立している。

## 6. 所感

**設計として優れている点**

- 「原則を引用したら必ず具体的な決定に紐づけさせる」検証ループ（SKILL.md冒頭）は、原則集を作っただけで終わらせず実運用に効かせる仕組みとして非常に強い。単に「原則リストを読め」で終わらせず、返信フォーマットの中に検証手段を埋め込む発想は転用価値が高い。
- 「Inconclusive is not a pass」「self-reportを信用しない」「ledgerに証拠を残す」という一貫した哲学が全プレイブックを貫いており、単発のルールでなくシステム全体で「主張ではなく証拠」を強制している。エージェントの自己申告に頼らない設計は、長時間自律実行させるほど価値が上がる。
- `skip: <reason>` / `n/a: <reason>`という統一記法で「やらないと決めたこと」を可視化する規律。silent droppingを構造的に防止する、非常に安価で効果の高いパターン。
- Orchestrateプレイブックが自らのオーバーヘッドを実測値付きで開示し、「小さい仕事にはこのプレイブックを使うな」と自己抑制させている点。ツールを作った人間が自分のツールの適用範囲を正直に書く姿勢は珍しく、参考になる。
- 停止条件（predicate/ceiling/READY等）を事前に明文化させることで、「なんとなく完了した気になる」「プラトーで諦める」「述語を緩めて勝利宣言する」という典型的な失敗パターンを名指しで塞いでいる。

**自分たちのClaude Code運用に転用できそうな点**

- 「タスクの冒頭で適用プレイブック(または相当するチェックリスト)をtodoにverbatimでコピーさせ、スキップは`skip:`付きで残す」という運用は、Claude Code側でも `TodoWrite` を使い、CLAUDE.md やスキルの手順をそのままtodoに転記させる形で再現できそう。今の運用では手順の要約や省略が起きがちなので、この規律は有効だと思われる。
- Bug fixプレイブックの「再現→二分探索→修正→検証」の直線的因果連鎖と、各段階でランタイム証拠を要求する構造は、そのまま自分たちのバグ修正フローにも取り込める。特に「ユニットテストは不具合が無いことの証明にならない」という明言は、テストが通っただけで完了扱いにしがちな傾向への良い矯正になる。
- Bugbotトリアージのようなレビュー自動化ツールへの向き合い方（fix/dismiss/askの3分類＋学習パターンの team-useful な蓄積）は、自分たちがCode ReviewやCIの自動指摘にどう向き合うかのテンプレートとして使える。特に「dismissする場合は具体的な反証を書く」という規律は、ノイズを無視する際の説明責任を担保する良い仕組み。
- 「委譲したサブエージェントの`Done`という自己申告をそのまま信用しない、diffを自分でレビューして自分の言葉で要約する」という一貫した方針は、サブエージェント運用の基本姿勢として非常に真っ当で、そのまま自分たちの運用指針に取り込みたい。
