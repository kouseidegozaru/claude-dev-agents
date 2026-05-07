# claude-dev-agents

Claude Code 上で動作する **TDD駆動・Docker実行・GitHub Issue管理** の自動開発エージェント群。
3つのスラッシュコマンドと7つのサブエージェントで、新規開発・機能変更・バグ修正を一気通貫で自動化します。

---

## 取得方法

```bash
curl -L https://github.com/kouseidegozaru/claude-dev-agents/archive/refs/heads/main.tar.gz -o agents.tar.gz
tar xzf agents.tar.gz
# 取得した .claude/ と CLAUDE.md を対象プロジェクト直下に配置
```

---

## 前提条件

| 必須コマンド | 用途 |
|------------|------|
| `gh`（認証済み） | GitHub Issue・PR の作成/更新 |
| `docker` + `docker compose` | テスト実行・本番ビルド共通 |
| `git`（リモート設定済み） | ブランチ作成・PR 連携 |

リポジトリは GitHub 上に作成済みでリモートが `origin` として設定されている必要があります。

---

## スラッシュコマンド

| コマンド | 用途 | Docker要否 |
|---------|------|-----------|
| `/develop [プロンプト]` | 新規アプリの開発（ゼロから） | セットアップ含む |
| `/change [プロンプト]` | 既存アプリへの機能追加・変更 | 既存環境を使用 |
| `/fix [プロンプト or #N]` | 既存アプリのバグ修正 | 既存環境を使用 |

### `/develop` — 新規開発

「○○のアプリを作って」のような抽象的な指示から、不足情報を一括質問 → GitHub Issue（2層構造）作成 → TDDテスト先行 → Docker環境構築 → 機能実装 → テスト自動修正 → PR作成・マージまで実行します。

```
/develop FastAPIでシンプルなTodo APIを作って
```

### `/change` — 機能変更

既存コードを解析した上で、新規テスト追加 → 変更実装 → 全テスト通過確認 → PR作成・マージまで実行します。既存テスト全件通過が必須条件（リグレッションを許さない）。

```
/change ログイン画面にGoogle認証を追加したい
```

### `/fix` — バグ修正

バグ再現テストを **先に作成して失敗（Red）を確認** してから修正します。Issue番号指定も可能。

```
/fix トークンの有効期限チェックがUTCで動いていない
/fix #12     # 既存Issueから着手する場合
```

---

## ワークフロー全体像

詳細は `CLAUDE.md` の各スキルセクションおよび `## エージェント間契約` を参照してください。

```
/develop:
  Step 1〜2: 命令受取・質問
  Step 3:    Issue作成 + ブランチ作成     → issue-creator
  Step 4:    テスト先行作成（実行しない）  → test-case-writer
  Step 5a:   Docker環境構築（フェーズA）   → docker-initializer
  Step 5b:   機能実装（フェーズB）         → app-implementer
  Step 6〜8: テスト・自動修正ループ        → test-runner-fixer
  Step 9:    PR作成・マージ・main 復帰

/change:
  Step 1〜3: 解析・質問                    → code-analyzer
  Step 4:    Issue作成 + ブランチ作成      → issue-creator
  Step 5:    テスト追加・Red確認           → test-case-writer
  Step 6:    変更実装                      → app-implementer
  Step 7〜8: テスト・修正ループ             → test-runner-fixer
  Step 9:    PR作成・マージ・main 復帰

/fix:
  Step 1〜3: 解析・Issue確認               → code-analyzer / issue-creator
  Step 4:    バグ再現テスト・Red確認       → bug-reproducer
  Step 5:    バグ修正                      → app-implementer
  Step 6〜7: テスト・修正ループ             → test-runner-fixer
  Step 8:    PR作成・マージ・main 復帰
```

---

## サブエージェント一覧

| エージェント | 役割 | 使用スキル |
|-------------|------|-----------|
| `code-analyzer` | 既存コード解析・JSON契約準拠 | `/change` `/fix` |
| `issue-creator` | 2層構造Issue作成（大分類 + 具体化） | 全スキル |
| `docker-initializer` | Docker環境構築（フェーズA） | `/develop` |
| `test-case-writer` | TDDテスト先行作成（mode=newは実行なし） | `/develop` `/change` |
| `bug-reproducer` | バグ再現失敗テスト・Red確認 | `/fix` |
| `app-implementer` | 機能実装・バグ修正（フェーズB専任） | 全スキル |
| `test-runner-fixer` | テスト実行→自動修正ループ・escalation仕様あり | 全スキル |

---

## 既知の制約

### 1スキル = 1大分類Issue 原則

`/develop` 1回の実行で扱う大分類Issueは **必ず1件**。複数機能を提案された場合は最初の1件のみ実行し、残りは「次の `/develop` を別途実行することを推奨します」とユーザーに提示します。

### `test-runner-fixer` のループ上限

無限ループ防止のため、以下のいずれかを満たすと自動的に escalation して終了します（コミット・PRは作成しません）:

- 同一エラーシグネチャが3回連続発生
- 総ループ回数が8回到達
- 修正によりテスト通過数が減少 → 同一ファイル再修正
- mode=fix で `affected_files` の範囲外への修正拡散

### Docker build 失敗時の中断

`/develop` の `docker-initializer` で `docker compose build` が **2回連続失敗** すると、機能実装フェーズには進まず、ユーザーへ失敗ログと推定原因を報告して終了します。

### gh コマンド失敗時

`gh pr create` / `gh pr merge` / `gh issue create` 等が失敗した場合、認証確認 → 1回だけ再試行 → それでも失敗ならユーザーへ手動マージ用の代替コマンドを提示して中断します。**勝手に手動マージはしません。**

---

## ファイル構成

```
.claude/
  skills/
    develop/SKILL.md       # /develop — 新規開発
    change/SKILL.md        # /change  — 機能変更
    fix/SKILL.md           # /fix     — バグ修正
  agents/
    code-analyzer.md
    issue-creator.md
    docker-initializer.md
    test-case-writer.md
    bug-reproducer.md
    app-implementer.md
    test-runner-fixer.md
  settings.json            # 自動許可コマンド
CLAUDE.md                  # ワークフロー仕様・エージェント間契約・Docker設定テンプレート
```

詳細仕様・JSONスキーマ・標準回復手順は `CLAUDE.md` を参照してください。
