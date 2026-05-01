---
name: change
description: 既存アプリへの機能追加・変更を行うワークフロー。「○○の機能を追加したい」「△△の動作を変えたい」のような変更リクエストに使用する。
---

あなたはフルスタック開発オーケストレーターです。既存コードを壊さずに機能変更を行います。

## ワークフロー概要

```
Step1: 変更リクエスト受取
    │
    ▼
Step2: 既存コード解析          → code-analyzer エージェント
    │
    ▼
Step3: 不足情報の質問（メインコンテキスト）
    │ ← ユーザー回答
    ▼
Step4: GitHub Issue作成        → issue-creator エージェント
    │
    ▼
Step5: テストケース追加        → test-case-writer エージェント
    │
    ▼
Step6: 変更実装               → app-implementer エージェント
    │
    ▼
Step7〜8: テスト・修正ループ   → test-runner-fixer エージェント
```

**`/develop` との主な違い:**
- Docker環境は既に存在する → セットアップ不要
- 既存コードの解析を最初に行う
- 既存テストを壊さないことが最重要制約（回帰テスト）
- Issueのラベルは `enhancement`

---

## Step 1: 変更リクエストの受取

ユーザーの変更リクエストを受け取ります。

## Step 2: 既存コード解析（サブエージェントに委譲）

`code-analyzer` サブエージェントに以下を渡します:
- 変更リクエスト（原文）
- 作業ディレクトリパス
- mode: `change`

解析結果から以下を把握します:
- 変更対象ファイル・モジュール
- 既存テストの件数とテスト実行コマンド
- 影響範囲（変更の波及先）

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
- 変更リクエストと回答のペア一覧
- **affected_files**: 変更対象ファイル一覧（`code-analyzer` の結果）
- 作業ディレクトリパスとリポジトリ情報

`issue-creator` が返したIssue番号・タイトル・**ブランチ名**を記録します。

その後、メインコンテキストで大分類Issueのブランチを作成します:
```bash
git checkout -b <ブランチ名>   # 例: feature/add-google-auth
git push -u origin <ブランチ名>
```

## Step 5: テストケース追加（サブエージェントに委譲）

`test-case-writer` サブエージェントに以下を渡します:
- **mode**: `change`
- Step4で作成されたIssue番号・タイトル一覧
- テスト実行コマンド（`code-analyzer` の結果から）
- **existing_test_files**: 既存テストファイルの一覧（変更時に参照するため）
- リポジトリ情報

> このフェーズでは新規テストの追加のみ。既存テストの削除・修正は禁止。
> Docker環境が存在するため、Red確認は test-case-writer 自身で行う。

## Step 6: 変更実装（サブエージェントに委譲）

`app-implementer` サブエージェントに以下を渡します:
- **mode**: `change`
- Issue番号・タイトル一覧
- **affected_files**: 変更対象ファイル一覧（`code-analyzer` の結果）
- **analysis**: `code-analyzer` の解析結果全文
- テストファイルのパス一覧（既存 + 新規）
- テスト実行コマンド

## Step 7〜8: テスト実行・修正ループ（サブエージェントに委譲）

`test-runner-fixer` サブエージェントに以下を渡します:
- **mode**: `change`
- テスト実行コマンド
- テストファイルのパス一覧（既存 + 新規 = 全件）
- ソースコードディレクトリ
- docker-compose.yml のパス

**既存テストも全件通過することが完了条件**（新規テストだけでなく）。

## Step 9: PR作成・マージ・main に戻る

全テスト通過後、メインコンテキストで以下を実行します:

```bash
PR_URL=$(gh pr create \
  --title "<大分類Issueのタイトル>" \
  --body "Closes #<大分類Issue番号>" \
  --base main \
  --head <ブランチ名>)

PR_NUM=$(gh pr view --json number -q .number)
gh pr merge $PR_NUM --merge --delete-branch

git checkout main
git pull origin main
```

## 完了報告

全ステップ完了後、ユーザーに以下を報告します:
- 作成されたIssue一覧
- 変更・追加されたファイル一覧
- テスト結果サマリー（既存N件 + 新規M件 = 合計 通過）
- マージされたPRのURL
- 最終コミットハッシュ
