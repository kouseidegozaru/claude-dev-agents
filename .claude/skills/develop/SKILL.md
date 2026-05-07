---
name: develop
description: 抽象的な命令からGitHub Issue作成・TDD実装・テスト自動修正まで行うフル開発ワークフロー。ユーザーが「○○○のアプリを作って」のような抽象的な指示を出した際に使用する。
---

あなたはフルスタック開発オーケストレーターです。以下のワークフローを厳密に守って開発を進めてください。

## ワークフロー概要

```
Step 1: 命令受取
Step 2: 不足情報の一括質問
Step 3: GitHub Issue作成 + ブランチ作成    → issue-creator
Step 4: テストケース先行作成（実行しない）  → test-case-writer
Step 5a: Docker環境構築（フェーズA）        → docker-initializer
Step 5b: 機能実装（フェーズB・Red→Green）   → app-implementer
Step 6〜8: テスト実行・自動修正ループ        → test-runner-fixer
Step 9: PR作成・マージ・main 復帰
```

サブエージェントを活用してメインコンテキストを節約することが最重要です。
ユーザーとのインタラクションが必要なStep 1〜2のみメインコンテキストで処理し、
Step 3以降は専用サブエージェントに委譲します。

---

## Step 1: 命令の受取と解析

ユーザーから受け取った命令を内部で分析します。以下の観点でギャップを洗い出してください:

- **技術選定**: 言語・フレームワーク・ライブラリの指定有無
- **実行環境**: CLI / Web / モバイル / デスクトップ
- **データ永続化**: DB種別・ストレージ方式
- **認証・権限**: ログイン機能・ロール管理の有無
- **外部連携**: API・サービス連携の有無
- **規模・性能**: 想定ユーザー数・レスポンス要件
- **UI/UX**: デザイン要件・アクセシビリティ
- **テスト**: カバレッジ目標・テストフレームワーク指定
- **デプロイ**: 実行環境・CI/CD要件
- **Docker**: ベースイメージ指定・公開ポート・追加サービス（DB・キャッシュ等）の構成

## Step 2: 不足情報の一括質問

分析結果をもとに、**一度だけ**まとめて質問します。後から追加質問は行いません。

フォーマット:
```
以下の情報を教えてください（実装に必要な項目のみ抜粋）:

1. [質問内容] （例: ...）
2. [質問内容] （例: ...）
...
```

ユーザーの回答を受け取ったら Step 3 に進みます。

## Step 3: GitHub Issue作成 + ブランチ作成（サブエージェント → メインコンテキスト）

`issue-creator` サブエージェントに以下を渡します:

- **mode**: `new`
- **label_hint**: `feature`（具体化Issueには `setup` / `docker` / `feature` / `test` のラベルを使い分け）
- 元の命令文（原文そのまま）
- Step 2 の質問と回答のペア一覧
- 作業ディレクトリパスとGitHubリポジトリ情報

**1スキル = 1大分類Issue 原則**:
`/develop` 1回の実行で扱う大分類Issueは **必ず1件のみ**。`issue-creator` の出力JSON `parent_issue` が1件であることを確認する。
複数機能を同時に求められた場合（例: 「ToDoアプリと家計簿アプリを作って」）は、最初の1機能のみ実行し、残りはユーザーへ「次の `/develop` で別途実行することを推奨します」と提案する。

`issue-creator` の出力JSON（CLAUDE.md「エージェント間契約 / `issue-creator` 出力スキーマ」）から以下を取り出して記録します:

- `parent_issue.number` — 大分類Issue番号
- `parent_issue.branch` — 作成するブランチ名（**単数**）
- `child_issues` — 具体化Issueの一覧（番号・ラベル・順序）

その後、メインコンテキストで大分類Issueのブランチを **1本だけ** 作成します:

```bash
git checkout -b <parent_issue.branch>   # 例: feature/user-auth
git push -u origin <parent_issue.branch>
```

`gh` コマンド失敗時は CLAUDE.md「標準回復手順 / gh コマンド失敗時」に従う。

## Step 4: テストケース先行作成（サブエージェントに委譲）

`test-case-writer` サブエージェントに以下を渡します:

- **mode**: `new` （※テストの実行はしない・Red確認は Step 5b で `app-implementer` が実施）
- 具体化Issue一覧（`child_issues`）。`docker` ラベルのIssueは `tests/test_docker_setup.sh`（スモークテスト）として作成される
- 技術スタック情報（Step 2 の回答から）
- テスト実行コマンド（`docker compose run --rm app <テストフレームワークのコマンド>` 形式）
- GitHubリポジトリURL

## Step 5a: Docker環境構築（サブエージェントに委譲・フェーズA）

`docker-initializer` サブエージェントに以下を渡します:

- **tech_stack**: 言語・フレームワーク・ライブラリ
- **docker_config**: ベースイメージ・公開ポート・追加サービス
- **setup_issues**: `setup` / `docker` ラベルが付いた具体化Issue一覧（番号・タイトル）
- **smoke_test_path**: `tests/test_docker_setup.sh`（Step 4 で `test-case-writer` が作成済み）
- **working_dir**: 作業ディレクトリパス

`docker-initializer` は以下を担当します:

1. プロジェクト構造作成
2. Dockerfile / docker-compose.yml / .env.example / .dockerignore 作成
3. `docker compose build` 成功確認
4. `tests/test_docker_setup.sh` 通過確認
5. `setup` / `docker` ラベルの具体化Issue ごとにコミット & Close

**失敗時の中断ルール**:
`docker-initializer` から `status: "docker_build_failed"` が返ってきた場合、
**Step 5b 以降には進まず**、ユーザーへ失敗ログと推定原因を報告して終了する。

## Step 5b: 機能実装（サブエージェントに委譲・フェーズB）

Step 5a が `status: "ready"` で完了した場合のみ進行。

`app-implementer` サブエージェントに以下を渡します:

- **mode**: `new`
- **issues**: `feature` ラベルの具体化Issueのみ（`setup` / `docker` は Step 5a で Close 済み）
- **test_files**: テストファイルのパス一覧（Step 4 の結果）
- **test_command**: `docker-initializer` から返却された `test_command`
- **compose_file**: `./docker-compose.yml`（および `./docker-compose.test.yml` がある場合は両方）
- **working_dir**: 作業ディレクトリパス

`app-implementer` は各 `feature` Issue について **個別に Red → Green** を回しながら順に実装します。

## Step 6〜8: テスト実行・自動修正ループ（サブエージェントに委譲）

`test-runner-fixer` サブエージェントに以下を渡します:

- **mode**: `new`
- **test_command**: Docker経由のテストコマンド
- **test_files**: テストファイルのパス一覧
- **src_dir**: ソースコードの主要ディレクトリ
- **compose_file**: `docker-compose.yml` のパス

サブエージェントは全テストが通るまで自律的に修正ループを繰り返します。
無限ループ防止条件（CLAUDE.md「`test-runner-fixer` escalation フォーマット」）に該当して escalation されてきた場合、
**PR作成・マージはせず**、ユーザーへ状況を報告して終了する。

## Step 9: PR作成・マージ・main に戻る

全テスト通過後、メインコンテキストで以下を実行します:

```bash
# PR作成
PR_URL=$(gh pr create \
  --title "<大分類Issueのタイトル>" \
  --body "Closes #<大分類Issue番号>" \
  --base main \
  --head <parent_issue.branch>)

# PR番号を取得してマージ
PR_NUM=$(gh pr view --json number -q .number)
gh pr merge $PR_NUM --merge --delete-branch

# main に戻って最新を取得
git checkout main
git pull origin main
```

`gh pr create` / `gh pr merge` 失敗時は CLAUDE.md「標準回復手順 / gh コマンド失敗時」に従う。

## 完了報告

全ステップ完了後、ユーザーに以下を報告します:

- 作成された大分類Issue・具体化Issue一覧（番号・タイトル）
- Docker環境（フェーズA）構築結果
- 実装されたファイル一覧
- テスト結果サマリー（通過数/総数）
- マージされたPRのURL
- 最終コミットハッシュ

複数機能を提案された場合（1スキル=1大分類Issue 原則により後送りされたもの）も併せて提示します。
