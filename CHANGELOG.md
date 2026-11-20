# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
