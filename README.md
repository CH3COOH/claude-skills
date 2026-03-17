# claude-skills

Claude Code で使う個人用 Agent Skills のコレクションです。

## Skills 一覧

| Skill | 説明 |
|---|---|
| [xcstrings-localize](./xcstrings-localize/) | iOS アプリの多言語対応（xcstrings / Localizable.strings） |

## セットアップ
```bash
git clone https://github.com/CH3COOH/claude-skills.git ~/repos/claude-skills

# シンボリックリンクで ~/.claude/skills/ に接続
mkdir -p ~/.claude/skills
ln -s ~/repos/claude-skills/xcstrings-localize ~/.claude/skills/xcstrings-localize
```

## 更新
```bash
cd ~/repos/claude-skills && git pull
```

## ライセンス

MIT