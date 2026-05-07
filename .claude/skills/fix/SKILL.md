---
name: fix
description: 既存アプリのバグを修正するワークフロー。「○○が動かない」「△△でエラーが出る」のようなバグ報告に使用する。Issue番号の指定も可能（例: /fix #12）。
---

あなたはバグ修正の専門家です。バグを再現→原因特定→修正→回帰確認の順で確実に直します。

## ワークフロー概要

```
Step 1: バグ説明の受取（Issue番号または自由記述）
Step 2: 既存コード解析             → code-analyzer
Step 3: GitHub Issue作成 + ブランチ作成 → issue-creator（Issue未存在の場合のみ）
Step 4: バグ再現テスト作成・Red確認  → bug-reproducer
Step 5: バグ修正実装                → app-implementer
Step 6〜7: テスト・修正ループ        → test-runner-fixer
Step 8: PR作成・マージ・main 復帰
```

**`/develop`・`/change` との主な違い:**

- Docker環境は既に存在する → セットアップ不要
- ユーザーへの質問は最小限（バグ内容が不明瞭な場合のみ確認）
- バグ再現テストを **先に作成して失敗を確認** してから修正する（Red確認は `bug-reproducer` の絶対責務）
- Issueのラベルは `bug`

---

## Step 1: バグ説明の受取

ユーザーからバグの説明を受け取ります。

**Issue番号が指定された場合** (`/fix #12` 等):

```bash
gh issue view 12 --json number,title,body,labels
```

でIssueの詳細を取得し、Step 2 へ進みます（Step 3 の Issue作成はスキップ）。

**自由記述の場合:**

バグ内容が明確でない場合のみ確認します（聞けることは1〜2点に絞る）。

**重要: 単に「教えてください」と聞かない**。ユーザーのバグ説明から推測できる典型的なシナリオを **3つの選択肢（うち1つを推奨）** として提示し、加えて **自由回答** も受け付けます。これにより「該当するものを選ぶだけ」で素早く回答できるようにします。

各質問は以下の構成を必ず守ること:

- **質問内容** — 何を確認したいか（1行）
- **選択肢 A / B / C** — 想定される典型ケース3つ。`★` を付けたものが Claude の推測（もっとも可能性が高いと考えるもの）
- **その他** — 自由記述用

確認できるテーマ（必要なものに絞る・最大2項目）:

- バグの再現手順（どう操作したら起きるか）
- 期待する動作と実際の動作
- エラーメッセージ・スタックトレース（あれば）

フォーマット例:
```
バグ再現に必要な情報を確認させてください。「A / B / C / その他」から選んでください。
`★` が付いた選択肢が、現時点で Claude が推測する有力ケースです。

1. **再現条件**
   - A. ログイン画面で空のパスワードを送信したとき
   - B. 既にログイン済みの状態で再度 /login にアクセスしたとき ★推測
   - C. パスワードに記号（@など）を含めたとき
   - その他: 自由記述

2. **発生環境**
   - A. ローカル開発環境のみ ★推測
   - B. 本番環境のみ
   - C. 両方
   - その他: 自由記述
```

**3択の選び方の指針:**

- **A / B / C はバグ説明から推測される現実的なケース** にする
- **★推測の根拠は明示しない**（バグ報告では「推奨」ではなく「もっとも怪しいケース」を ★ にする）
- **独断専行しない** — ユーザーが「その他」を選んだら必ず従う

内容が十分であれば確認せずに Step 2 へ進みます。

## Step 2: 既存コード解析（サブエージェントに委譲）

`code-analyzer` サブエージェントに以下を渡します:

- **request**: バグの説明（Issue詳細または自由記述）
- **mode**: `fix`
- **working_dir**: 作業ディレクトリパス

`code-analyzer` の出力JSON（CLAUDE.md「エージェント間契約 / `code-analyzer` 出力スキーマ」）を `analysis_json` として保持し、後続ステップで再利用します。

JSONから以下を取り出します:

- `affected_files` — バグ修正対象ファイル
- `bug_location` — バグの推定箇所（file・line・summary）
- `test_command` — テスト実行コマンド
- `existing_test_files` / `existing_test_count` — 既存テスト件数（回帰判定用）

## Step 3: GitHub Issue作成 + ブランチ作成（サブエージェント → メインコンテキスト）

### Issue作成（新規の場合のみ）

Issue番号が既に指定された場合は **Issue作成をスキップ**し、ブランチ作成のみ行います。

新規バグ報告の場合は `issue-creator` サブエージェントに以下を渡します:

- **mode**: `fix`
- **label_hint**: `bug`
- **original_instruction**: バグ説明
- **affected_files**: `analysis_json.affected_files`
- 1件の大分類Issueのみ作成（バグ1つ = 大分類Issue1件）
- 修正規模が小さい（1〜2コミットで完結）見込みなら **具体化Issue は作らず大分類Issueのみ** にしてよい。`issue-creator` の出力JSON `child_issues` は空配列 `[]` でよい

### ブランチ作成（メインコンテキスト）

Issue番号が確定したら（既存指定または新規作成問わず）、ブランチ名を以下のルールで決定します:

1. Issue 本文に `## ブランチ名` セクションがあればその値を使う（`issue-creator` 出力の `parent_issue.branch`）
2. なければ Issue タイトルから `fix/<kebab-case>` を自動生成
3. ローカル/リモートに同名ブランチが既に存在する場合は `fix/<summary>-<issue-number>` を使う

```bash
# 例
git checkout -b fix/login-crash       # 通常パターン
# 既存があれば: git checkout -b fix/login-crash-12

git push -u origin <ブランチ名>
```

`gh` / `git push` 失敗時は CLAUDE.md「標準回復手順 / gh コマンド失敗時」に従う。

## Step 4: バグ再現テスト作成（サブエージェントに委譲）

`bug-reproducer` サブエージェントに以下を渡します:

- **bug_description**: バグの説明（原文）
- **issue_number**: 大分類Issue番号
- **child_issue_number**: 具体化Issue番号（無い場合は省略可）
- **analysis_json**: `code-analyzer` のJSONブロック全体
- **test_command**: `analysis_json.test_command`
- **working_dir**: 作業ディレクトリパス

このエージェントは:

1. バグを再現する失敗テストを作成
2. Docker経由でテストを実行して **失敗を確認**（Red 確認は本エージェントの絶対責務）
3. 既存テストが全て通過していることも確認
4. コミット

`bug-reproducer` から `status: "bug_not_reproduced"` が返ってきた場合は **Step 5 以降には進まず**、ユーザーにバグ再現条件の追加情報を求めて終了する。

## Step 5: バグ修正実装（サブエージェントに委譲）

`app-implementer` サブエージェントに以下を渡します:

- **mode**: `fix`
- **issues**: バグIssue（大分類Issue 1件、具体化Issue があれば併せて）
- **test_files**: 既存全テスト + バグ再現テスト（Step 4 で作成された `tests/test_bug_<N>.py`）
- **affected_files**: `analysis_json.affected_files`
- **analysis_json**: `code-analyzer` のJSONブロック全体
- **test_command**: `analysis_json.test_command`
- **compose_file**: `analysis_json.compose_file`
- **修正の目標**: バグ再現テストを通過させること（既存テストの回帰なし）

## Step 6〜7: テスト実行・修正ループ（サブエージェントに委譲）

`test-runner-fixer` サブエージェントに以下を渡します:

- **mode**: `fix`
- **test_command**: Docker経由のテストコマンド
- **test_files**: 既存全テスト + バグ再現テスト
- **src_dir**: ソースコードディレクトリ
- **compose_file**: `analysis_json.compose_file`

**完了条件:**

- バグ再現テスト（`tests/test_bug_<N>.py`）が通過する
- 既存テストが全件通過する（`existing_test_count` 以上が通過 = リグレッションなし）

escalation された場合は **PR作成・マージはせず**ユーザーへ報告して終了する。

## Step 8: PR作成・マージ・main に戻る

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

`gh pr create` / `gh pr merge` 失敗時は CLAUDE.md「標準回復手順 / gh コマンド失敗時」に従う。

## 完了報告

全ステップ完了後、ユーザーに以下を報告します:

- 修正した大分類Issue番号とタイトル
- 修正ファイル一覧
- テスト結果サマリー（バグ再現テスト + 既存テスト 全件通過）
- マージされたPRのURL
- 最終コミットハッシュ
