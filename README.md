# super-code-review

Claude Code 用の包括的コードレビュー plugin。

6つのスキル（差分レビュー / プロジェクト全体レビュー / 実装前プランレビュー / 差分サマリー / 反復修正レビュー / PRレビュー＆自動承認）と6つの専門サブエージェントを組み合わせ、ソフトウェア開発の各フェーズに応じたレビュー・把握支援を提供します。

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
- `gh` CLI（GitHub CLI）: **PRベースブランチの自動検出を使う場合、および `pr-review` を使う場合**に必須。引数でベースブランチを明示指定する場合、または `git add` でステージ変更をレビューする場合は `gh` は不要

## 提供スキル

| スキル | 用途 | 起動するエージェント |
|---|---|---|
| `code-review` | 差分レビュー（ステージ変更 or ベースブランチとの diff） | 下記4種から差分に応じて動的に起動（0体〜複数体） |
| `project-review` | コードベース全体 or 領域指定の包括レビュー | タスクごとに必要な観点を並列 |
| `plan-review` | 実装着手前のプラン・設計案レビュー | plan-reviewer 単体 |
| `diff-summary` | 差分の概要・構成・アーキテクチャ影響の要約（指摘なし） | diff-summarizer 単体 |
| `iterative-review` | `code-review` を内包し、`Scope=このPR` の指摘がなくなるまで「修正→再レビュー」を自動反復 | code-review と同じ動的起動を反復ごとに実行 |
| `pr-review` | 他者のPRをローカルに取得して `code-review` を実行し、指摘をPRにコメント。承認条件を満たせば自動 Approve | code-review と同じ動的起動 |

差分レビュー系は bug-detector / security-reviewer / consistency-checker / quality-reviewer の4種を使う。起動数は固定せず、差分の内容（例: ドキュメントのみなら consistency-checker だけ）と量（大きい差分はチャンクに分けて同じ観点を複数体）からオーケストレーターが決め、レビュー前に「起動計画」として提示する。自明な差分ではエージェントを起動せず直接確認することもある。

## 使い方

```bash
# 差分レビュー（PRのベースブランチを自動検出）
/super-code-review:code-review

# 差分レビュー（ベースブランチを明示）
/super-code-review:code-review main

# 差分レビュー（要件・仕様のファイルと照らし合わせる。PR 本文とリンク先の issue は自動で読む）
/super-code-review:code-review --spec docs/specs/login.md

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

# 他者のPRをレビューしてコメント投稿、問題なければ自動 Approve（ブランチ名 / PR番号 / URL）
/super-code-review:pr-review feature/foo

# 投稿・Approve せずに結果だけ確認
/super-code-review:pr-review 123 --dry-run
```

`pr-review` は Must 指摘なしなどの承認条件をすべて満たす場合のみ Approve し、それ以外はコメント投稿のみで終えます（Request changes は付けません。条件の詳細は `skills/pr-review/SKILL.md` の Step 4）。CI の結果は承認条件に含めず、参考情報としてコメントに記載します（マージ前の CI の強制はブランチ保護に委ねます）。実行には追跡ファイルに未コミットの変更がない状態が必要で、終了時に元のブランチへ戻ります。PRブランチのスクリプト（lint・テスト・git hooks）はローカルで実行しません。

## プロジェクト構成

```
.claude-plugin/marketplace.json    # マーケットプレイス登録情報
plugins/super-code-review/
  .claude-plugin/plugin.json       # プラグインメタデータ
  skills/
    code-review/SKILL.md           # 差分レビュー
    code-review/references/        # 共通ルール・PR コメント投稿ルール等（他スキル・エージェントも参照）
    project-review/SKILL.md        # プロジェクト全体 or 領域指定
    plan-review/SKILL.md           # 実装着手前のプラン
    diff-summary/SKILL.md          # 差分の要約（指摘なし）
    iterative-review/SKILL.md      # 反復修正レビュー
    pr-review/SKILL.md             # 他者PRのレビュー・コメント投稿・自動承認
  agents/
    bug-detector.md                # 潜在バグ・ロジックエラー検出
    security-reviewer.md           # セキュリティ脆弱性検出（OWASP Top 10）
    consistency-checker.md         # 既存パターン・規約との整合性検証
    quality-reviewer.md            # 可読性・保守性・パフォーマンス評価
    plan-reviewer.md               # 実装プランと既存コードベースの整合性評価
    diff-summarizer.md             # 差分の要約生成（指摘なし）
```

## レビュー出力形式

`code-review` の出力は、毎回すべての指摘を人が読まなくて済むように絞っています。

| 節 | 中身 |
|---|---|
| ヘッダ | Verdict、変更の規模、**検証**（lint・テストの結果。すべて成功なら1行）、**要件**（何と照らし合わせたか） |
| まとめ | 何をする変更か、マージしてよいか、人が決めることが何件あるかを2〜4文で |
| 確認してほしいこと | 人の判断が必要な指摘だけを詳しく（仕様の解釈、構成から見直すべき変更、破壊的変更など）。選択肢とおすすめを添える |
| AI で直せる指摘 | 1行ずつの表。「F-3 を直して」のように番号で修正を頼める |
| 別タスク候補 | このPRの外で対応するものを1行ずつ |

除外した指摘、エージェントごとの内訳、件数の集計表は出しません。

### 要件との照合

PR 本文、PR 本文からリンクされた GitHub の issue / PR（最大5件・1段だけ）、`--spec` で渡したファイルを要件として読み、抜け（Missing）・作りすぎ（Extra）・取り違え（Misunderstood）がないかを確かめます。見つかったものはすべて「確認してほしいこと」に出します。要件が見つからなければ何も出さずに通常のレビューだけを行います。GitHub 以外の URL は開きません。

要件テキストは指摘を増やす根拠にだけ使い、コードから見つかった指摘を減らす根拠には使いません。


レビュー結果は **Priority（優先度）× Scope（スコープ提案）** の2軸で整理されます（`code-review` / `project-review` / `plan-review` / `iterative-review` / `pr-review` の各スキル共通）。`plan-review` のみ Scope を「現プラン内で対応 / 関連 PR で対応 / 別プラン化 or 実装後に code-review で再検証」と読み替えます。

### Priority（優先度）

| Priority | 意味 |
|---|---|
| Must | バグ、セキュリティ、クラッシュ、データ破損、要件と実装の食い違い、規約違反でビルド失敗 |
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
| bug-detector | 潜在バグ・ロジックエラー・エッジケース・型の不整合を検出 | code-review / project-review / iterative-review / pr-review |
| security-reviewer | セキュリティ脆弱性（OWASP Top 10）を専門検出 | code-review / project-review / iterative-review / pr-review |
| consistency-checker | 既存コードベースのパターン・規約との整合性を検証 | code-review / project-review / iterative-review / pr-review |
| quality-reviewer | 可読性・保守性・DRY・パフォーマンスを評価 | code-review / project-review / iterative-review / pr-review |
| plan-reviewer | 実装プランと既存コードベースの整合性を評価 | plan-review |
| diff-summarizer | 差分の概要・構成・アーキテクチャ影響を要約（指摘なし） | diff-summary |

## ライセンス

[MIT](./LICENSE)
