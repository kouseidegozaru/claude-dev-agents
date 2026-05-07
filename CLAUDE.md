# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

# 開発ワークフロー

TDD駆動・Docker実行・GitHub Issue管理の3本柱で開発・変更・修正を自動化します。

## スキル一覧

| コマンド | 用途 | Docker要否 |
|---------|------|-----------|
| `/develop` | 新規アプリの開発（ゼロから） | セットアップ含む |
| `/change` | 既存アプリへの機能追加・変更 | 既存環境を使用 |
| `/fix` | 既存アプリのバグ修正 | 既存環境を使用 |

---

## `/develop` — 新規開発

```
ユーザー命令
    │
    ▼ メインコンテキスト
[Step 1] 命令の受取・分析
[Step 2] 不足情報の一括質問 ← ユーザー回答
    │
    ▼ サブエージェントへ委譲
[Step 3]  GitHub Issue作成        → issue-creator
[Step 4]  テストケース先行作成    → test-case-writer  ※テスト実行はしない
[Step 5a] Docker環境構築           → docker-initializer ※フェーズA
[Step 5b] 機能実装                 → app-implementer    ※フェーズB（Red→Green）
[Step 6〜8] テスト・自動修正ループ → test-runner-fixer
[Step 9] PR作成・マージ・main復帰
```

---

## `/change` — 機能変更

```
変更リクエスト
    │
    ▼ サブエージェント → メインコンテキスト
[Step 2] 既存コード解析            → code-analyzer
[Step 3] 不足情報の質問（後方互換・データ移行等） ← ユーザー回答
    │
    ▼ サブエージェントへ委譲
[Step 4] GitHub Issue作成（enhancement） → issue-creator
[Step 5] テストケース追加         → test-case-writer  ※既存テストは変更しない／Red確認は自身で実施
[Step 6] 変更実装                 → app-implementer   ※Docker再セットアップ不要
[Step 7〜8] テスト・自動修正ループ → test-runner-fixer ※既存テストも全件通過が条件
[Step 9] PR作成・マージ・main復帰
```

---

## `/fix` — バグ修正

```
バグ説明（または /fix #N でIssue指定）
    │
    ▼ サブエージェント → メインコンテキスト
[Step 2] 既存コード解析・バグ箇所特定 → code-analyzer
[Step 3] GitHub Issue作成（bug）       → issue-creator ※Issue既存の場合スキップ
    │
    ▼ サブエージェントへ委譲
[Step 4] バグ再現テスト作成・失敗確認 → bug-reproducer  ※Red確認は本エージェントの絶対責務
[Step 5] バグ修正実装                → app-implementer ※Docker再セットアップ不要
[Step 6〜7] テスト・自動修正ループ   → test-runner-fixer
[Step 8] PR作成・マージ・main復帰
```

---

## サブエージェント一覧

| エージェント | 役割 | 使用スキル |
|-------------|------|-----------|
| `code-analyzer` | 既存コードを解析して変更箇所・影響範囲を特定（JSON契約準拠） | `/change` `/fix` |
| `issue-creator` | GitHub Issueを2層構造で設計・登録（大分類Issue=ブランチ単位、具体化Issue=コミット単位） | 全スキル |
| `docker-initializer` | 新規プロジェクトのDocker環境を構築（フェーズA） | `/develop` |
| `test-case-writer` | TDDテストを先行作成（mode=newは実行なし／mode=changeはRed確認まで実施） | `/develop` `/change` |
| `bug-reproducer` | バグを再現する失敗テストを作成・Red確認（自身の絶対責務） | `/fix` |
| `app-implementer` | 機能実装・バグ修正（フェーズB専任） | 全スキル |
| `test-runner-fixer` | テスト実行→自動修正→全通過まで繰り返し／escalation仕様あり | 全スキル |

---

## コミット規約

| プレフィックス | 用途 |
|--------------|------|
| `feat:` | 機能実装（例: `feat: Add login (#3)`） |
| `test:` | テストケースの追加・修正 |
| `fix:` | バグ修正（例: `fix: Fix UTC comparison in auth (#12)`） |
| `chore:` | 設定・依存関係・Docker構成の変更 |

---

## 用語集

このリポジトリ内のドキュメント・エージェント定義で使う呼称を統一します。

- **大分類Issue** （= 親Issue / ブランチ単位Issue / Layer-1 Issue）
  1機能・1変更・1バグ修正に対応。`feature/*`・`fix/*` ブランチ1本に1対1対応する。
- **具体化Issue** （= 子Issue / コミット単位Issue / Layer-2 Issue）
  大分類Issueの下にぶら下がる、1コミット相当の作業単位。
- **2層構造Issue** = 大分類Issue 1件 + 具体化Issue N件 のセット。
- **フェーズA** = `/develop` のDocker環境構築（`docker-initializer`が担当）。
- **フェーズB** = 機能実装・変更・バグ修正（`app-implementer`が担当）。

---

## ブランチ・PR運用ルール

- **1大分類Issue = 1ブランチ = 1PR** — 大分類Issue 1件につきブランチを1本だけ作成する
- 具体化Issue（子Issue）ごとにブランチは切らず、**同一ブランチ上にコミット単位で積み上げる**
- 1つの大分類Issue完了 = 1つのPRが main にマージされる
- **`/develop` 1回の実行で扱う大分類Issueは原則1件**
  - 複数機能が必要な場合は機能ごとに `/develop` を別実行することをユーザーに提案
  - 自動で複数ブランチを並列処理しない
- ブランチ名は `issue-creator` 出力の `parent_issue.branch` を使用する（後述の契約参照）

---

## エージェント間契約

エージェント間で受け渡されるデータと、失敗時の標準回復手順を以下に集約します。
**すべてのエージェントはこのセクションを参照し、入出力スキーマを厳守すること。**

### `code-analyzer` 出力スキーマ

`code-analyzer` は人間可読なMarkdownレポート（既存の出力形式）に加え、**末尾に必ず以下のJSONブロックを出力する**:

```json
{
  "tech_stack": {
    "language": "Python 3.11",
    "framework": "FastAPI",
    "test_framework": "pytest"
  },
  "test_command": "docker compose run --rm app pytest tests/ -v",
  "compose_file": "./docker-compose.yml",
  "existing_test_files": ["tests/test_user.py", "tests/test_auth.py"],
  "existing_test_count": 13,
  "affected_files": ["src/models/user.py", "src/services/auth.py"],
  "bug_location": {
    "file": "src/services/auth.py",
    "line": 45,
    "summary": "tokenの有効期限チェックがUTCではなくlocal timeで比較されている"
  }
}
```

- `affected_files` は **常にPOSIX形式の相対パス文字列の配列**
- `bug_location` は **mode=fix のときのみ** 含める（mode=change では省略）
- 後続エージェントは「Markdown レポートは要約として読み、JSONブロックを機械的にパースする」

### `issue-creator` 出力スキーマ

`issue-creator` は人間可読な表形式に加え、**末尾に必ず以下のJSONブロックを出力する**:

```json
{
  "parent_issue": {
    "number": 1,
    "title": "ユーザー認証機能の追加",
    "branch": "feature/user-auth"
  },
  "child_issues": [
    {"number": 2, "title": "プロジェクト初期化", "label": "setup",   "order": 1},
    {"number": 3, "title": "Docker環境構築",     "label": "docker",  "order": 2},
    {"number": 4, "title": "ユーザーモデル定義", "label": "feature", "order": 3}
  ]
}
```

- **ブランチは大分類Issue 1件につき必ず1本のみ**（=`parent_issue.branch`）
- 具体化Issueは別ブランチを作らず、同一ブランチ上で順番にコミットする
- **子Issue一覧の追記は `issue-creator` 自身の必須責務**で、すべての具体化Issue作成後に必ず大分類Issueの本文末尾を `gh issue edit` で更新する。追記が完了するまで `issue-creator` は出力を返さない
- mode=fix で既存Issueを再利用する場合は `child_issues=[]` （空配列）を返してよい

### 標準回復手順

#### gh コマンド失敗時（pr create / pr merge / issue create / issue edit）

1. `gh auth status` で認証確認
2. `gh pr list --head <branch>` 等で重複の有無を確認（既にPR/Issueがあれば既存を使用）
3. ネットワークエラー（exit code != 0 かつ `could not resolve` / `connection reset` 等）→ **60秒待機して1回だけ再試行**
4. それでも失敗 → 上位スキルに「gh操作失敗」として escalation。**手作業マージ用の代替コマンドをユーザーに提示**:
   ```
   gh pr create --title "..." --body "..." --base main --head <branch>
   gh pr merge <PR#> --merge --delete-branch
   ```
5. ユーザーへの通知後、ワークフローは中断する（**勝手に手動マージはしない**）

#### docker compose build 失敗時

- フェーズA（`docker-initializer`）の build 失敗: Dockerfile・依存関係ファイルを **最大2回まで自動修正**、それでも失敗なら escalation
- フェーズB（既存環境）の build 失敗: `test-runner-fixer` のパターン6（環境・依存関係エラー）に従う

#### `test-runner-fixer` escalation フォーマット

`test-runner-fixer` が修正不能と判断した場合、上位スキルに以下のJSONを返す:

```json
{
  "status": "unresolved",
  "reason": "same_error_3_times | loop_limit_exceeded | regression_loop | scope_diffusion",
  "loop_count": 5,
  "failing_tests": [
    {"file": "tests/test_x.py", "name": "test_y", "error_summary": "AssertionError: ..."}
  ],
  "last_attempt_diff_summary": "...",
  "suggested_next_action": "..."
}
```

上位スキルはこれを受けたら **PR作成・マージはせず**、ユーザーに状況を報告して終了する。

---

## 前提条件

- `gh` コマンドがインストール済みで認証済みであること
- `docker` および `docker compose` がインストール済みであること
- GitHubリポジトリが作成済みでリモートが設定済みであること
- テストはすべて `docker compose run --rm app <テストコマンド>` 経由で実行される

---

## スキルの呼び出し方

スキルは `Skill` ツールで呼び出します。ユーザーが `/develop`・`/change`・`/fix` と入力した場合、対応するスキルを実行してください。

## 自動許可コマンド

`.claude/settings.json` で以下のコマンドが自動許可されています:

- ファイル操作: `Read`・`Write`・`Edit`・`grep`・`ls`・`find`・`cat`・`head`・`tail`・`wc`・`file`・`stat`・`mkdir`・`cp`・`touch`
- バージョン管理: `git`・`gh`
- コンテナ: `docker`
- テスト・ビルド: `npx vitest`・`npx vite build`
- Web: `WebFetch`・`WebSearch`

---

## ファイル構成

```
.claude/
  skills/
    develop/
      SKILL.md           # /develop — 新規開発
    change/
      SKILL.md           # /change  — 機能変更
    fix/
      SKILL.md           # /fix     — バグ修正
  agents/
    code-analyzer.md     # 既存コード解析（/change・/fix 共通）
    issue-creator.md     # GitHub Issue作成（feature/enhancement/bug 対応）
    docker-initializer.md # /develop のDocker環境構築（フェーズA）
    test-case-writer.md  # TDDテスト先行作成
    bug-reproducer.md    # バグ再現テスト作成・Red確認（/fix 専用）
    app-implementer.md   # 機能実装・バグ修正（フェーズB専任）
    test-runner-fixer.md # テスト実行・自動修正ループ
  settings.json          # 自動許可コマンドの設定
CLAUDE.md                # このファイル
```

---

## 付録A: Docker設定テンプレート（共通参照）

`docker-initializer` および `app-implementer` がプロジェクトを初期化する際は、
プロジェクトの技術スタック・要件に合わせて以下のテンプレートをカスタマイズして使用する。

### Dockerfile（マルチステージビルド）

```dockerfile
# --- build stage ---
FROM <base-image> AS builder
WORKDIR /app
COPY <依存関係ファイル> .
RUN <依存関係インストール>
COPY . .
RUN <ビルドコマンド（あれば）>

# --- runtime stage ---
FROM <slim-image> AS runtime
WORKDIR /app
COPY --from=builder /app .
EXPOSE <port>
CMD ["<起動コマンド>"]
```

### docker-compose.yml（DB ありの場合）

```yaml
services:
  app:
    build: .
    ports:
      - "${APP_PORT:-8080}:8080"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app          # 開発時のホットリロード用

  db:
    image: postgres:16-alpine   # または mysql:8 / mongo:7 等
    environment:
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  db_data:
```

### docker-compose.yml（DB なし・CLIアプリ等の場合）

```yaml
services:
  app:
    build: .
    env_file: .env
    volumes:
      - .:/app
```

### docker-compose.test.yml（テスト用オーバーライド）

```yaml
# docker compose -f docker-compose.yml -f docker-compose.test.yml run --rm app
services:
  app:
    # volumes の上書き: Compose はリストを置換するため [] で開発用マウントを除去できる
    volumes: []
    environment:
      DB_NAME: test_db   # テスト用DB名に上書き
    # command は指定しない — run 時に引数で渡す

  db:
    environment:
      POSTGRES_DB: test_db
```

### .env.example

```
APP_PORT=8080
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=changeme
DB_HOST=db
DB_PORT=5432
```

### .dockerignore

```
.git
.env
node_modules/    # Node.jsの場合
__pycache__/     # Pythonの場合
*.pyc
.pytest_cache/
dist/
```

### スモークテストスクリプト（`tests/test_docker_setup.sh`）

```bash
#!/usr/bin/env bash
set -e
docker compose build
docker compose run --rm app echo "container ok"
docker compose down
echo "Docker smoke test passed"
```
