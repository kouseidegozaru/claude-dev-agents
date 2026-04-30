# 開発ワークフロー

このプロジェクトでは `/develop` スキルを使ったTDD駆動の開発ワークフローを採用しています。

## 使い方

```
/develop
```

その後、「○○○のアプリを作りたい」のように抽象的な命令を入力するだけです。
残りは自動で進みます。

---

## ワークフロー全体像

```
ユーザー命令
    │
    ▼
[Step 1] 命令の受取・分析（メインコンテキスト）
    │
    ▼
[Step 2] 不足情報の一括質問（メインコンテキスト）
    │  ← ユーザー回答
    ▼
[Step 3] GitHub Issue作成         → issue-creator エージェント
    │
    ▼
[Step 4] テストケース先行作成(TDD) → test-case-writer エージェント
    │
    ▼
[Step 5] アプリケーション実装      → app-implementer エージェント
    │
    ▼
[Step 6] テスト実行                ┐
    │                              │ test-runner-fixer エージェント
[Step 7] 自動修正                  │ （全通過まで繰り返し）
    │                              │
[Step 8] 全通過まで6,7を繰り返す   ┘
    │
    ▼
完了報告（Issue一覧・テスト結果・コミット履歴）
```

---

## サブエージェント一覧

| エージェント | 役割 | 使用ツール |
|-------------|------|-----------|
| `issue-creator` | 要件からGitHub Issueを設計・登録（Dockerセットアップ含む） | Bash (gh), Read |
| `test-case-writer` | IssueからTDDテストを先行作成 | Bash, Read, Write, Edit |
| `app-implementer` | Dockerfile・docker-compose.yml を含む実装全体 | Bash, Read, Write, Edit |
| `test-runner-fixer` | Docker経由でテスト実行→自動修正→全通過まで繰り返し | Bash, Read, Edit |

サブエージェントを分けることで、各フェーズのコンテキストを独立させ、
メインの会話コンテキストを節約しています。

---

## コミット規約

| プレフィックス | 用途 |
|--------------|------|
| `feat:` | 機能実装（Issue番号を末尾に付ける例: `feat: Add login (#3)`） |
| `test:` | テストケースの追加 |
| `fix:` | バグ修正・テスト失敗の修正 |
| `chore:` | 設定・依存関係の変更 |

---

## 前提条件

- `gh` コマンドがインストール済みで認証済みであること
- `docker` および `docker compose` がインストール済みであること
- GitHubリポジトリが作成済みでリモートが設定済みであること
- テストフレームワークの選定は `/develop` 実行時の質問で決定
- テストはすべて `docker compose run --rm app <テストコマンド>` 経由で実行される

---

## ファイル構成

```
.claude/
  skills/
    develop.md          # /develop スラッシュコマンドの定義
  agents/
    issue-creator.md    # GitHub Issue作成エージェント
    test-case-writer.md # テストケース先行作成エージェント
    app-implementer.md  # アプリ実装エージェント
    test-runner-fixer.md# テスト実行・自動修正エージェント
CLAUDE.md               # このファイル
```
