# Claude Skills

自定义 Claude Code skills 集合。

## Skills

| Skill | 用途 |
|-------|------|
| [image-ocr](./image-ocr) | 从 Markdown 笔记图片中识别文字 |
| [local-image-ocr](./local-image-ocr) | 本地 OCR 引擎（minicpm-v / Tesseract） |
| [notes-supplement](./notes-supplement) | 将 STT 语音文稿整合到笔记并格式化 |

## 安装

```bash
git clone git@github.com:redzhx/claude-skills.git ~/claude-skills
mkdir -p ~/.claude/skills
ln -s ~/claude-skills/*/ ~/.claude/skills/
```

## License

MIT
