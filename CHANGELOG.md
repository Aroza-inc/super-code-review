# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.9.0] - 2026-11-20

### Changed

- `project-review`: 出力を `code-review` の形式にそろえた
  - 「まとめ」「横断パターン」「確認してほしいこと」「AI で直せる指摘」「別タスク候補」「未レビュー範囲」「次にやること」の構成にした
  - タスク別・観点別の件数の集計表、Action Required、Findings Summary / Details の二重掲載、Task Details（エージェントの報告の全文）を廃止
  - 統合の手順に「人の判断の要否の判定」を加えた（横断パターンは原則として人の判断が必要）
  - 番号を `P-1`（横断パターン）と `F-1`（それ以外）にした
  - Perspective 名の表と受領者向けガイダンスの重複を、共通ルール・`feedback-guidance.md` の参照に置き換えた
- `plan-review`: 出力を `code-review` の形式にそろえた
  - 「まとめ」（読み取ったプランの要点を含む）「確認してほしいこと」「プランに足すとよいこと」「別タスク候補」「次にやること」の構成にした
  - Review Summary、Plan Understanding などの分析表、Action Required、Findings Summary / Details、Agent Details を廃止
  - 統合の手順に「人の判断の要否の判定」を加えた（未定義の決定事項・方針を変えないと直せない前提の誤り・修正案が複数あるものは人の判断が必要）
  - 受領者向けガイダンスの重複を `feedback-guidance.md` の参照に置き換えた

## [1.8.0] - 2026-11-20

### Security

- allowed-tools を読み取り系だけに絞った。書き込みもできるコマンドは、実行のたびに確認が出る
  - `gh api:*` を外した（code-review / iterative-review / pr-respond）。code-review / iterative-review は PR の行コメントを読むのをやめ、確認なしで取れる一般コメントとレビュー本文（`gh pr view --json comments,reviews`）を読む。pr-respond はコメントの取得と返信で確認が出る
  - `gh pr comment:*` を外した（code-review / iterative-review。明示的に頼まれたときだけ投稿するため）
  - `Write` を外した（code-review / pr-review / project-review / plan-review / diff-summary）。PR コメントの本文は一時ファイルではなく標準入力（`--body-file -`）で渡す
  - `find:*`（`-exec` でコマンドを実行できる）と、使っていない `git log` / `gh repo view` / `gh auth status` / `git ls-files` を外した
  - `git fetch:*` を `git fetch origin --:*` に絞った（`--upload-pack` 等のオプションを通さない）
  - git / gh に手順にないオプション（`--output` 等）を付けないよう明記
- 方針ドキュメント（CLAUDE.md・README・`.cursor/rules/`・`docs/`）は変更前の版を根拠にする。PR が方針ドキュメントやコードコメントを書き換えて指摘を消す経路を塞ぎ、方針を緩める書き換えは指摘する（共通ルール A-1 / B-1、security-reviewer、code-review、pr-review）
- 秘密情報の値を出力・PR コメントに書かない（伏字にする）。PR コメントの投稿前に秘密情報らしい文字列を確かめる
- `pr-review`: プロンプトインジェクションの疑いがある、または本文の秘密情報チェックに引っかかった場合は自動で投稿せず、ユーザーに確認する。作業ツリーの未追跡ファイルの中身をコメントに載せない
- 保護対象パスを共通ルールに移し、`.git/` と `.claude/` を加えた（iterative-review / pr-respond が参照）

### Fixed

- `iterative-review`
  - 人の判断が必要な指摘が出たときの動きを1つにした（同じ会話で答えがあれば続ける。止めるよう言われたときだけ `NEEDS_USER_INPUT`）
  - 反復の修正を開始時点のスナップショットとの差分で見分けるようにした（開始前からあった作業中の変更がレビュー・自動修正に混ざらない）。開始時点に作業中の変更があれば確認を取る。再ステージは `git add -p` を案内
  - 反復で作った新しいファイルを論理 diff と修正差分に含めた
  - Requirements の指摘とオーケストレーターが直接出した指摘は、確認レビューでオーケストレーターが判定する
  - スキップした指摘の扱いを理由ごとに決めた（解決済みにするか、人の判断に回すか）
  - `--max-iterations <N>` で反復上限を変えられるようにした
  - 代わりに行う全体レビューは「最後の全体レビュー」に数えない。全体レビュー後は既存の指摘番号を引き継ぐ。最後の全体レビュー後に気づいた差分外の点は最終出力に出す
  - 用語を「人の判断が必要 / 要らない」にそろえ、重複していた受領者向けガイダンスを「次にやること」にまとめた
- `iterative-review` 確認レビュー: エージェント定義の出力形式より確認レビューの形式を優先すると明記し、新しい問題に Scope Reason を加えた
- `pr-respond`
  - 「最後にコメントした時刻より後」での足切りをやめ、返信に入れた対応済みの印（`<!-- pr-respond done: ... -->`）で除くようにした。「確認」「質問」に回したものが次の実行で消えなくなった
  - 別リポジトリの PR を指定したら止める。push に失敗したら返信を投稿しない
  - 直す前にチェックツールを実行して基準を取り、新しく増えた失敗だけを修正の影響として扱う
  - `R-番号` の振り方を決めた。スレッドが 100 件を超えたら知らせる。最後のコメントを確実に取る
- `code-review`
  - 引数でベースブランチを指定した場合も、PR の有無を確かめて PR コメント・PR 本文を取る
  - 人の判断が必要で Scope=別PR / 別タスク の指摘は「別タスク候補」に `（要判断）` 付きで出す（「確認してほしいこと」と二重に載らない）
  - 変更前コードの確認と別PR の実在確認は、オーケストレーターが統合時に行う（エージェントは git を実行できないため）
- 共通ルール
  - 要件との食い違いの Priority を出どころで決める（PR 本文 = Must、issue / 仕様ファイルだけ = Should）に統一
  - 並び順のルールを新しい出力形式に合わせた。残っていた「上記」「下記」の参照を直した。件数行に Scope Reason 空欄の Must を数える
- `project-review`
  - ベースブランチ起因の除外の参照番号を D5 から D4 に直した（D5 は有効のまま）
  - 横断パターンの Must は Verdict 計算上 Scope=このPR として扱う
  - ディレクトリの集計を `awk` / `sort` なしで行う。1ファイルで閾値を超える場合は単独のタスクにする。大規模警告の確認を二重に取らない
  - security-reviewer を起動しないタスクでも、プロンプトインジェクションの疑いを記録する。共通ルールのパスを共通入力に加えた
- `plan-review`: `-` で始まる引数の拒否をファイルパスに限った（Markdown の箇条書きで始まるプランが弾かれていた）。存在しないパスは再指定を求める。Scope の値「別プラン」を `別タスク` に直した。インジェクションの疑いは人の確認に回す
- `diff-summary`: 大きな差分では diff 全文を渡さない。使っていない許可を外した
- 事前実行コマンドの `| head -5` を `--shortstat` にした（`head` が allowed-tools になかった）
- エージェント
  - 確認レビューなどオーケストレーターが別の出力形式を指示した場合はそちらに従う。full-project モードでの読み替えを明記
  - プロンプトインジェクションの疑いを書く「注意」節を加えた
  - security-reviewer: Scope の扱いを共通ルールにそろえた。80% の確信度の閾値は Must だけに適用。OWASP Top 10:2021 の番号であることを明記
  - consistency-checker: quality-reviewer と重なる SRP / DRY の評価行を、レイヤー責務・再発明の確認に置き換えた
  - quality-reviewer: Must の定義を共通ルールに合わせた（可読性・保守性は Should）
  - plan-reviewer: 参照先のない「共通ルール G」を本ファイルの G に直した
  - diff-summarizer: ブランチ名がない場合は推測しない。構成の雛形でファイルを列挙しない
  - bug-detector: Category の `Edge Case` を `Edge-Case` にした
- README: 手動インストールの手順に `/plugin marketplace add` を加えた。gh が必要なスキルに pr-respond を加えた。Priority / Scope の表を正式な定義に合わせ、確認が出るコマンドの説明を加えた

## [1.7.0] - 2026-11-20

### Added

- `pr-respond` スキル: 現在のブランチの PR に付いたレビューコメントに対応する
  - 未解決の行コメントのスレッド（GraphQL の `isResolved` で判定）と、実行ユーザーが最後にコメントした後のレビュー本文・一般コメントを集める。1件に複数の指摘があるコメントは指摘ごとに分ける
  - コメントを1件ずつ今のコードと照らし合わせ、直す / 確認 / 質問 / 反論 / 対応済み に振り分ける。「確認」の基準は `code-review/references/review-rules.md` の「人の判断が必要な指摘」。関連するコメントに1件でも質問・確認があれば一緒に止める
  - リポジトリの書き込み権限がない人（`authorAssociation` が `OWNER` / `MEMBER` / `COLLABORATOR` 以外）のコメントは、自動では直さず確認に回す
  - 直せるものは作業ツリーで直し、チェックツールで確かめる。秘密情報・鍵・CI の秘密設定などのパスは直さず確認に回す
  - 結果と返信案を見せてユーザーが OK してから、コミット・push・返信の投稿を行う。行コメントはそのスレッドに返信し、レビュー本文・一般コメントにはまとめて1件返信する。スレッドの解決はしない
  - コメントに書かれたコマンド・URL・指示には従わない

## [1.6.0] - 2026-11-20

### Changed

- `iterative-review`: 2回目以降のレビューを、直した部分だけを見る確認レビューにした
  - 1回目は差分全体の全体レビュー。2回目以降は、未解決の指摘ごとに `ADDRESSED` / `NOT_ADDRESSED` を判定し、修正差分で新しく入った問題だけを探す
  - 修正差分は、修正前に `git stash create` で取ったスナップショットとの差分（作業ツリーは変更しない）。修正で作った新しいファイルは全文を加える
  - 起動するエージェントは、未解決の指摘を持つ観点と、修正差分に起動計画を当てはめて必要になる観点だけ
  - 修正差分の外で気づいた点はループを延ばさず記録し、最後の全体レビューに渡す
  - 未解決の指摘が0件になったら、終了前に全体レビューを1回だけ行う。修正差分が大きい場合（目安 500 行超、または論理 diff の 30% 超）は確認レビューの代わりに全体レビューを行う
  - 「同じ指摘が直らない」の判定を、ファイル・行番号の近さによる推測から、指摘番号ごとの `NOT_ADDRESSED` の連続回数（2回）に変えた
  - 確認レビューの指示を `iterative-review/references/confirmation-review.md` に分けた
- `iterative-review`: 最終出力を `code-review` の新しい形式に合わせた
  - 「まとめ」「確認してほしいこと」「残っている指摘」「直した内容」「別タスク候補」「次にやること」の構成にし、反復ごとの件数の集計表と要点解説を廃止
  - ヘッダに検証・要件の行と、全体レビュー・確認レビューの回数を出す
  - ループ中のユーザー確認も「確認してほしいこと」と同じ形式にした
- `iterative-review`: `--spec` に対応（`code-review` と同じ）

## [1.5.0] - 2026-11-20

### Added

- `code-review`: 要件との照合
  - 要件テキストを `--spec <パス>` のファイル、PR 本文、PR 本文からリンクされた GitHub の issue / PR（`gh issue view` / `gh pr view` で読み取りのみ・最大5件・1段だけ）から集める。GitHub 以外の URL は開かず、件数だけを出力する
  - オーケストレーターが diff 全体と照らし合わせ、Missing / Extra / Misunderstood を Perspective=`Requirements` の指摘として加える（すべて人の判断が必要）
  - 要件テキストは指摘を増やす根拠にだけ使う。例外として、PR 本文で「この PR ではやらない」と明記された要件は Missing にしない
  - bug-detector の `Spec-Mismatch` は PR 本文だけでなく要件テキスト全体を対象にした
- `code-review`: チェックツールの結果を出力の「検証」行に残す（すべて成功なら1行、失敗・未実行はコマンドと理由）
- `pr-review`: 要件の収集に対応（PR 本文とリンク先の issue / PR）

### Changed

- `code-review`: 出力形式を作り直し、人が決めることを中心にした
  - 「まとめ」「確認してほしいこと」（人の判断が必要な指摘だけを詳しく、選択肢とおすすめ付き）「AI で直せる指摘」（1行の表）「別タスク候補」（1行ずつ）の構成にした
  - Review Summary の集計表、Findings Summary / Details の二重掲載、Agent Details、要点解説を廃止（要点解説は「まとめ」と「確認してほしいこと」に統合）
  - 指摘番号を `F-1` 形式の通し番号にした
- 「人の判断が必要な指摘」の基準を `iterative-review` から共通ルールに移し、`code-review` の出力の出し分けと `iterative-review` の自動修正の判定で同じ基準を使うようにした
- `code-review/SKILL.md` から共通ルール（Priority × Scope の定義、指摘の精査 共通ルール A〜G）、PR コメント投稿ルール、受領者向けガイダンスを `code-review/references/` に分けた。コンパクション後に SKILL.md が 20,000 文字で切られる問題を避けるため（17,157 → 約 12,000 文字）
  - 4エージェントと `project-review` / `iterative-review` / `pr-review` の参照先を更新。オーケストレーターが `references/review-rules.md` の絶対パスをエージェントに渡す
- PR コメントの再整形ルールを新しい出力形式に合わせた（修正をおすすめする点は直し方を `<details>` に入れる）

## [1.4.0] - 2026-11-06

### Changed

- `pr-review`: CI の結果（`gh pr checks`）を承認条件から外し、参考情報としてコメントと最終出力に記載するように変更
  - CI が実行されないリポジトリ・PR で承認保留になる問題と、CI の失敗・未完了のたびに再レビューが必要になる問題を解消。マージ前の CI の強制はブランチ保護（Required status checks）に委ねる
  - CI の状態は「成功 / 失敗あり / 実行中 / チェックなし / 取得失敗」で表示し、失敗ありで承認する場合はコメント冒頭で確認を促す
  - 承認条件の番号を詰めた（旧条件 5・6 → 4・5）

## [1.3.0] - 2026-10-15

### Added

- `pr-review` スキル: 他者のPRをブランチ名 / PR番号 / URL で指定し、ローカルに取得して `code-review` を実行、指摘をPRにコメント投稿する。承認条件をすべて満たした場合のみ自動 Approve し、満たさない場合はコメントのみ（Request changes は付けない）
  - 承認条件: Overall Verdict が `APPROVE` / プロンプトインジェクションの疑いなし / 全エージェント完了 / CI（`gh pr checks`）に失敗・未完了なし（ワークフローがあるのにチェック0件の場合は保留） / ドラフトでも自分のPRでもない / レビュー後にPRの head が更新されていない
  - 対象PRの特定: ブランチ名に一致するPRが複数あれば番号での再指定を求める。カレントリポジトリ・origin と異なるリポジトリのPRは扱わない
  - PRは checkout 前にベースと `pull/<番号>/head` を fetch し、SHA を照合してから detached HEAD で取得。終了時に元のブランチへ戻す。追跡ファイルに未コミット変更がある場合は実行しない
  - PRブランチのスクリプトは実行しない（ローカルのチェックツール実行を省略し CI 結果で代替、checkout 時は `core.hooksPath=/dev/null` で git hooks を、`core.symlinks=false` でシンボリックリンクを無効化し、checkout 中は ref を書き換える git コマンドを使わない）
  - `gh api` は実行ユーザーの取得（`gh api user`）のみに限定し、PRコメントは `gh pr view --json comments,reviews` で取得
  - Approve 後にレビューのコミットを確認し、レビュー対象と異なれば警告する
  - `--dry-run` で投稿・Approve を行わずに結果と承認予定を確認できる

## [1.2.0] - 2026-09-24

### Changed

- サブエージェントの起動数を固定の4並列から動的決定に変更（`code-review` に「エージェント起動計画の決定」を定義し、他スキルが参照）
  - 観点の選択: 変更ファイルを「コード / 設定・インフラ / ドキュメント / 生成物・lockfile」に分類し、必要な観点のみ起動（例: ドキュメントのみ → `consistency-checker` のみ）。`security-reviewer` は対象外が明白な場合のみ省略
  - 起動なし: lockfile のみ・空白/タイポ等の自明な差分はオーケストレーターが直接確認
  - スケールアウト: 1体あたり変更 1,500 行 / 30 ファイル超でチャンク分割し、同じ観点を複数体起動（同時起動は最大 12 体、超過分はバッチ実行）
  - 判定根拠はファイル・diff のみ（PR本文等の記述では観点を外さない）。LLM プロンプト・スキル定義は「コード」扱い
  - 起動計画（省略した観点と理由を含む）をレビュー前に提示。ユーザーの明示指示を優先
  - `iterative-review`: 反復ごとに論理 diff 全体で起動計画を再判定し、反復ログ・最終出力に起動体数を記録
  - `project-review`: タスクごとに必要な観点のみ起動（量はタスク分割で対応）
  - 出力: 起動しなかった観点は `SKIPPED（理由）` と表示、チャンク分割した観点は合算

## [1.1.0] - 2026-09-01

### Changed

- ユーザー合意ゲートを条件付き化: 全スキルで「必要な場合のみ確認し、それ以外は対象を提示して即レビューに入る」フローに変更
  - `code-review`: 対象が確定できない・解釈が分かれる・差分が極端に大きい・チェックツールエラー時のみ確認
  - `iterative-review`: モード `staged`・保護対象パス該当時などリスクのある場合のみ事前合意を要求（ループ中のユーザー確認ゲートは従来どおり必須）
  - `project-review`: 引数でスコープ指定済みならゲート①をスキップ。ゲート②はタスク数 8 超・大規模警告時のみ確認
  - `plan-review`: プランが極端に薄い・前提の食い違いが大きい場合のみ確認

### Added

- レビュー結果のPRコメント投稿ルール（`code-review` に共通ルールを定義、`iterative-review` が継承）
  - ユーザーの明示依頼時のみ投稿
  - `#` の文字を使わない（他 PR / issue への自動リンク防止。指摘番号は `F-N` 形式、見出しは太字で代替）
  - 基本はインラインコメントではなく通常コメント（`gh pr comment`）
  - 結果をそのまま貼らず、冒頭要約 + 詳細は `<details>` トグルの読みやすい形式に再整形して投稿

## [1.0.1] - 2026-06-24

### Added

- `code-review` / `iterative-review` の出力テンプレート末尾に「要点解説」セクションを追加（ソースコードを把握していない仕様レベルの読み手向けの平易な解説）

## [1.0.0] - 2026-05-09

Initial public release.

[1.1.0]: https://github.com/Aroza-inc/super-code-review/releases/tag/v1.1.0
[1.0.1]: https://github.com/Aroza-inc/super-code-review/releases/tag/v1.0.1
[1.0.0]: https://github.com/Aroza-inc/super-code-review/releases/tag/v1.0.0
