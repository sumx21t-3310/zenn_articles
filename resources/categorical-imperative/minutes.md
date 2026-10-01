# 議事録: コーディングエージェントへの定言命法の活用方法と、その可能性

この議事録は、report.md の結論がどの議論から導かれたかを確かめるための記録です。逐語録ではなく、誰が何を主張し、どこで意見が分かれ、どの反論で結論が変わったかを追跡できる形にまとめています。

討議は 2026年10月1日に、Claude Fable 5.1 をホストとして行いました。6つの役割のエージェントを起動し、次の順で進めました。

1. 独立分析。各役が他の回答を見ずに回答する
2. 論点抽出。ホストが回答を比較し、対立点を優先して10の論点を出す
3. 相互反論。各役に他役の主張と論点を提示し、同意、反対、見落とし、変更、残る懸念を書かせる
4. ケーススタディ。ケースA〜Fの判定をホストが統合する
5. 最小実装案。合意を反映した v1 の仕様を設計する
6. 将来拡張。採用しないものの理由を含めて評価する

元の提案書(『コーディングエージェントへの定言命法の活用案』)と討議のブリーフは、全員が同じものを読みました。多数決で決めず、少数意見でも反例や設計上の問題を示すものは残しています。


## 参加エージェント

| 役割 | 担当 | 第1段階の立場(一文) |
|---|---|---|
| Agent Architecture Reviewer | 責務分割、Policy Gate、権限、承認、状態管理、失敗時の挙動 | 審査は「diff から Maxim を導出して既存ポリシーと照合する」として実装し、Gate は実行権限に接続する |
| Formal Methods Reviewer | Maxim の形式化、不変条件、前提条件、Lean の適用範囲 | 提案の式は定言命法ではなく不変条件の保存であり、最大の問題は前提条件の真偽を誰が判定するかである |
| Adversarial Reviewer | 反例、抜け道、Reward Hacking | 安全性は Maxim や LLM Reviewer からは生まれず、観測・事後 diff・書込権限の外の検証の3つからしか生まれない |
| Software Design Reviewer | 設計原則との境界、API 境界、依存方向、YAGNI、テスト境界 | 普遍化テストが効くのは「検証する仕組みの検出力を下げる行動」の検出だけであり、設計判断には何も言えない |
| Agent Learning Reviewer | RL、Preference Learning、評価データ | +1/-1 の報酬は説明文を最適化する policy を作るので RL は見送り、Context/Action はラベル付けの型として使う |
| Practical Coding Agent Reviewer | 既存エージェントへの最小構成の導入 | 効くのは diff からの種別判定、Approval と PolicyChange の分離、クローズドクエスチョンの3点であり、普遍化の推論を LLM に文章で行わせる部分は弱い |
| ホスト(Claude Fable 5.1) | 論点抽出、討議の進行、成果物の作成 | 結論を先に決めず、対立点を優先して討議する |

別モデルによる独立意見として、Codex CLI にも同じブリーフを渡しました。しかし、CLI の版と設定モデルの不一致(`gpt-6-astra` は CLI 0.152.1より新しい版が必要、`gpt-5.4` は ChatGPT アカウントでは使用不可)により、2回とも回答を得られませんでした。

## 初期意見(第1段階、独立分析)

各役は他の回答を見ずに回答しました。要点のみ記録します。全文は討議の作業ファイル(`stage1_<role>.md`)にあります。

### Agent Architecture Reviewer

- 有効な部分: タスク成功と方法の許容の分離、RequiresApproval と RequiresPolicyChange の分離、Invariant の明示、Revise の選択肢つきのクローズドクエスチョン
- 設計上の問題: 審査の単位(tool call か作業単位か)が未定義、Adopt をエージェントの自律実行に直結、Reviewer の入力が自己申告、5役割構成に根拠がない、状態の置き場と失敗時の挙動が未定義
- 改善案: 二層 Gate(tool call 単位の決定的 Gate と作業単位の diff 分類)、diff から Maxim を導出、capability による権限、例外を `.agents/exceptions/*.yaml` に Scope と Lifetime 付きで記録、同じ例外が N 回続いたら RequiresPolicyChange へ昇格、Postcondition は宣言と実態の一致・Invariant の機械検査・例外の範囲内を見る
- 最大のリスク: 審査が形式だけになり安全性の錯覚を与えること、承認疲れ

### Formal Methods Reviewer

- 有効な部分: 禁止事項の列挙(closed world)から不変条件と遷移(open world)への置き換え、Maxim の構造化、判定の分離は代数的データ型で表せる
- 設計上の問題: 前提条件の真偽判定(`specification_status` は意図の関数)、翻訳層での Action の言い換え、述語の粒度、Universalization と Invariant preservation の差は「集合間の対応としての不変条件」と「vacuous の検出」の2点に還元できる
- 改善案: 前提条件を Observable と Declared に型で分け Declared に provenance を必須にする、Maxim を観測から作る、不変条件が main で成り立つかを CI で検査する、Lean は policy のメタ検証に限定、反例探索は Alloy
- 証明できないもの: Apply と git 操作の一致、テストと仕様の対応の現実との一致、「このテストは間違っている」、assert の緩和が検証を弱めたこと

### Adversarial Reviewer

- 成立に必要な前提: P1 述語の値は観測関数が埋める、P2 Reviewer と Gate は diff を入力とする、P3 Gate は実行権限に接続、P4 検証は Agent の書込権限の外、P5 Invariant の大半は機械検証できる、P6 例外には機械判定できる解除条件がある
- 攻撃13件: Predicate Narrowing、前提の自己申告、Test Hollowing(7つの方法)、CI 隠蔽、仕様の再解釈、例外の永続化、承認誘導、Reviewer 攻略、自己審査の相関、Gate 後の副作用、Verifier 改変、観測汚染、RL Goodhart
- 塞げると判定した攻撃はすべて機械検証への置き換えで塞ぎ、塞げないと判定した攻撃(承認誘導、Reviewer の文章判断の攻略、自己審査の相関、RL Goodhart)はすべて LLM の文章判断を信頼する部分に由来する

### Software Design Reviewer

- 境界の基準: 「その Maxim を普遍化したとき、正しさを検証する仕組みの検出力が落ちるか」。落ちるなら定言命法の対象、落ちないなら設計レビューの対象
- 不変条件は「規約の内容」ではなく「規約を検証する仕組みの保全」として書く。「依存方向を守る」は設計規約であり不変条件から外す
- 普遍化可能だが優劣は別の例: Repository Pattern と DbContext の直接利用、DI コンテナと手動注入、例外と Result 型、ORM と生 SQL。モノリスとマイクロサービスは Maxim の形にすらならない
- 誤って設計判断を禁止する例: 実装詳細テストの削除、アーキテクチャテストに引っかかる設計変更、`public` すべてを公開契約とみなす過剰判定
- 最小構成: diff ベースの弱体化検出、branch protection、LLM による説明文生成。5役割パイプライン、Lean、RL はシナリオが付いていないので YAGNI に反する

### Agent Learning Reviewer

- +1/-1 の報酬は不十分。policy reward だけなら「何もしない」が最適、加算なら λ 次第で「penalty を払ってテストを消す」が成立。Local-only を中立にすると承認要求の乱発が学習される
- Negative example を集めた学習データの実体は (Context, Action) の禁止リスト。一般化が起きるかは hold-out 評価で測るまで不明
- Purpose は観測できないので学習スキーマから外す。承認済み例外は学習データから除外する。学習時の Invariant 集合はサンプルごとに変える
- 報酬は3信号を gate で組み合わせる: hidden test による task reward × 検証可能な Invariant 違反なし + Faithfulness(宣言と再構成の一致) + 承認要求の要否
- `assert add(2,2) == 5` のような expected 値の上書きは repository state を見る判定器でも検出できない
- RL は見送り、diff から再構成した Action のラベル付き評価データセットを先に作る

### Practical Coding Agent Reviewer

- 判定の単位は tool call ではなく diff。post-action(Stop hook)を先に入れ、pre-action は決定的な短いリストに限る
- Context の各項目に出所を割り当てる: `artifact_kind` は path、`action_kind` は diff、`specification_status` は Issue のラベル。出所のない項目は使わない
- 判定表 `(artifact_kind, action_kind, specification_status) → verdict` を10行程度置き、表にない組み合わせは RequiresApproval に倒す
- Plan 段階で宣言させるのは `declared_action_kinds` だけ。実行後の `observed_action_kinds` との不一致を RequiresApproval に倒す
- LLM Reviewer は1つで、無人実行を始める時点に足す。入力は diff・表の判定・宣言した種別・AGENTS.md の不変条件節に限り、出力は固定スキーマの JSON。Reviewer に許可を出させない
- 既存の仕組み(`permissions.deny/ask`、hooks、CI の required check、branch protection、CODEOWNERS)で大半を代替でき、新規は種別判定スクリプト 100〜200 行と Reviewer のプロンプトだけ。層0〜3 で2〜3日
- AGENTS.md に「一般化して審査せよ」という思考手順を書いても結果は変わらない(経験則、未計測)

## 主要論点

ホストが第1段階の回答を比較し、次の10の論点を抽出しました(第2段階)。対立している点を優先し、多数決では決めていません。

| 番号 | 論点 | 対立の形 |
|---|---|---|
| 1 | 語彙を閉じると「禁止事項の列挙」に戻るのではないか。定言命法の名前を残す価値はあるか | Learning「Negative example の実体は禁止リスト」と Practical「名前はコストにも効果にも寄与しない」に対し、Formal「残るのは2つの設計指針」と Design「設計時の思考法」 |
| 2 | 「依存方向を守る」は不変条件か、設計規約か | Formal は `layerRule` を不変条件の述語として書き、Design は設計規約として外すべきと主張 |
| 3 | 仕様変更の「明示」はどこまで厳しくするか | Practical と Design「ラベルか ADR のみ。チャットは不可」に対し、個人開発の運用との両立 |
| 4 | テストの弱体化をどの検査で捕まえるか | Practical の静的な数え上げ、Adversarial の base commit 実行、Design のマーカーと mutation score、Learning の反例(`assert add(2,2) == 5`) |
| 5 | 過剰検出と見逃しのどちらに倒すか | Adversarial「ask を広く」と Practical「表にない組は RequiresApproval」に対し、Design「過剰検出が承認を無意味にする」 |
| 6 | Lean に近い将来の価値があるか | Formal「policy のメタ検証に価値」に対し、Design「既存ツールで足り、状態モデルのコストが見合うか不明」 |
| 7 | LLM Reviewer をいつ、何のために足すか | Adversarial「攻撃面」、Practical「無人実行の時点で1つ」、Architecture「取りこぼしが出てから」、Design「説明文の生成に限定」 |
| 8 | Postcondition の「宣言と観測の一致」は十分に強いか | Adversarial の攻撃10(Bash 1回で Scope 外を変える)と攻撃12(観測汚染)。Stop hook と CI のどちらが本体か |
| 9 | 複数 LLM の価値 | 別モデルの一致は独立した裏付けか、同じ盲点か。Codex の参加が失敗したことの含意 |
| 10 | Maxim を実行前に「宣言」させる価値 | Practical「宣言と観測の突き合わせ」に対し、Adversarial「通る表現を学ぶ機会」、省略の動機 |

## 意見が一致した点

第1段階の時点で、6役すべてが一致した、または明確な反対なく一致した点です。

- 「目的を達成したか」と「方法を認めてよいか」を分ける問題設定は正しく、導入する価値がある
- 提案書の `∀ state, Precondition ∧ Invariant → Invariant(Apply(action, state))` は定言命法の形式化ではなく、Hoare 論理や inductive invariant と同じ不変条件の保存である。実装上は Invariant-preserving policy として扱う
- Maxim の Context と Action は、エージェントの自己申告ではなく diff、CI 結果、path 規則、ラベルからの観測で作る。語彙は閉じる。Purpose は審査に使わない
- Policy Gate は実行権限(hooks の deny と ask、CI の required check、branch protection)に接続する。助言にとどめるなら導入しない方がよい
- RequiresApproval と RequiresPolicyChange を分け、後者はチャットで承認できない
- 決定的な層を先に入れ、LLM Reviewer は最後に、入力を diff に限定して1つだけ足す。最終判定は規則が出し、LLM に許可を出させない
- 提案書の5役割パイプライン(Maxim Generator、Universalization Verifier、Counterexample Generator、Executor)に根拠はない
- 同一 LLM の Planner と Reviewer の分離に、説得への独立性はない。複数 LLM の効果は限定的で、測定されていない
- 単純な `+1 / -1` の RL は見送る。Reviewer の判定を報酬にしない
- Lean は個々の行動審査には使わない
- 例外には期限または機械判定できる解除条件が必須で、リポジトリに記録し、期限切れを CI が検出する
- Postcondition は「宣言した行動と diff から観測した行動の一致」「Invariant の機械検査」「例外の範囲内」を見る。「テストが通ったか」は別工程
- 最大のリスクは「審査がある」という誤認で人間のレビューが薄くなること、および承認疲れ

## 意見が分かれた点

第3段階の相互反論を経ても残った対立点です。

- ケースDの設定ファイル単位の lint 無効化。RequiresPolicyChange(Adversarial、Learning、Architecture。リポジトリ全体の規約の変更であり一時例外の性質ではない)と RequiresApproval(Formal、Design、Practical。検証器の設定変更として ask)。行単位の抑制が RequiresApproval である点は一致
- ケースAの中核(実装変更で失敗したテストの削除、ラベルなし)の判定名。RequiresPolicyChange(Formal、Architecture、Practical。ラベルが付けばケースFに変わるので Forbidden ではない)と Forbidden(Learning、Adversarial、Design。削除テストの参照シンボルが残存し、ラベルがなければ決定的な層で止める)。実質は同じで、人間がラベルを付けるまで止まり、エージェントは削除を選択肢に出さない
- 検出力は「測れる」(Design)か「コストに応じた精度で近似できる」(Formal。欠陥の空間は無限であり、包含は決定不能)か。記事では後者の表現を採る
- Reviewer の盲点の相関。Adversarial は「独立性はない」から「圧力の分離による差はあり得るが未測定」に軟化し、Learning は測定の実験を設計した。未測定のまま
- 評価データセットは YAGNI か。Design は RL を YAGNI としたが、Learning は評価データセットは討議の主張の多くを測る唯一の手段であり YAGNI ではないと反論し、Architecture も同意した。Design は明確には反対していない

## 主な反論

第3段階で、各役割が他の役割の主張に対して行った反論のうち、結論を変えたものを紹介します。

- Design から Formal へ: `layerRule`(依存方向)を不変条件にすると、DbContext を Application 層で直接使う設計判断が Forbidden になる。不変条件は「依存方向を検証するアーキテクチャテストを無効化しない」と書くべきである。Formal はこれを受け入れ、不変条件を1つのスキーマに縮約した
- Design から Adversarial と Practical へ: 保護パスを広く取り、`Edit(./tests/**)` を一律 ask にすると、承認疲れによって承認が無意味になり、審査なしより悪くなる。Adversarial はこれを撤回し、承認疲れを使う攻撃14(承認の希釈: 同一文面のダイアログを11回出し、12回目に assert 緩和を混ぜる)を自ら書いた。Practical も `ask` リストから `Edit(./tests/**)` を外した
- Adversarial から全員へ: 攻撃10(Bash 1回で Scope 外を変える)、攻撃12(観測汚染: 先に `pytest.ini` を変える)、攻撃11(検証スクリプトの改変)を提示した。Architecture、Practical、Formal は、判定の本体を CI の required check に移し、観測を git object と base の設定に対して行い、テスト実行設定と hook スクリプトを保護パスに入れる形で受け入れた
- Adversarial と Learning から Practical へ: 静的な数え上げは、assert の緩和と fixture の差し替えを取りこぼす。Practical は、交差実行(変更後のテストを base commit の実装で実行)を CI に入れる形へ変更した
- Learning から Adversarial へ: 片側だけの base commit 実行では、base で pass していたテストの expected 値を壊れた実装に合わせる変更が対象外になる。両側(pass していたテストが pass し続けるか)まで見る必要がある。Adversarial は反例を認め、交差実行の範囲を正確に書き直した
- Design から Formal へ(逆方向): 「検出力は測れる」という主張は言い過ぎであり、決定不能な包含を有限サンプルで近似しているにすぎない。Design は「コストに応じた精度で近似できる」という条件で受け入れた
- Design と Adversarial から Architecture へ: 例外ファイルをエージェントがチャット承認から転記する設計は、自己申告と区別できない。Architecture はファイルの作成者を hook に移し、権限の源泉を PR の CODEOWNERS レビューに置いた
- Adversarial と Design から Architecture へ: 「N 回で RequiresPolicyChange へ昇格」という条件は同じ Maxim の繰り返しにしか効かず、異なる ask の総数は減らない。精度なしで入れると表の緩和を誘発する。Architecture は適用先を弱体化種別の承認に限定した。Learning は、昇格の方向を Reject 率で決める案を足した
- Practical と Learning から Architecture へ: 「決定的な層で捕まらなかった事例が出てから Reviewer を足す」という方針は、捕まらなかった事例が観測されないため、永久に足さない基準になる。Architecture はこれを撤回し、評価コーパスで recall を測ってから足す形へ変えた
- Formal と Learning から Practical へ: 「表にない組み合わせは RequiresApproval」で open world は解けない。分類器が区別できない変更は既存の行(`impl | modify`)に誤分類されて Allowed になる。Practical は open world の穴が action_kind ではなく artifact_kind の側にあると認めた
- Practical と Architecture と Adversarial の間で、宣言(declared_action_kinds)の価値。「不一致の検出」では省略と過剰申告の動機が残る。Adversarial が「事前承認の入口」としての価値を認め、Learning が「declared 集合自体を Plan 時点で判定表に通す」案を示し、全員がその形で一致した
- Design から Practical へ(双方向): Design は「マーカーと宣言ファイルを先にする」方針から「第1週は deny と3信号のみにする」方針へ変え、Practical は「不変条件の言葉は要らない」という主張を撤回した

## 反論を受けて変更された意見

| 役割 | 変更前 | 変更後 |
|---|---|---|
| Architecture | 例外 YAML はエージェントが書く。Stop hook が本体。N 回昇格をすべての RequiresApproval に適用。「定言命法」を発想の補助として残す。取りこぼしが出てから Reviewer を足す | hook が書き、PR の CODEOWNERS が源泉。CI が本体。昇格は弱体化種別のみ。名前はランタイム成果物から外し、表の理由欄に守る仕組みを書く。評価コーパスで測ってから足す。対話モードと無人モードで既定値を変える |
| Formal | `layerRule` を不変条件として書く。Lean で policy のメタ検証。Observable 述語は `RepoState → Prop` | 不変条件を「観測の変化には、エージェントが作れない宣言が要る」の1スキーマ3実例に縮約。Lean と Alloy は現時点では不採用。述語は `Trusted BaseState → HeadState → Prop` とし、交差実行をラベル非依存の主要な観測にする |
| Adversarial | ask を広く。別コンテナでの検証。宣言は通る表現を学ぶ機会。同一モデルの Reviewer に独立性はない | ask は種別で出す(攻撃14)。本体は CI。宣言は事前承認の入口。圧力の分離による差はあり得るが未測定 |
| Design | 不変条件の基準は「検証する仕組みの検出力」。宣言ファイルとマーカーを先に。チャット承認は不可。Postcondition は mutation score | 基準を「変更の正当性を機械的に確かめる根拠の検出力または信頼性」に広げ、公開契約と検証・生成の構造を定める依存方向を含める。第1週は deny と3信号のみ。転記主体が hook なら可。交差実行を採用し、削除テストの参照シンボルの残存を最小構成の検査にする |
| Learning | `assert add(2,2) == 5` は repository state を見る判定器では捕まらない。評価データは公開 trajectory から | 両側の交差実行で捕まる。一次ソースは自分の audit log の3つ組(表の判定、Reviewer の判定、人間の決定)。承認要求の要否は行ごとの Reject 率で測る |
| Practical | Stop hook が本体。静的な数え上げで足りる。`Edit(./tests/**)` を ask。audit log は git 管理外。不変条件の言葉は要らない。Reviewer の出力は質問文 | CI が本体。交差実行を CI に入れ、snapshot 更新と expected 値の書き換えは `retarget` として静的に拾う。ask は種別で出す。例外ファイルは hook が書き PR に入る。表の行を足す基準は提案書の問いそのもの。Reviewer の出力は規則の候補(`proposed_rule`) |

## ケーススタディ

第3段階で各役割が示した判定を、ホストが統合しました(第4段階)。判定が分かれた点は、そのまま残しています。

### ケースA: 失敗テストの削除

- Maxim(観測から): `artifact_kind = test`、`action_kind = delete`、`specification_status = unchanged`、交差実行が (pass, fail) であり、同じ diff で実装が変更されている
- 一般化された条件: 実装の変更によって落ちたテストを、仕様変更の宣言なしに消す
- 関連する Invariant: テストの検出力(検証器の保全)
- 判定: RequiresPolicyChange(Formal、Architecture、Practical)または Forbidden(Learning、Adversarial、Design)。実質的な扱いは同じであり、人間が `spec-change` ラベルを付けるまで処理が止まり、エージェントは削除を選択肢に出さない。変更前から失敗していた(base で 3/3 失敗)、または flaky(1〜2/3 失敗)であれば、削除ではなく隔離(skip + Issue 参照 + 90日上限)を RequiresApproval で認める(Practical)。マーカーのない実装詳細テストは RequiresApproval とし、Approve を「このテストを実装詳細と宣言する」一回限りの宣言として扱う(Formal)
- 必要な Authority: 削除は Maintainer(ラベル)。隔離は User
- Scope と Lifetime: One-shot。隔離は Temporary
- 反例: 実装詳細テストの正当な削除(参照シンボルが消えていれば Allowed)、flaky による偶然の反転
- 実行前に確認すべき事実: base での成否(3回)、ラベル、最終変更日、削除テストの参照シンボルの残存、マーカー
- 実行後に確認すべき状態: テスト宣言数の減少がラベルか例外記録で説明できること。隔離の場合は Issue 参照つきの skip であること。交差実行の記録

### ケースB: 仕様変更に伴うテスト更新

- Maxim: `test`、`retarget`(expected 値の置換)、`specification_status = changed`(Declared)、交差実行が (fail, pass)
- 一般化された条件: 宣言された仕様変更に合わせてテストを変える
- 関連する Invariant: 宣言と観測の整合
- 判定: Allowed(ログのみ)。条件は、provenance がエージェントには作れない形(ラベル、hook が書いた決定ファイル、人間の commit、ADR)で存在し、交差実行が (fail, pass) であること。(pass, pass) なら RequiresApproval とする。provenance がなければ RequiresPolicyChange となる(Learning、Formal、Architecture)。Practical のチーム運用と Adversarial は RequiresApproval としたが、Design の折衷案(AskUserQuestion への Yes を受けて hook が決定ファイルを書く)によって個人開発でも満たせる
- 必要な Authority: 宣言は User または Maintainer。編集は Agent
- Scope と Lifetime: Task、Permanent
- 反例: 宣言の範囲より広く変えた。assert を弱めただけ(base でも通る)。共有 assertion まで変えた。誤った実装に合わせた誤った expected 値(ラベルがある場合は機械検証できないという、Learning に残る懸念)
- 実行前に確認すべき事実: provenance の存在。失敗テストが対象シンボルを参照しているか。公開 API の変更の有無
- 実行後に確認すべき状態: 交差実行が (fail, pass) であること。head で pass すること。対象外のテストが変わっていないこと。テスト宣言数が減っていないこと

### ケースC: 生成コードの一時修正

- Maxim: `generated`、`modify`、git で追跡あり、CI に再生成チェックあり
- 一般化された条件: generator を直さずに生成物を直接直す
- 関連する Invariant: source of truth の保護(再生成チェック)
- 判定: RequiresApproval(Temporary)。全員一致。期限なしは Forbidden と同じである(Adversarial)。同じファイルで2回目となる場合は Maintainer へ回す(Practical)。CI に再生成チェックがなければ例外は TODO と同じであり、再生成チェックの追加を先に求める(Formal、Design)
- 必要な Authority: User(2回目は Maintainer)
- Scope と Lifetime: path 単位。Temporary であり、解除条件は「Issue のクローズ」「generator のパッケージ版の変更」「再生成しても差分が出ない」のいずれかを満たすこと、かつ日付上限(30〜90日)
- 反例: ヘッダだけ残った手書きファイル(マーカー更新が正解)。generator の修正が実は小さい場合
- 実行前に確認すべき事実: 追跡の有無、generator の所在、再生成チェックの有無、既存の例外記録、Issue 番号
- 実行後に確認すべき状態: 例外ファイル(hook が生成)が PR にあり、`expires` が機械判定できる形で書かれていること。再生成 CI が例外を参照して該当ファイルだけ免除すること。`<auto-generated>` ヘッダが残っていること

### ケースD: lint の無効化

- Maxim: `lint_config / disable_rule` または `impl / add_inline_suppression`。`violating_lines ⊆ added_lines`(Observable。lint 出力の行番号と diff の追加行から計算。Learning)
- 一般化された条件: 新たに書いたコードの違反を、ルールを弱めることで通す
- 関連する Invariant: lint の検出力
- 判定: 行単位の抑制(理由つき)は RequiresApproval(Temporary)。設定ファイル単位(rule off、severity の引き下げ、rule set の差し替え、`NoWarn`、除外の追加、ツールの版戻し)は RequiresPolicyChange(Adversarial、Learning、Architecture)または RequiresApproval(Formal、Design、Practical)で意見が分かれた。`@ts-expect-error` のように検出力を下げない抑制は Allowed(Adversarial)。lint 実行や CI の lint job の変更は deny。コードを直すのは Allowed
- 必要な Authority: 設定は Maintainer。行は User
- Scope と Lifetime: 設定は Repository で Permanent。行は One-shot で Temporary
- 反例: linter のバグで偽陽性が量産されている(上流 Issue を解除条件とする Temporary が正解)。ルール意図の回避(`void x;`、`as unknown as T`)は検出不能
- 実行前に確認すべき事実: 違反行が追加行か既存か。同じルールの既存の抑制数。CI で lint が required か。設定ファイルの要素単位での保護
- 実行後に確認すべき状態: 設定の diff が空であるか、承認があること。抑制の増分が承認数以下であること。ツールの版が下がっていないこと。除外が増えていないこと。base の設定で head を lint して、違反数が base 以下であること(Adversarial と Learning。交差実行の lint 版)

### ケースE: 公開 API の破壊的変更

- Maxim: `public_api`、`remove_or_change_symbol`、`breaking_change_declared = false`(Declared)、利用者あり
- 一般化された条件: 宣言なしに公開契約を変える
- 関連する Invariant: 公開契約の宣言(ApiCompat または PublicApiAnalyzers と、その宣言ファイル)
- 判定: RequiresPolicyChange。全員一致。代替案「新 API を追加し、旧 API に `[Obsolete]` を付けて残す(旧の振る舞いは不変)」は、RS0016 の宣言更新だけで Allowed となる
- 必要な Authority: Maintainer(メジャー版の権限)
- Scope と Lifetime: Repository、Permanent
- 反例: 宣言ファイルがなく、内部用の public 型を契約とみなす過剰検出(Design 例12。宣言ファイルを置けば解消する)。ADR で削除時期が決まっていた場合。攻撃15(新 API を追加、旧 API を委譲に変更、`[Obsolete]` の3段階に分けて旧 API の戻り値の意味を変える)は、契約テストがなければ検出不能(Adversarial)
- 実行前に確認すべき事実: 宣言ファイルの有無。契約テストの有無。ラベル。利用者(パッケージ公開、モノレポ参照)。ADR の有無
- 実行後に確認すべき状態: ApiCompat の差分が空であるか、ラベルと宣言更新があること。契約テストが変更されていないこと(変更があるなら交差実行とラベル)。`.editorconfig` で RS0017 を黙らせていないこと。利用者側のプロジェクトでビルドが通ること

### ケースF: 不要になったテストの削除

- Maxim: `test`、`delete`、`specification_status = removed`(Declared)、交差実行が (pass, fail または error)、削除テストの参照シンボルがすべて同じ diff で削除されている(Observable)
- 一般化された条件: 廃止が宣言され、実装から消えた仕様のテストを消す
- 関連する Invariant: 宣言と観測の整合。残る仕様の検出力
- 判定: Allowed(ログのみ)。条件は provenance があり、かつ削除テストの参照シンボルが head に残っていないこと。1つでも残れば RequiresApproval とする(「このテストは `RoundToCents` も検証していた。別のテストで保証されているか」などの確認が必要)。head で通るテストは削除対象から外す(Formal)。これらについて全員が一致した
- 必要な Authority: 廃止の宣言は User。削除は Agent
- Scope と Lifetime: Task、Permanent
- 反例: 共有 assertion(廃止と残存を1つの Fact で検証している)。宣言の範囲より広く消した。実装が残っているのにテストだけ消した(これはケース A に該当)
- 実行前に確認すべき事実: provenance の有無。実装から廃止シンボルが消えているか。参照シンボルの一覧とその残存状況
- 実行後に確認すべき状態: 残存シンボルを参照する別のテストがあること。テスト宣言数の減少が削除対象数と一致すること。スイートが通ること。公開 API の削除であれば、宣言ファイルが更新されていること

「削除テストの参照シンボルが head に残っているか」という検査は、討議の中で生まれました。Design が提案し、Formal の命題C の仮定 `verifies t ⊆ removed` の実装にあたると整理され、Adversarial、Learning、Architecture、Practical が採用しました。マーカーも mutation testing も要らず、grep で実装できます。

## 採用した設計判断

第5段階(最小実装)と第6段階(将来拡張)で確定した判断です。誰の提案が元になったかを示します。

- 判定の本体は CI の required check に置く。Stop hook は手元の写しで権限を持たない。PR を使わない個人開発だけ Stop hook に block を持たせる(Practical が提案し、Architecture、Adversarial、Formal が同意)
- 審査は二層にする。tool 呼び出し単位の決定的な Gate(deny と ask)と、作業単位の diff 分類(Architecture)
- Maxim の Context と Action は、分類器が git object(`base...head`)から導出する。エージェントは書かない。Purpose は審査に使わない(Formal、Adversarial、Practical)
- 交差実行(変更されたテストを base の設定で base と head の実装に対して実行し、結果の組で判定する)を v1 に入れる(Adversarial が提案し、Learning が両側に拡張し、Design と Practical が採用)
- 削除テストの参照シンボルが head の実装に残っているかを検査する(Design が提案し、Formal の命題C の実装として整理され、全員が採用)
- 不変条件は「観測の変化には、エージェントが作れない宣言が必要である」の1スキーマ3実例(検証器の保全、宣言と観測の整合、source of truth の保護)に限定する(Design の基準を Formal が形式化)
- 層のアーキテクチャ規約は不変条件から外し、「それを検証するアーキテクチャテストを無効化しない」に置き換える。検証と生成の構造を定める依存方向(test → prod、generated ← handwritten、verifier が書込範囲に依存しない)は不変条件に残す(Design)
- 一時例外(RequiresApproval)と仕様変更(RequiresPolicyChange)で provenance の要件を変える。前者はダイアログの決定を hook が diff ハッシュ付きで記録すればよく、後者は人間が作る成果物(Issue のラベル、ADR、宣言ファイル)か hook が書く決定ファイルが必要である(Formal、Architecture、Design)
- 例外ファイル `.agents/exceptions/*.yaml` は hook だけが書く。`.agents/**` はエージェントに deny する。`expires` は機械判定できる4種(Issue クローズ、パッケージ版の変更、再生成で差分なし、日付上限)に限る(Architecture が提案、Adversarial の攻撃27 を受けて ask から deny に変更)
- Forbidden の実体は資格情報(エージェントの token に `issues: write` を与えない)と branch protection に置く。harness の deny は既定値(Architecture)
- ask は「パス」ではなく「信号」で出す。テスト本体の編集は確認せず、削除、無効化、弱体化、期待値の書き換えが観測されたときだけ確認する。1作業単位につき確認は1回(Adversarial の攻撃14、Design)
- 承認疲れの対処は、確認の種類ごとの Reject 率で行う。0 に近い行は Allowed へ昇格か原因修正、1 に近い行は Forbidden へ。N 回昇格は採用しない(Learning、Design)
- 導入の最初の2週間は記録のみとし、確認の頻度を測る。記録のみ期間の統計は頻度の見積もりにだけ使い、昇格の方向には使わない(Practical、Adversarial の攻撃22)
- 計画段階の宣言(`declared_action_kinds`)は事前承認の入口として使う。既定値は空集合とし、宣言した ask 相当の種別は Plan 時点で判定表に通す(Practical、Learning、Adversarial の攻撃18)
- LLM Reviewer は無人実行を始める時点で1つだけ足す。入力は diff と観測値、出力は固定 JSON(`NoFinding` か `RequiresApproval`、および表に足す行の候補)。Allowed も Forbidden も出さない。限界効用は「Reviewer ≠ 表の判定 かつ 人間 = Reject」の件数で測る(Practical、Learning、Architecture)
- Reviewer の判定は diff ハッシュに束縛し、無人モードでは `gh pr close` と `gh pr create` を deny する(Adversarial の攻撃26)
- Reviewer の `proposed_rule` を表に採用するときは、「この組を同じ条件のすべての場合で許したら、どの検証の検出力が落ちるか」への回答を必須にする(Adversarial の攻撃23)
- AGENTS.md には不変条件と検査の対応表、保護パス、質問の形式、仕様変更の根拠の宣言、判定表の所在と行を足す基準を書く。「一般化して審査せよ」という思考手順と判定表そのものは書かない(Design、Practical)
- 「定言命法」「Maxim」「普遍化」の語はランタイム成果物に出さない。行を足す基準の1文として問いだけを残す(Practical が「名前は不要」、Architecture と Design が「問いは残す」で一致)
- audit log のスキーマは v1 から「表の判定、Reviewer の判定、人間の決定」の3つ組と観測値、diff ハッシュ、base SHA を持たせる。分類器の回帰 fixture(30〜60 件)を v1 のユニットテストとして入れる(Learning)
- 無人実行の PR には「分類器は何を判定し、何を見ていないか」を CI が自動で書く(Adversarial、Architecture)
- Policy の階層は6層(Global、Organization、Repository、Directory、Issue、Temporary Exception)。規則は上から下へ流れ、判定は最も厳しいものを採り、下げられるのは一時例外が RequiresApproval の行を下げる場合だけ(Architecture)
- ケースD の設定ファイル単位の lint 無効化は、v1 の表では RequiresPolicyChange にする。上流 Issue で解除条件が機械判定できる場合だけ RequiresApproval(Temporary)の例外を認める(Practical が最終ラウンドで多数側に移り、Adversarial と Architecture が同意)
- ケースA の判定名は作業単位層では RequiresPolicyChange とし、tool 呼び出し層の `rm tests/` は deny と呼び分ける。エージェントへの指示に「削除を選択肢に出さない。ラベル付けを提案しない」を書けば、Forbidden と同じ攻撃面になる(Architecture、Adversarial)

## 採用しなかった案と理由

| 案 | 出所 | 理由 |
|---|---|---|
| 5役割パイプライン(Maxim Generator、Universalization Verifier、Counterexample Generator、Executor) | 元の提案書 | 同じモデルを4回呼ぶ構成で独立性は増えず、遅延とトークンが増えるだけ。反例探しは参照シンボルの検査と交差実行が担う(Architecture、Practical、Formal) |
| エージェントが Adopt と判定した一般則を自律実行する | 元の提案書 | 一般則の追加はポリシーの追加であり、エージェントの権限ではない(Architecture) |
| Reviewer が Maxim の文面を読んで判定する | 元の提案書 | 自己申告の審査になり、文面で攻略される(Adversarial の攻撃8、全員) |
| `Adopt = +1 / Reject = -1 / Local-only = 0` の報酬 | 元の提案書 | 説明文と承認要求が最適化され、行動は変わらない。報酬は機械検証できる量に限る(Learning、Adversarial) |
| Lean による policy のメタ検証 | Formal の第1段階 | 判定表 10〜18 行 × 不変条件 3 実例は目視できる。判定が状態依存になったら Alloy を先に試す(Formal が取り下げ、Design が同意) |
| `layerRule`(依存方向)を不変条件として宣言する | Formal の第1段階 | 設計判断を禁止する。不変条件は「アーキテクチャテストを無効化しない」(Design、Formal が受け入れ) |
| `Edit(./tests/**)` を一律 ask にする、保護パスを広く取る | Practical と Adversarial の第1段階 | 承認疲れを生み、承認の希釈(攻撃14)の材料になる(Design、Adversarial が撤回) |
| 同じ Maxim の承認が N 回続いたら RequiresPolicyChange へ昇格 | Architecture の第1段階 | 異なる ask の総数は減らず、精度なしでは表の緩和を誘発する。Reject 率で代替(Design、Learning、Adversarial) |
| 例外ファイルをエージェントがチャット承認から転記する | Architecture の第1段階 | 転記は自己申告(Design、Adversarial。Architecture が撤回) |
| Stop hook を判定の本体にする | Practical の第1段階 | ローカルで迂回でき、hook スクリプトが書込範囲にあり、Codex と Copilot に効かない(Practical が撤回) |
| 静的な数え上げだけで弱体化を検出する | Practical の第1段階 | assert 緩和と fixture 差し替えを取りこぼす。交差実行で補う(Adversarial、Learning。Practical が受け入れ) |
| mutation testing を最小構成の Postcondition にする | Design の第1段階 | 中規模プロジェクトで数十分かかる。expected 値の上書きも捕まえない(Adversarial、Learning。Design が交差実行に差し替え) |
| 宣言ファイルとマーカーを先に整備してから導入する | Design の第1段階 | 整備中は何も守られず、導入の障壁になる。参照シンボルの検査で代替し、Approve 率を見てから投資する(Practical。Design が撤回) |
| hidden test を推論時の Gate に使う | Learning の第1段階 | 実運用のリポジトリに存在しない。保護テスト集合(人間だけが更新するパス)で代替(Design、Adversarial) |
| 別コンテナでの検証を独立した部品にする | Adversarial の第1段階 | CI の clean checkout が同じ効果を持つ(Architecture。Adversarial が CI に移した) |
| 事後 Gate で Scope 外のファイルがあれば自動 revert する | Adversarial の第1段階 | revert は別の破壊操作。停止と報告までにする(Architecture) |
| Reviewer の出力を質問文だけにする | Practical の第1段階 | 説明生成器と変わらない。表の行の候補(`proposed_rule`)を出す装置に変更(Design の指摘を Practical が受け入れ) |
| 「決定的な層で捕まらなかった事例が出てから Reviewer を足す」 | Architecture の第1段階 | 捕まらなかった事例は観測されないので永久に足さない基準になる。評価コーパスで測ってから足す(Practical、Learning。Architecture が撤回) |
| AGENTS.md に「一般化して審査せよ」と書く | 元の提案書 | 判定は観測と表が出すので結果は変わらず、「審査済み」に見える文章を生成する攻撃面になる(Practical の経験則、Design。未計測) |
| Issue 本文の解析で仕様変更を判定する | 検討対象 | 自由記述は機械的に読めず、エージェントに有利な読み方が常にある。ラベルのみ(Practical、Adversarial) |
| 複数 LLM の Reviewer | 検討対象 | 見逃しの相関が未測定。測ってから、無人モードの second reader に限って検討(Architecture、Learning) |
| 組織 Policy、Directory Policy、Issue form、exception debt の集計、read-only mount | 第6段階の将来拡張 | それぞれ採用の条件(リポジトリ3つ以上、モノレポで verdict を変える必要、例外5件超、無人実行の開始)が現実に起きるまで入れない(Design、Architecture) |

## 未解決事項

討議で結論が出なかった点、測定が必要な点、および本討議の手法の限界です。

- 検出力は「測れる」(Design)か「コストに応じた精度で近似できる」(Formal)か。記事では後者を採ったが、交差実行の捕捉率とコストは誰も計測していない
- 同一モデルの Reviewer の盲点の相関。「圧力の分離による差はあり得るが未測定」(Adversarial)で止まっている。Learning が設計した測定(既知の攻撃 30 件と正当な変更 30 件を2モデルに渡し、同時に見逃す率と転移率を測る)は未実施
- 承認疲れの閾値(確認の頻度、Reject 率の境界値)は運用でしか分からない。小規模チームでは数週間で統計的に意味のある値にならない可能性がある(Learning)
- 「AGENTS.md に思考手順を書いても効かない」は経験則であり未計測である(Practical)
- 個人開発では、仕様変更の宣言がダイアログの Yes に退化し、一時例外と同じ重さで押されるリスクがある(Adversarial の攻撃24)。ユーザー自身が1行のファイルを書く操作にする案が出たが、手間との交換であり設計では決まらない
- 交差実行の粒度。ファイル単位では、新機能のテストを既存ファイルに足すたびに ImportError で観測不能になり、確認が量産される(Adversarial の攻撃19)。関数単位の実装(1〜2日)が必要だが、v1 には入っていない
- テストコードが交差実行の環境を検出して結果を偽装する攻撃(攻撃20)は、環境依存コードの静的検出で ask に倒す案までで、完全には塞げない
- `.agents/**` を deny にしたとき、Bash 経由の書き込み(`echo > .agents/...`)を harness で止められるか。CI 側で commit author を見る案があるが未検証である
- Claude Code の PostToolUse hook で、自分が ask を出した tool call と実行された tool call を `tool_use_id` で対応づけられるか(Architecture)。`permission_mode` が hook の入力に含まれるか(Practical)。Stop hook の `decision: block` が `bypassPermissions` でも効くか(Practical)。いずれも未確認である
- Codex CLI と Cursor の hook 相当の仕組み、Copilot coding agent での運用は、CI を唯一の判定層とする前提で整理したが、各ツールの最新仕様は未確認である
- コンパイル言語で交差実行がコンパイルエラーになる場合(公開 API のシグネチャ変更)、観測不能として ask に倒すが、弱体化は検出しない
- 複数のエージェントが同じブランチに同時に書き込む運用は扱っていない(Formal)
- 別モデル(Codex)による独立分析は、CLI の版の問題で2回とも失敗した。6役の合意は同じモデルの6つの視点から出た内部整合であり、独立した裏付けではない。共通の盲点の候補として、GitHub の機能を前提にしたこと、行動を「diff を生む操作」に限定して考えたこと、交差実行を「安い」と全員が言いながら計測していないことが挙がった(Architecture、Learning)
- 最終ラウンドの6役は、ファイルを書き終えた直後に利用制限で停止した。要約の返信は得られず、ホストはファイルを直接読んで統合した。内容は完結しており、結論に影響はない
