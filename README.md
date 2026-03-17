# claude-skills

Claude Code で使う個人用 Agent Skills のコレクションです。

## Skills 一覧

| Skill | 説明 |
|---|---|
| [xcstrings-localize](./xcstrings-localize/) | iOS アプリの多言語対応（xcstrings / Localizable.strings） |
| [git-rebase](./git-rebase/) | 指定範囲のコミットを1つに統合（squash）する interactive rebase 手順の生成 |

## セットアップ
```bash
git clone https://github.com/CH3COOH/claude-skills.git ~/repos/claude-skills

# シンボリックリンクで ~/.claude/skills/ に接続
mkdir -p ~/.claude/skills
ln -s ~/repos/claude-skills/xcstrings-localize ~/.claude/skills/xcstrings-localize
ln -s ~/repos/claude-skills/git-rebase ~/.claude/skills/git-rebase
```

## 更新
```bash
cd ~/repos/claude-skills && git pull
```

## ライセンス

MIT