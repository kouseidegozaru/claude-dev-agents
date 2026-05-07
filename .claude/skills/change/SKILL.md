---
name: change
description: 既存アプリへの機能追加・変更を行うワークフロー。「○○の機能を追加したい」「△△の動作を変えたい」のような変更リクエストに使用する。
---

あなたはフルスタック開発オーケストレーターです。既存コードを壊さずに機能変更を行います。

## ワークフロー概要

```
Step 1: 変更リクエスト受取
Step 2: 既存コード解析             → code-analyzer
Step 3: 不足情報の質問（メインコンテキスト）
Step 4: GitHub Issue作成 + ブランチ作成 → issue-creator
Step 5: テストケース追加（Red確認込み） → test-case-writer
Step 6: 変更実装                    → app-implementer
Step 7〜8: テスト・修正ループ        → test-runner-fixer
Step 9: PR作成・マージ・main 復帰
```

**`/develop` との主な違い:**

- Docker環境は既に存在する → セットアップ不要（フェーズA は無し）
- 既存コードの解析（`code-analyzer`）を最初に行う
- 既存テストを壊さないことが最重要制約（回帰テスト）
- Issueのラベルは `enhancement`

---

## Step 1: 変更リクエストの受取

ユーザーの変更リクエストを受け取ります。

## Step 2: 既存コード解析（サブエージェントに委譲）

`code-analyzer` サブエージェントに以下を渡します:

- **request**: 変更リクエスト（原文）
- **mode**: `change`
- **working_dir**: 作業ディレクトリパス

`code-analyzer` の出力JSON（CLAUDE.md「エージェント間契約 / `code-analyzer` 出力スキーマ」）を `analysis_json` として保持し、後続ステップで再利用します。

JSONから以下を取り出します:

- `tech_stack` — 言語・フレームワーク・テストフレームワーク
- `test_command` — テスト実行コマンド
- `compose_file` — docker-compose.yml のパス
- `existing_test_files` / `existing_test_count` — 既存テストの件数（回帰判定用）
- `affected_files` — 変更対象ファイル

## Step 3: 不足情報の一括質問

解析結果をもとに、**一度だけ**まとめて質問します。

変更特有の確認事項:

- **後方互換性**: 既存APIを変更する場合、破壊的変更を許容するか
- **データ移行**: DB スキーマを変更する場合、既存データの扱い
- **段階的リリース**: 機能フラグ・フィーチャートグルが必要か
- **影響範囲の確認**: 解析で判明した波及先の変更可否

ユーザーの回答を受け取ったら Step 4 に進みます。

## Step 4: GitHub Issue作成 + ブランチ作成（サブエージェント → メインコンテキスト）

`issue-creator` サブエージェントに以下を渡します:

- **mode**: `change`
- **label_hint**: `enhancement`
- **original_instruction**: 変更リクエスト原文
- **requirements**: Step 3 の質問と回答のペア一覧
- **affected_files**: `analysis_json.affected_files`（`code-analyzer` の結果）
- 作業ディレクトリパスとリポジトリ情報

`issue-creator` の出力JSON `parent_issue` から **ブランチ名（単数）** を取得します。
**1スキル = 1大分類Issue 原則**: 出力JSONの `parent_issue` は1件のみ。複数返ってきた場合は最初の1件のみ使用し、残りはユーザーへ提案として提示する。

メインコンテキストで大分類Issueのブランチを作成します:

```bash
git checkout -b <parent_issue.branch>   # 例: feature/add-google-auth
git push -u origin <parent_issue.branch>
```

`gh` 失敗時は CLAUDE.md「標準回復手順 / gh コマンド失敗時」に従う。

## Step 5: テストケース追加（サブエージェントに委譲）

`test-case-writer` サブエージェントに以下を渡します:

- **mode**: `change`
- **issues**: 具体化Issue一覧
- **test_command**: `analysis_json.test_command`
- **existing_test_files**: `analysis_json.existing_test_files`（既存テストへの追記時に参照）
- リポジトリ情報

このフェーズでは新規テストの追加のみ。既存テストの削除・修正は禁止。
**Red 確認は `test-case-writer` 自身が実施する**（Docker環境が存在するため）。

## Step 6: 変更実装（サブエージェントに委譲）

`app-implementer` サブエージェントに以下を渡します:

- **mode**: `change`
- **issues**: 具体化Issue一覧
- **affected_files**: `analysis_json.affected_files`
- **analysis_json**: `code-analyzer` のJSONブロック全体
- **test_files**: テストファイルのパス一覧（既存 + 新規）
- **test_command**: `analysis_json.test_command`
- **compose_file**: `analysis_json.compose_file`

## Step 7〜8: テスト実行・修正ループ（サブエージェントに委譲）

`test-runner-fixer` サブエージェントに以下を渡します:

- **mode**: `change`
- **test_command**: Docker経由のテストコマンド
- **test_files**: テストファイルのパス一覧（既存 + 新規 = 全件）
- **src_dir**: ソースコードディレクトリ
- **compose_file**: `analysis_json.compose_file`

**既存テストも全件通過することが完了条件**（`existing_test_count` 以上が通過）。
escalation された場合は **PR作成・マージはせず**ユーザーへ報告して終了する。

## Step 9: PR作成・マージ・main に戻る

全テスト通過後、メインコンテキストで以下を実行します:

```bash
PR_URL=$(gh pr create \
  --title "<大分類Issueのタイトル>" \
  --body "Closes #<大分類Issue番号>" \
  --base main \
  --head <parent_issue.branch>)

PR_NUM=$(gh pr view --json number -q .number)
gh pr merge $PR_NUM --merge --delete-branch

git checkout main
git pull origin main
```

`gh pr create` / `gh pr merge` 失敗時は CLAUDE.md「標準回復手順 / gh コマンド失敗時」に従う。

## 完了報告

全ステップ完了後、ユーザーに以下を報告します:

- 作成されたIssue一覧
- 変更・追加されたファイル一覧
- テスト結果サマリー（既存N件 + 新規M件 = 合計 通過）
- マージされたPRのURL
- 最終コミットハッシュ
