# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
