---
name: ios-bump-version
description: >
  iOSアプリのバージョンをアップデートします。
  新しいバージョン番号を指定すると、ブランチ作成・project.pbxproj 更新・コミットまで自動実行します。
  「バージョンを X.X.X にアップデート」「バージョンを上げて」「Bump up」などで起動します。
allowed-tools: Bash, Read, Edit
argument-hint: "<new-version> （例: 2.9.10）"
metadata:
  author: Kenji Wada
  version: 1.0.0
  category: release
---

# Bump Version スキル

iOSアプリ の `MARKETING_VERSION` を更新し、リリースブランチへのコミットまで自動実行します。

## 事前バリデーション

以下を確認し、問題があれば処理を中断してユーザーに案内すること。

| チェック項目 | 確認コマンド | 問題がある場合 |
|-------------|-------------|--------------|
| 引数の有無 | - | 新バージョン番号を確認する |
| バージョン形式 | - | `X.X.X` 形式（数字とドット）でなければ拒否 |
| 作業ディレクトリ | `git status --porcelain` | 未コミット変更があれば先に commit/stash を案内 |
| 現在のブランチ | `git branch --show-current` | `release-vX.X.X`・`release-vX.X.X-feature/*`・`develop`・`master` のいずれかであることを確認。それ以外は処理を中断する |
| 現在のバージョン | `grep -n "MARKETING_VERSION" *.xcodeproj/project.pbxproj` | 目標バージョンと同一なら「既に最新です」と警告して終了 |

## 手順

ここでは例として MyApp というアプリのバージョンを修正する。

### Step 1: 現在の状態を確認

```bash
git branch --show-current
git status --porcelain
```

現在のバージョンを確認する：

```bash
grep -n "MARKETING_VERSION" MyApp.xcodeproj/project.pbxproj
```

### Step 2: フィーチャーブランチを作成

現在のブランチ名に応じてブランチ名を決定する。

**ケース 1: `release-vX.X.X` 形式のリリースブランチ上（通常）:**

```bash
git checkout -b release-v<base-version>-feature/bump-up-to-v<new-version>
```

例: ベースが `release-v2.9.10`、新バージョンが `2.9.10` の場合
```bash
git checkout -b release-v2.9.10-feature/bump-up-to-v2.9.10
```

**ケース 2: `release-vX.X.X-feature/*` または `release-vX.X.X-fix/*` 形式（既に feature/fix ブランチ上）:**

新規ブランチは作成しない。現在のブランチのまま作業を続ける。

**ケース 3: `develop`・`master` などその他のブランチ:**

```bash
git checkout -b feature/bump-up-to-v<new-version>
```

### Step 3: project.pbxproj を更新

`MyApp.xcodeproj/project.pbxproj` 内の `MARKETING_VERSION` をすべて新バージョンに更新する。

- 対象行: `MARKETING_VERSION = <old-version>;`（Debug・Release 各 1 行、計 2 行）
- 変更後: `MARKETING_VERSION = <new-version>;`

**注意**: テストターゲットは変更しない。

更新後に確認する：

```bash
grep -n "MARKETING_VERSION" MyApp.xcodeproj/project.pbxproj
```

### Step 4: コミット

コミットメッセージは必ず以下の形式を使用する：

```
Bump up v<new-version>
```

```bash
git add MyApp.xcodeproj/project.pbxproj
git commit -m "Bump up v<new-version>"
```

## 出力形式

完了後、以下を報告する：

1. **作成したブランチ名**
2. **変更前 → 変更後のバージョン**
3. **コミットハッシュ**

## 注意事項

- `MARKETING_VERSION = 1.0;` の行は変更しない（テストターゲット用）
- 変更対象は必ず Debug・Release の 2 行
- コミットメッセージは `Bump up vX.X.X` 形式で統一
