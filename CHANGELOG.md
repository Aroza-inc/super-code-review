# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
