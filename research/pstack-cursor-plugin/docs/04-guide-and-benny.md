# pstack 利用ガイドと benny 自動化パック調査

対象: Cursor 用プラグイン pstack の `docs/guide/` 全11ファイル、`automations/benny/` 一式、`.cursor-plugin/plugin.json`、および `README.md`。

---

## 1. 想定ワークフロー全体像

pstack のガイドは「エージェントをマイクロマネジメントしない」ことを一貫した思想として、setup から夜間自律実行まで1本の線でつながる学習曲線を提示している。

| 章 | 要点 |
|---|---|
| 01-setup | `/add-plugin pstack` でインストール → `/setup-pstack` でモデルロールを設定（`~/.cursor/rules/pstack-models.mdc` に出力）→ 検証手段(verify skill)が無ければ `/create-verification-skill` を1回だけ提案 → 新規チャットで小さな実タスクを試す。 |
| 02-poteto-mode | `/poteto-mode` が全ての入口。ゴールを言うだけで22個のプレイブックの中から1つにマッチさせ、todoリストに手順をコピーする。「儀式ではなくゴールを言う」「話題が変わったら "new task" と言う」「並列作業には worktree を要求する」という3つの使い方の型を提示。 |
| 03-understand | コードを触る前に理解する4skill: `/how`（挙動説明）、`/why`（経緯調査、複数エビデンスソースを並列照会）、`/teach`（howとwhyを織り交ぜた説得的説明）、`/recall`（自分の過去コンテキストの再構築）。理解を飛ばして直すと「症状の対症療法」になりやすいと警告。 |
| 04-design | 実装前に形を決める: `/architect`（型・境界を先に固める、内部で`/how`→`/why`→`/arena`を実行）、`/arena`（N候補を並列生成し審査員が採点、良い所を接木）、`/swarm`（独立したスライス/レースをN並列でカバーし1レポートに集約）、`/interrogate`（複数モデルでdiffを崩しにいく）。設計にどれだけ投資すべきかの判断ラダーも提示。 |
| 05-build-and-clean | 各ビルド系プレイブック（bug fix/feature/refactoring/perf/hillclimb）は「観測した事実を言えばプレイブックが手順を補う」設計。`/tdd`は安いテストパスがある時だけ。TypeScript規約は`.ts`/`.tsx`に自動ロード。仕上げは`/deslop`（コード、cursor-team-kit提供）・`/unslop`（文章）・`/no-comments`（Comment Sickoによるコメント審査）で分業。 |
| 06-verify-and-ship | 「コンパイルが通った」は証拠にならない、が原則。完了条件を最初のプロンプトに書く。`/create-verification-skill`でプロジェクト固有の検証skillを生成し、`/maintain-verification-skill`で鮮度を保つ。PRは worktree から小さく順序立てたコミットで開く（Opening a PRプレイブック）。Babysitがコンフリクト→レビュー→CIの順にPRをmerge-readyまで育てるが、マージ自体はしない。Shippingが各PRを別のフレッシュエージェントで独立検証してから、緑が連続する範囲だけをGraphite merge-when-readyで着地させる。 |
| 07-overnight | 夜間契約(overnight contract)の型と、複数PR/プログラム規模へのスケールを提示（後述4章で詳述）。 |
| 08-principles | 21個の原則名を「エージェントを一言で誘導する語彙」として提示。 |
| 09-make-it-yours | `/automate-me`で自分の過去transcriptから自分専用のmodeスキルを生成、`/reflect`でセッションの教訓を蓄積、skill執筆は`create-skill`ビルトイン経由、`/technical-writing`で文書品質を担保、skill変更は`Eval`プレイブックでブラインドテストする。 |
| 10-recipes-and-pitfalls | コピペ用の定型プロンプト集と、繰り返される失敗パターン（後述4章で詳述）。 |

一貫する哲学は「計画スキルを敢えて用意しない」（README: "personally, i don't believe in planning. the best spec is code."）こと、そして「ゴール＋完了条件」を渡せば、あとはプレイブックとモデルルーティングが処理を代行するという分業設計である。

---

## 2. モデルルーティング戦略

pstack は「フロンティアモデルにはそれぞれ強みと弱みがある」という前提に立ち、役割ごとに最適なモデルを割り当てるマルチモデル戦略をデフォルトで内蔵している。

- **デフォルトパネル構成**: README に明記されている既定は **fable / sol / grok / opus 5** の4モデル。役割分担は以下の通り:
  - **sol** — 仕様が精密に決まったコード（precisely-specified code）
  - **grok** — 機械的で高速なコード（fast mechanical code）
  - **fable** — 文章と判断力が必要な作業（prose and judgment）
  - **opus 5** — （READMEのパネル列挙に含まれるがデフォルトの役割説明では上記3種に触れられていない。審査員(judge)やレビューパネルなど、より高い推論品質が必要な役割に充てられていると推測される）

- **設定の仕組み**: `/setup-pstack` が利用可能なモデルを検出し、各ロール（code delegates, judgment, review panels）に対してユーザーに質問し、`~/.cursor/rules/pstack-models.mdc` という「常時適用ルール」を書き出す。すべてのpstackスキルがこのルールを読む。ロールに設定行が無ければスキル側のデフォルトにフォールバックする、という差分オーバーライド方式。

- **`auto` / `inherit-parent`**: ロールに `auto` または `inherit-parent` を指定すると、サブエージェントの `model` フィールド自体を省略し、親チャットのモデルをそのまま継承する。**これはモデルスラッグではない**という注意が01-setupと10-recipes-and-pitfallsの両方で強調されている（「`auto`をモデルスラッグとして扱う」ことは明示的なpitfallとして列挙）。

- **パネル役割でのモデル多様性の意図**:
  - `/arena`: N個の候補を生成させ、可能な場合は「別のモデルファミリー」の読み取り専用審査員(judge)が採点する。モデルの多様性そのものが目的ではなく、決定の質を上げる手段。
  - `/interrogate`: 同じdiffを複数の異なるモデルファミリーのレビュアーに送る。「モデルの多様性こそが本質」で、複数モデルが独立に指摘した内容は高信頼シグナルとして扱われる。
  - `/reflect`: transcriptを3つの並列レビュアーに送り、シンセサイザーが提案を分類する。
  - `/show-me-your-work`: 夜間実行の要約を返す前に、書いた本人とは別のモデルファミリーのレビュアーにtranscriptと決定ログを読ませてから返す。
  - Shippingプレイブック: 「変更を審査するエージェントは、それを書いたエージェントであってはならない」という原則を明記し、各PRを新しい別エージェントで検証する。

- **swarm workers**: `/setup-pstack` は `/swarm` のデフォルトワーカーモデルも設定する。レースの各アームが個別にモデルを指定しない限り、この既定値が使われる。

つまりpstackのモデル戦略は、単一モデルに全工程を任せるのではなく、(1) タスクの性質（精密なコード/機械的なコード/文章と判断）ごとにモデルを割り振り、(2) 検証・審査の局面では意図的に「書いた本人と異なるモデル」を立てることで、自己採点バイアスとモデル固有の盲点を相殺する設計だと分析できる。

---

## 3. 夜間・長時間自律実行（07-overnight）

### `/loop` との組み合わせ方

`/loop` は Cursor 組み込みの「起床メカニズム」であり、pstack のスキルではない。Autonomous run プレイブックがこれを使って、イベントまたはハートビートごとに完了条件を再チェックする。

夜間契約(overnight contract)の型（ガイドの例文より）:

```text
im going to bed. migrate every caller to the new parser in a fresh worktree off <base>.
done means zero old callers, all parser fixtures pass, old api deleted.
keep a decision log. don't ask me before committing.
/loop until done. if you're truly stuck after a few hours, stop and write up why.
```

この一文の各要素が持つ機能:
- 「im going to bed」= セッションオーバーライド。エージェントは以後質問をやめて進み続ける。
- 「done means...」= 完了条件をチェック可能な形にする。
- 「fresh worktree off `<base>`」= 他の作業と衝突しない隔離。
- 「don't ask me before committing」= コミット許可を先回りで与える。
- `/loop` = イベント/ハートビートで完了条件を再チェックする駆動部。
- エスケープハッチ（「本当に詰まったら止めて理由を書く」）= 8時間の創造的なゴール再解釈を防ぐ安全弁。

ループの1周期は「完了条件チェック → 最小限の正当化された変更 → 実アーティファクトでの検証 → 進捗があればコミット、なければ破棄 → 決定ログに1行記録 → 最初に戻る」というサイクル。進捗のない変更は握りつぶさず破棄され、プラトー（停滞）はピボットのきっかけであって停止の理由にはならず、完了条件は静かに緩められない。

### 安全装置

- **決定ログ**: `/show-me-your-work` が `decisions.tsv`（複数の実行が同じディレクトリを共有する場合は `.audit/<task-slug>.tsv`）に、時刻・フェーズ・決定・理由・エビデンスへのポインタ・結果を記録する。デフォルトはローカル保存のみで、レビュアーが追跡できる規模の作業のときだけコミットする。
- **朝のレビュー**: `/show-me-your-work catch me up on what you did last night` と聞くと、要約を返す前に別モデルファミリーのレビュアーがログとtranscriptを読み、返信の末尾に「Attention」セクション（注視すべき点）を付ける。ユーザーはそこから読むことで、ログ全体の再読を避けられる。
- **worktreeの隔離**: 夜間実行は`/figure-it-out`経由でルーティングされ、フェーズ設計と決定ログの配線を事前に行う。
- **キュー/プログラム規模へのスケール**: 単一タスクを超える夜には3段階のプレイブックが用意される。
  - **Autopilot-full**: 独立した複数PRのキューをマージまで自動化。各PRに1人のオーナーエージェントがつき、オーナー自身の判定ではマージされない。フレッシュな検証者の群れ(swarm)が全てのmerge-readyヘッドを検査し、クリーンな判定だけがマージを許可する。
  - **Autopilot-stack**: 同じオーナーループを走らせるが何もshipしない。1本の線形Graphiteスタックに全リンクの検証者判定を付けて起きたら自分でレビュー・着地する。変更同士が結合している場合や自分の目で見てからマージしたい場合向け。
  - **Orchestrate**: 単一エージェントのセッションを超えるプログラム（複数日、多数のスタックPR、サブエージェントの艦隊）向けの常設コーディネーターチャット。コーディネーター自身はコードを書かず、ブリーフを作成しサブエージェントの完了物を集約し、未マージの最下位PRを常にグリーンに保つ。1エージェントで終わる作業であればこのプレイブック自体が夜間契約に差し戻す設計になっている。

### 失敗パターン

- **「期間」は完了条件にならない**: "work on this for 4 hours" はチェックできる対象を与えない。`/loop`には合否判定可能な述語(predicate)を渡すべき、という明示的な pitfall。
- **並列エージェントが1つのworktreeを共有する**: 互いのファイルを上書きし、diffが考古学的に読み解く対象になる。
- **`auto`をモデルスラッグと誤解する**: 実際は「モデルフィールドを省略して親チャットのモデルを継承する」という意味。
- **グリーンビルドを成功の証拠として報告する**: ビルドが通ることはコンパイルできることの証明にしかならない。実コマンド・実フロー・実際に保存された値・プロファイルでの証拠を要求すべき。

---

## 4. レシピと落とし穴（10-recipes-and-pitfalls）

### 具体的なレシピ（コピペ用の定型プロンプト）

1. **未知のサブシステムを理解する**: `use /how first to understand how this initialization works. then use /why to figure out why it broke recently.`（メカニクスが先、履歴が後）
2. **設計のセカンドオピニオンを得る**: `ask /arena for a second opinion on this thread and our approach`（自分の設計案も候補の1つとして扱わせる。コミット前の安い保険）
3. **独立したスライスを並列チェック**: `/swarm check every package under packages/ against its check.sh. one worker per package. one report.`
4. **ブランチを懐疑的にレビュー**: `/interrogate the whole branch, but skeptically. don't change anything yet. no nitpicks unless it's an actual bug or regression in behavior.`（「まだ変更しないで」「実際のバグ/回帰でなければ細かい指摘は不要」という修飾語が実際に効く）
5. **失敗テストを通してバグを直す**: `/poteto-mode repro the duplicate write first. if there's a cheap test path, /tdd it. then fix and rerun.`（「安いテストパスがあれば」という条件が重要。ブリトルなモックを強制するくらいなら実コマンドの方が強い証拠）
6. **不在時も実行を誠実に保つ**: 夜間契約の短縮版（会話に既にタスクと完了条件がある場合に有効）
7. **ドリフトしている実行を軌道修正する**: 原則名を1行で使う（例: `apply prove it works. show me the real output, not the build log.`）
8. **返信を平易な言葉で得る**: `/bro` の一言で技術的すぎる返信を人間の言葉に言い換えさせる

### 明示された落とし穴（pitfalls）

- **プロンプトでskillを列挙すること**: 「use /how then /architect then /arena」のようにプレイブックが既に並べている手順を並べ直すと、順序を変えたり手順を落としたりする。ゴールと制約を述べ、デフォルトを上書きしたい時だけskill名を出す。
- **曖昧な完了条件**: 「もっと良くして」は`/loop`にチェック可能な材料を与えない。
- **1つのworktreeでの並列エージェント**: 互いのファイルを上書きする。「own worktree per attempt」と言えば隔離は無料で手に入る。
- **`/arena`をカバレッジ目的で使うこと**: `/arena`は1つの設計/コードのブリーフを繰り返して接木するもの。スライス分割やレースのアグリゲートは`/swarm`の仕事。
- **すべてのレビューコメントを受け入れること**: bot・人間ともに本物の指摘とノイズを同じリストに混ぜてくる。`/interrogate`は act-on / dismissed に理由付きで仕分けする。
- **`auto`をモデルスラッグとして扱うこと**（3章と重複記載）
- **グリーンビルドだけで成功報告すること**（3章と重複記載）
- **`SKILL.md`を手書きすること**: Authoring or modifying a skillプレイブック経由にすることで検証とレビューが働く。

---

## 5. benny 自動化パック

### 何を自動化するのか

benny は pstack にバンドルされた「休眠状態」の自動化パックで、Slack上のissueレポートに対して2つのCursor Automationが連携して動く。スラッシュスキルとしては登録されず、`.cursor/automations/benny/` にファイルとしてコピーされてから、Cursor Automationのライブプロンプトが直接読みに行く方式（`disable-model-invocation: true` が両SKILL.mdに設定されている）。

- **automation 1（triage-issue-reports）**: 設定済みのSlackチャンネルに新しいトップレベル報告が来たら起動。スレッドと添付を読み、bug/performance/feature request/question or feedback/rerouteに分類し、原因を辿ってから経路決定する。issueトラッカーで重複を検索し、確信度の高い重複は既存issueを更新、明確な新規バグのみチケットを作成する。元スレッドに `[benny:bug]` / `[benny:performance]` / `[benny:other]` のいずれか1つのマーカーを付けて1回だけ返信する。ソースチャンネルにルートメッセージを投稿することは絶対にない。
- **automation 2（reproduce-and-fix-issues）**: 同じ新規報告をトリガーに開始するが、automation 1が付けた信頼できるトリアージマーカーを待つ。既存のPRやマージ済みコミットが既にissueを解決していそうな場合は「検証モード」に切り替え、競合する修正は作らない。確認済みバグ/性能問題であれば、設定済みのcontrol adapterとfeature mapを使い、実UIを通して症状を2回再現し、スクリーンショット・動画・読み取り専用の状態クロスチェックを取る。確定した再現の後、1回だけ範囲を絞った根本原因修正を試み、安いテストパスがあれば`/tdd`を使い、影響範囲(blast radius)をスモークテストし、before/afterの証拠が揃ってからドラフトPRを開く。

両automationとも「ソースチャンネル・ルートスレッド座標は不変」「コーディネーターだけがSlackに投稿できる」「委譲されたワーカーは読み取り専用でSlack書き込み権限を一切持たない」という共通のハードルールを持つ。

### 各スキルの詳細

- **`triage-issue-reports/SKILL.md`**: ソース座標の固定 → レポート全文と添付の読み込み → `/how`・`/why`を使った原因トレース → 分類（bug/performance/feature request/question or feedback/reroute） → routing.mapによる経路決定（オーナーへのpingはデフォルトでオフ、厳格な4条件を満たす場合のみ許可） → issue-tracker adapter経由の重複チェックと作成判断 → 1件だけの検証済みverdict投稿 → 設定された猶予時間だけフォローアップを監視、という10ステップ構成。「チケットを作るくらいなら作らない方がいい」「推測でチケットを作らない」という保守的な原則が随所にある。

- **`reproduce-and-fix-issues/SKILL.md`**: 座標固定 → トリアージ契約(trusted marker)を待つ → 所有権/既存修正物のゲート判定（人が明確に修正を宣言していれば手を引く、既存PRやコミットがあれば`verify-existing-fix.md`のモードに切り替え） → オプションのoperations threadでのステータス投稿 → control adapterとfeature mapのロードと7要件チェック → レポート精査 → 実UIでの再現（2回、独立した状態でのリセットを含む） → エビデンス収集とメディアレビュー（read-only reviewerが「discriminating broken stateが写っているか」を判定） → 再現結果の報告 → 既存修正の検証（該当する場合） → 修正実施の資格判定（7条件すべてを満たす場合のみ） → root-causeと実装（コーディネーターが全てのSlack投稿・最終diffレビュー・コミット・PRを所有、狭く絞ったコード編集のみサブエージェントに委譲可） → before/afterの証明 → ドラフトPRのオープン → フォローアップとクリーンアップ、という15ステップ構成。

- **`setup-benny/SKILL.md`**: パックの丸ごとコピー（`.cursor/automations/benny/`への同期、宛先専用ファイルの保護、ユーザー所有設定の非上書き） → pstackを対象リポジトリの`.cursor/settings.json`で有効化 → 共有pstackスキル（how, why, tdd, unslop, および5つの原則スキル）がプロジェクトスコープで解決することを検証 → 設定例のアダプト（configuration.example.yaml, feature-map.example.mdをパック外にコピー） → 必須選択項目の確認（Slackチャンネル、リポジトリ、トラッカー、control skill、feature-map等） → 統合能力のチェック → routing mapの準備 → control adapterの検証 → ライブautomationの準備（新規作成時はビルトイン`/automate`を1つずつ完了させる、既存automationの更新時は`/automate`を使わずチェックリストで手動編集させる） → スレッド安全性テスト（7項目）、という8段階のセットアップ手順。「secretはYAMLに書かない」「plugin cache pathを直接参照しない」「既存automationを重複作成しない」といった防御的な指示が多数含まれる。

### templates のプロンプト設計

`triage-automation-prompt.md` と `reproduce-automation-prompt.md` は、ビルトインの `/automate` にintentを伝えるための「二次的な内部ソース資料」として位置づけられている。両テンプレートとも共通の構造を持つ:

1. 対象のSKILL.mdを「このrunで読んで従え」と直接指示する
2. 設定ソースへのパスをプレースホルダー(`{{BENNY_CONFIG_PATH}}`)で参照し、コミット済みでない場合はパラフレーズに切り替える
3. トリガーのJSON構造（`source_channel_id`, `message_ts`, `thread_ts`）を明示
4. ソース座標が不変であること、欠落時は投稿せず停止することを明記
5. コーディネーターだけがSlackポスターであり、子プロンプトは`SendSlackMessage`等のSlack書き込みAPIを明示的に禁止されること
6. ソースチャンネルへのルート投稿を絶対にしないこと

`configuration.example.yaml` はSlack/repository/tracker/routing/control/verdict_markers/status_emoji/budgets/modelsの9セクションで構成され、全てプレースホルダー値。特に `budgets` セクション（poll_seconds, verdict_wait_minutes, repro_minutes, fix_minutesなど）が細かく時間制約を切っている点、`allow_source_root_posts: false` や `allow_worker_slack_writes: false` のような「フェイルクローズ」を強制するブール値が並ぶ点が特徴的。

### FOR_AGENTS.md の意図

`FOR_AGENTS.md` は人間がCursorに指示する「セットアップの入口ファイル」として設計されている。冒頭で「私が実現したい自動化」を一人称の意図(intent)として記述し、triage/reproduceそれぞれのtrigger・behavior・tracker・tools・outcome・boundaryを明文化した後、「for the agent」セクションでエージェント向けの実行手順（ステップ1〜7）に切り替わる。

この構成が示す意図は次の通り:
- **人間の意図とエージェントの手順を分離する**ことで、人間は「何をしたいか」だけを書き、機械的な処理（マージ、設定検証、`/automate`呼び出し）はエージェント手順として固定化している。
- **discovered benny slash skillを探すな**という明示的な禁止（`SKILL.md`はスラッシュスキルとして登録されていないため誤発見を防ぐ）。
- **既存ファイルの上書き禁止と差分レビューの要求**により、パックの再配布（アップデート）が既存のユーザー設定を破壊しないことを保証している。
- **secretやfeature map、routing mapをパック外に置く**という指示が繰り返し出てくる。これはパックの「リフレッシュ（再コピー）」がユーザーの秘匿情報や環境固有設定を巻き込んで消さないようにするための設計。
- **fail closed**の原則が全体を貫く。チャンネル座標、トラッカーアクセス、control adapter、feature mapのいずれかが欠落・不確実なら、両automationとも停止する。

---

## 6. プラグインパッケージング

`.cursor-plugin/plugin.json` はpstackプラグインのメタデータを定義する単一ファイルで、以下のフィールドを持つ:

```json
{
  "name": "pstack",
  "displayName": "pstack",
  "version": "0.14.5",
  "description": "...",
  "author": { "name": "Lauren Tan" },
  "homepage": "https://github.com/cursor/plugins/tree/main/pstack",
  "repository": "https://github.com/cursor/plugins",
  "license": "MIT",
  "keywords": [...],
  "category": "developer-tools",
  "tags": [...],
  "skills": "./skills/",
  "agents": "./agents/"
}
```

わかる構造上の特徴:

- Cursorプラグインは `cursor/plugins` という中央リポジトリの1ディレクトリとして配布される（`repository`フィールドがモノレポを指し、`homepage`がそのサブディレクトリを指す）。単体の独立リポジトリではなくnpmライクなモノレポ配布形式。
- `skills` と `agents` は相対パスでディレクトリを指すだけのシンプルな宣言。実体は `skills/<skill-name>/SKILL.md`（frontmatterに`name`, `description`, 場合によっては`disable-model-invocation: true`を持つMarkdown）と `agents/<agent-name>.md` という規約に依存しており、`plugin.json`自体はスキル一覧を列挙しない（ディレクトリを丸ごとスキャンする設計と推測される）。
- `category`/`tags`/`keywords`はCursorのプラグインマーケットプレイス的なUIでの検索・分類用メタデータと見られる。
- `automations/benny/` はこの `skills`/`agents` 宣言の対象外に意図的に置かれている。benny内の`SKILL.md`群は`disable-model-invocation: true`を持ち、プラグインマニフェストの`skills`ディレクトリにも含まれない（README・FOR_AGENTS.md・setup-benny/SKILL.mdすべてで「plugin cache pathやスラッシュスキル発見に頼るな、対象リポジトリにコピーされた実ファイルを直接読め」と繰り返し警告）。これは、ユーザー固有の設定（Slackチャンネル、トラッカー、feature map）を含む自動化パックを、汎用プラグインの配布ライフサイクル（アップデートで上書きされる）から意図的に切り離すための設計である。
- バージョニングはセマンティックバージョン単体のフィールド(`version`)のみで、pstack内部のスキル群がバージョンごとにどう変わるかを追跡する仕組みはplugin.json自体には見当たらない（skillsディレクトリ内で管理されていると推測される）。

---

## 7. 所感

ガイドの教育設計として優れている点をいくつか挙げる。

1. **「1つだけ覚えるなら」で入口を1行に圧縮している**: README冒頭でも guide/README.md でも、まず動くコピペ用の1行プロンプトを見せてから詳細に降りていく構成になっている。読者が読み終える前に手を動かせる。

2. **各章が「何を書くか」ではなく「何を言えばよいか」を教える**: ガイド全体を通じて、スキル名の羅列ではなく自然文のプロンプト例（しかも意図的にカジュアルな文体）を大量に提示している。「エンジニアが同僚に頼む言葉」をそのまま使えることを実演しており、抽象的なベストプラクティス集ではなく模倣可能な実例集になっている。

3. **各章末に必ず1つのpitfallと次章へのリンクを置く構成**: 学んだことをすぐ誤用しそうな地点で釘を刺し、読み進める動線を切らさない。特に02章の「skillを列挙するな」、03章の「/howを飛ばすな」、05章の「クリーンアップは任意の磨き上げではない」など、初心者が最初につまずくポイントを先回りしている。

4. **「証拠」という一貫した評価軸を全章に通している**: 06章の"It compiles" is not evidence.から始まり、07章の決定ログ、10章の「グリーンビルドでの成功報告」pitfallまで、「検証」という概念を1つの原則（Prove It Works）に集約し、それを異なる文脈（PR・夜間実行・レシピ）で繰り返し具体化している。単発の教訓の寄せ集めではなく、1つの哲学を多面的に反復する設計になっている。

5. **段階的な信頼の委譲という物語構造**: setup → 理解 → 設計 → 実装 → 検証 → 夜間自律という並びは、単なる機能カタログの順ではなく「小さなタスクを1つ手伝わせて信頼する→やがて一晩预けられるようになる」という心理的な弧を描いている。07章冒頭の「これが今までの全てへの見返りだ("This is the payoff for everything before it.")」という一文がその設計意図を明言しており、ガイド全体が逆算的に構成されていることがわかる。

一方で、benny自動化パックのドキュメント（FOR_AGENTS.md、SKILL.md群）はガイドとは対照的に非常に形式的・防御的な文体で書かれている。これはガイドが「人間への説得」を目的とするのに対し、bennyのファイル群は「エージェントへの厳密な実行指示」であり、Slackへの誤投稿やsecret漏洩といった実害を防ぐための契約書的な設計であるためだと考えられる。両者の文体の違い自体が、pstackが「人間向けの教育コンテンツ」と「エージェント向けの実行契約」を明確に書き分けているという設計思想を裏付けている。
