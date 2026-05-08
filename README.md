# super-code-review

Claude Code 用の包括的コードレビュー plugin。

5つのスキル（差分レビュー / プロジェクト全体レビュー / 実装前プランレビュー / 差分サマリー / 反復修正レビュー）と6つの専門サブエージェントを組み合わせ、ソフトウェア開発の各フェーズに応じたレビュー・把握支援を提供します。

## インストール

### マーケットプレイス経由（推奨）

```
/plugin marketplace add Aroza-inc/super-code-review
/plugin install super-code-review@super-code-review
```

### 手動クローンで使う場合

ネットワーク制約・社内ミラー利用・特定リビジョン固定など、マーケットプレイス経由での取得が使えない環境向け。リポジトリを任意の場所にクローンし、ローカルパスを marketplace として登録します。

```bash
git clone https://github.com/Aroza-inc/super-code-review.git ~/.claude/plugins/marketplaces/super-code-review
```

```
/plugin install super-code-review@super-code-review
```

## 前提条件

- [Claude Code](https://claude.ai/code) がインストール済みであること
- `gh` CLI（GitHub CLI）: **PRベースブランチの自動検出を使う場合のみ**必須。引数でベースブランチを明示指定する場合、または `git add` でステージ変更をレビューする場合は `gh` は不要

## 提供スキル

| スキル | 用途 | 起動するエージェント |
|---|---|---|
| `code-review` | 差分レビュー（ステージ変更 or ベースブランチとの diff） | 4並列 |
| `project-review` | コードベース全体 or 領域指定の包括レビュー | タスクごとに4並列 |
| `plan-review` | 実装着手前のプラン・設計案レビュー | plan-reviewer 単体 |
| `diff-summary` | 差分の概要・構成・アーキテクチャ影響の要約（指摘なし） | diff-summarizer 単体 |
| `iterative-review` | `code-review` を内包し、`Scope=このPR` の指摘がなくなるまで「修正→再レビュー」を自動反復 | code-review と同じ4並列を反復ごとに実行 |

「4並列」= bug-detector / security-reviewer / consistency-checker / quality-reviewer の同時実行。

## 使い方

```bash
# 差分レビュー（PRのベースブランチを自動検出）
/super-code-review:code-review

# 差分レビュー（ベースブランチを明示）
/super-code-review:code-review main

# 差分レビュー（ステージされた変更のみ。gh 不要）
git add <files>
/super-code-review:code-review

# プロジェクト全体レビュー
/super-code-review:project-review

# 領域指定でプロジェクトレビュー
/super-code-review:project-review "packages/auth/**"

# 実装前プランレビュー（プラン本文を直接渡す or ファイルパス指定）
/super-code-review:plan-review "docs/plans/auth-refactor.md"

# 差分のサマリー（指摘ではなく把握用）
/super-code-review:diff-summary

# 反復修正レビュー（指摘がなくなるまで自動修正→再レビュー）
/super-code-review:iterative-review
```

## プロジェクト構成

```
.claude-plugin/marketplace.json    # マーケットプレイス登録情報
plugins/super-code-review/
  .claude-plugin/plugin.json       # プラグインメタデータ
  skills/
    code-review/SKILL.md           # 差分レビュー
    project-review/SKILL.md        # プロジェクト全体 or 領域指定
    plan-review/SKILL.md           # 実装着手前のプラン
    diff-summary/SKILL.md          # 差分の要約（指摘なし）
    iterative-review/SKILL.md      # 反復修正レビュー
  agents/
    bug-detector.md                # 潜在バグ・ロジックエラー検出
    security-reviewer.md           # セキュリティ脆弱性検出（OWASP Top 10）
    consistency-checker.md         # 既存パターン・規約との整合性検証
    quality-reviewer.md            # 可読性・保守性・パフォーマンス評価
    plan-reviewer.md               # 実装プランと既存コードベースの整合性評価
    diff-summarizer.md             # 差分の要約生成（指摘なし）
```

## レビュー出力形式

レビュー結果は **Priority（優先度）× Scope（スコープ提案）** の2軸で整理されます（`code-review` / `project-review` / `plan-review` / `iterative-review` の各スキル共通）。`plan-review` のみ Scope を「現プラン内で対応 / 関連 PR で対応 / 別プラン化 or 実装後に code-review で再検証」と読み替えます。

### Priority（優先度）

| Priority | 意味 |
|---|---|
| Must | バグ、セキュリティ、クラッシュ、データ破損、規約違反でビルド失敗 |
| Should | 設計の一貫性、アンチパターン、明確なDRY違反、保守性低下、規約違反 |
| Nit | 命名・微小な可読性、好みの範囲、現状で実害なし |

### Scope（スコープ提案）

| Scope | 意味 |
|---|---|
| このPR | 当該diffに閉じて修正でき、副作用が小さい |
| 別PR | 既に進行中／予定されている関連PRがある |
| 別タスク | アプリ全体に波及（命名・a11y・共通基盤の統一等）、後続チケットの方が効率的 |

Overall Verdict は「Priority=Must かつ Scope=このPR」の指摘が1件以上あれば `REQUEST_CHANGES`、それ以外は `APPROVE`。Scope=`別PR` / `別タスク` には Scope Reason を必須で記載します（`iterative-review` の最終ステータスは `APPROVED / NEEDS_USER_INPUT / LIMIT_REACHED / ABORTED` の独自用語）。

## サブエージェント

| エージェント | 役割 | 主に呼ばれるスキル |
|---|---|---|
| bug-detector | 潜在バグ・ロジックエラー・エッジケース・型の不整合を検出 | code-review / project-review / iterative-review |
| security-reviewer | セキュリティ脆弱性（OWASP Top 10）を専門検出 | code-review / project-review / iterative-review |
| consistency-checker | 既存コードベースのパターン・規約との整合性を検証 | code-review / project-review / iterative-review |
| quality-reviewer | 可読性・保守性・DRY・パフォーマンスを評価 | code-review / project-review / iterative-review |
| plan-reviewer | 実装プランと既存コードベースの整合性を評価 | plan-review |
| diff-summarizer | 差分の概要・構成・アーキテクチャ影響を要約（指摘なし） | diff-summary |

## ライセンス

[MIT](./LICENSE)
