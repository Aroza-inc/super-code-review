# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## プロジェクト概要

Claude Code 用の包括的コードレビュープラグイン。5つのスキル（差分レビュー / プロジェクト全体レビュー / 実装前プランレビュー / 差分サマリー / 反復修正レビュー）と6つの専門サブエージェントを組み合わせ、ソフトウェア開発の各フェーズに応じたレビュー・把握支援を提供する。

## アーキテクチャ

```
.claude-plugin/marketplace.json    # マーケットプレイス登録情報（owner: Aroza-inc）
plugins/super-code-review/
  .claude-plugin/plugin.json       # プラグインメタデータ
  skills/
    code-review/SKILL.md           # 差分レビュー（ステージ変更 or ベースブランチとのdiff）
    project-review/SKILL.md        # プロジェクト全体 or 領域指定の包括レビュー
    plan-review/SKILL.md           # 実装着手前のプランレビュー
    diff-summary/SKILL.md          # 差分の概要・構成・アーキテクチャ影響の要約（指摘なし）
    iterative-review/SKILL.md      # code-review を内包し、Scope=このPR の指摘がなくなるまで修正→再レビューを反復
  agents/
    bug-detector.md                # 潜在バグ・ロジックエラー検出
    security-reviewer.md           # セキュリティ脆弱性検出（OWASP Top 10）
    consistency-checker.md         # 既存パターン・規約との整合性検証
    quality-reviewer.md            # 可読性・保守性・パフォーマンス評価
    plan-reviewer.md               # 実装プランと既存コードベースの整合性評価
    diff-summarizer.md             # 差分の要約生成（指摘なし、把握用）
```

### スキルとエージェントの対応

| スキル | 用途 | 起動するエージェント |
|---|---|---|
| `code-review` | 差分レビュー | bug-detector / security-reviewer / consistency-checker / quality-reviewer（4並列） |
| `project-review` | 全体 or 領域指定レビュー | タスクごとに上記4エージェントを並列実行 |
| `plan-review` | 実装前プランレビュー | plan-reviewer（単体） |
| `diff-summary` | 差分の要約（指摘なし、把握用） | diff-summarizer（単体） |
| `iterative-review` | 反復修正レビュー（Scope=このPR の指摘がなくなるまで修正→再レビュー） | code-review と同じ4エージェントを反復ごとに並列実行 |

### キーコンセプト

- **SKILL.md がオーケストレーター**: 各スキルがレビュー対象モードの決定 → ユーザー合意 → エージェント実行 → 指摘精査 → 出力の一連フローを定義
- **各エージェントは責務が厳密に分離**: 他エージェントの担当領域は意図的にスキップし、境界を越えるのはその問題が自分の担当領域に直接影響する場合のみ
- **エージェントのモデル選定**: 全エージェントが `model: opus` を使用
- **指摘精査プロセスが重要**: サブエージェントの結果をそのまま出力せず、起因判定（diff時）・実装妥当性検証・YAGNIチェック・優先度再判定を行う
- **指摘の評価は2軸（Priority × Scope）で全スキル共通**: `code-review` / `project-review` / `plan-review` および 5エージェント全てが採用。`Priority` = `Must` / `Should` / `Nit`、`Scope` = `このPR` / `別PR` / `別タスク`。`plan-review` (`plan-reviewer`) では Scope を「現プラン内で対応 / 関連 PR で対応 / 別プラン化 or 実装後に code-review で再検証」と読み替える

### レビューフロー（`code-review` 例）

1. レビュー対象モード決定（ステージ変更あり → `staged`、なし → `base-diff`、両方空 → 終了）
2. ベースブランチ決定（モード `base-diff` のみ：引数 > `gh auth status` > `gh pr view` > エラー案内）
3. プロジェクトのチェックツール（lint, typecheck等）を可能な範囲で実行
4. レビュー対象をユーザーに確認
5. 4エージェントを並列実行（各エージェントに diff、ファイル一覧、モード、ベースブランチ名、PRコメント等を渡す）
6. 結果を精査（ベースブランチ起因の除外、コンパイル可能性の検証、Priority・Scope の再判定）
7. 2軸（Priority=Must/Should/Nit × Scope=このPR/別PR/別タスク）でアクションプランとして出力

他スキル（`project-review` / `plan-review` / `iterative-review`）の具体的なフローは各 SKILL.md を参照（`iterative-review` は `code-review` の上位オーケストレーターとして動作）。

## プラグイン配布形式

Claude Code Marketplace プラグインとして配布。`marketplace.json` でオーナー・プラグイン一覧を定義し、`plugin.json` で個別プラグインのメタデータを管理する。

## 言語

このプロジェクトのドキュメント・エージェント定義はすべて日本語。
