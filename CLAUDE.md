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
[Step 3] GitHub Issue作成         → issue-creator
[Step 4] テストケース先行作成     → test-case-writer  ※テスト実行はしない
[Step 5] 実装                     → app-implementer
         ├─ フェーズA: プロジェクト初期化 + Docker構築（TDDなし）
         └─ フェーズB: 機能実装（Red → Green サイクル）
[Step 6〜8] テスト・自動修正ループ → test-runner-fixer
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
[Step 5] テストケース追加         → test-case-writer  ※既存テストは変更しない
[Step 6] 変更実装                 → app-implementer   ※Docker再セットアップ不要
[Step 7〜8] テスト・自動修正ループ → test-runner-fixer ※既存テストも全件通過が条件
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
[Step 4] バグ再現テスト作成・失敗確認 → bug-reproducer  ※Docker実行で失敗を確認
[Step 5] バグ修正実装               → app-implementer ※Docker再セットアップ不要
[Step 6〜7] テスト・自動修正ループ   → test-runner-fixer
```

---

## サブエージェント一覧

| エージェント | 役割 | 使用スキル |
|-------------|------|-----------|
| `code-analyzer` | 既存コードを解析して変更箇所・影響範囲を特定 | `/change` `/fix` |
| `issue-creator` | GitHub Issueを設計・登録（feature/enhancement/bug対応） | 全スキル |
| `test-case-writer` | TDDテストを先行作成（実行はしない） | `/develop` `/change` |
| `bug-reproducer` | バグを再現する失敗テストを作成・実行確認 | `/fix` |
| `app-implementer` | 実装（新規開発はDockerセットアップ含む） | 全スキル |
| `test-runner-fixer` | テスト実行→自動修正→全通過まで繰り返し | 全スキル |

---

## コミット規約

| プレフィックス | 用途 |
|--------------|------|
| `feat:` | 機能実装（例: `feat: Add login (#3)`） |
| `test:` | テストケースの追加・修正 |
| `fix:` | バグ修正（例: `fix: Fix UTC comparison in auth (#12)`） |
| `chore:` | 設定・依存関係・Docker構成の変更 |

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
    test-case-writer.md  # TDDテスト先行作成（実行なし）
    bug-reproducer.md    # バグ再現テスト作成・失敗確認（/fix 専用）
    app-implementer.md   # アプリ実装
    test-runner-fixer.md # テスト実行・自動修正ループ
  settings.json          # 自動許可コマンドの設定
CLAUDE.md                # このファイル
```
