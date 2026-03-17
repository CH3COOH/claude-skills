---
name: git-rebase
description: 指定範囲のコミットを1つに統合（squash）するためのinteractive rebase手順とコミットメッセージを生成する。「コミットをまとめたい」「squashしたい」「rebaseでコミット整理」「履歴をきれいにしたい」「コミット統合」などの依頼時に使用。
allowed-tools: Bash
argument-hint: "<from-commit(古い方)> <to-commit(新しい方)>"
metadata:
  author: Kenji Wada
  version: 1.0.0
  category: git
---

# Git Interactive Rebase スキル

指定範囲のコミットを1つに統合（squash）するための手順とコミットメッセージを生成します。

**役割の分担:**
- Claude が行う: `git log` / `git status` / `git branch` などの読み取りコマンドを実行し、状態を把握する
- ユーザーが行う: `git rebase -i` などの履歴変更コマンドを実行する

---

# 使用例

## 例1: 直近3コミットをまとめる
ユーザー: 「直近3つのコミットを1つにまとめたい」
実行: `git-rebase HEAD~3 HEAD`
結果: 3コミットが1つのコミットに統合される

## 例2: 特定範囲のコミットを統合
ユーザー: 「abc1234からdef5678までのコミットをsquashして」
実行: `git-rebase abc1234 def5678`
結果: 指定範囲のコミットが統合され、適切なメッセージが提案される

## 例3: ブランチの作業をまとめる
ユーザー: 「featureブランチの全コミットを1つにしたい」
実行: mainブランチとの分岐点を特定し、rebaseコマンドを提示
結果: ブランチ上の全コミットが1つに統合される

---

# 事前バリデーション

以下を Bash ツールで確認し、問題があれば手順の提示前にユーザーへ案内すること：

| チェック項目 | 確認コマンド | 問題がある場合の対応 |
|-------------|-------------|-------------------|
| 引数の数 | - | `<from-commit>` と `<to-commit>` の両方を教えてもらう |
| 作業ディレクトリ | `git status --porcelain` | 未コミットの変更があれば先に commit または stash するよう案内 |
| rebase中でないこと | `.git/rebase-merge` / `.git/rebase-apply` の存在を確認 | 進行中なら `--abort` または `--continue` で解決するよう案内 |
| コミットの存在 | `git rev-parse --verify <from-commit>` / `<to-commit>` | 存在しないハッシュはユーザーに再確認 |
| コミット順序 | `git merge-base --is-ancestor <from> <to>` | from が to の祖先でない場合は順序を入れ替えるよう案内 |
| push済みか否か | `git branch -r --contains <from-commit>` | push済みであれば force push が必要な旨を事前に警告 |

**問題が確認された場合は、以降の処理を行わずユーザーに解決を依頼すること。**

---

# 手順

## Step 1: 現在の状態を確認（Claude が実行）

```bash
git status
git branch --show-current
git log --oneline -20
```
- detached HEAD 状態の場合は警告を表示

## Step 2: 対象コミットの内容を分析（Claude が実行）

```bash
git log --oneline '<from-commit>^'..'<to-commit>'
git log --stat '<from-commit>^'..'<to-commit>'
git log --merges '<from-commit>^'..'<to-commit>'
```
- 統合対象のコミット数・変更内容・コミットメッセージの傾向を把握する
- マージコミットが含まれる場合は以下を案内：
  > ⚠️ 対象範囲にマージコミットが含まれています。通常のrebaseではマージの履歴が失われます。
  > マージ履歴を保持する場合は `git rebase -i --rebase-merges '<from-commit>^'` を使用してください。

## Step 3: Interactive rebase コマンドをユーザーに提示

### from-commit がリポジトリの最初のコミット（root commit）の場合
```bash
git rebase -i --root
```

### 通常の場合
```bash
git rebase -i '<from-commit>^'
```

## Step 4: エディタでの操作手順を説明

エディタが開いたら、以下のように編集する：

**変更前：**
```
pick abc1234 最初のコミット
pick def5678 2番目のコミット
pick ghi9012 3番目のコミット
```

**変更後：**
```
pick abc1234 最初のコミット
squash def5678 2番目のコミット
squash ghi9012 3番目のコミット
```

- 最初のコミットは `pick` のまま維持
- 2つ目以降のコミットを `squash`（または `s`）に変更
- 保存してエディタを閉じる

## Step 5: 統合後のコミットメッセージを提案

Step 2 で把握したコミット内容と、プロジェクトの既存コミットメッセージのスタイルに合わせて提案する：

```
<type>(<scope>): <簡潔な要約>

<詳細な変更内容>
- 変更点1
- 変更点2
- 変更点3

統合されたコミット:
- <hash1> <original message 1>
- <hash2> <original message 2>
- <hash3> <original message 3>
```

**type の例：** feat, fix, refactor, docs, style, test, chore
※ プロジェクトが Conventional Commits を使っていない場合は既存スタイルに合わせる。

---

# 出力形式

以下の順序で出力すること：

1. **対象コミットの一覧**（ハッシュ、メッセージ、変更ファイル数）
2. **実行すべきコマンド**（コピペ可能な形式）
3. **エディタでの編集手順**（具体的なbefore/after例）
4. **推奨コミットメッセージ**（そのまま使用可能な形式）
5. **rebase完了後の確認コマンド**

```bash
# 完了後の確認（ユーザーが実行）
git log --oneline -10
git show --stat HEAD
```

---

# 注意事項

## リモートにpush済みのコミットの場合
> ⚠️ **警告**: 対象コミットがリモートにpush済みの場合、rebase後に `git push --force-with-lease` が必要です。
> 他の開発者と共有しているブランチでは、チームに事前に通知してください。

## コンフリクトが発生した場合

```bash
# コンフリクトしたファイルを確認
git status

# 手動で解決後
git add <resolved-files>
git rebase --continue

# 特定のコミットをスキップする場合
git rebase --skip
```

---

# トラブルシューティング

## rebaseを中断したい場合
```bash
git rebase --abort
```
元の状態に完全に戻ります。

## 間違えてrebaseを完了してしまった場合
```bash
# reflogから元の状態を確認
git reflog

# 元のコミットに戻す（例：5つ前の状態）
git reset --hard HEAD@{5}
```

## エディタが開かない・閉じてしまった場合
```bash
# rebaseを再開
git rebase --edit-todo
```
