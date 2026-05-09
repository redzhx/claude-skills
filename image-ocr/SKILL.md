---
name: image-ocr
description: "Use when recognizing text and content from images in markdown notes — uses model vision to read images, update image alt-text descriptions, and extract text content as blockquotes below each image"
metadata:
  author: 42ailab
  version: 1.0.0
  title: 图片OCR识别工具
  description_zh: 使用大模型视觉能力识别Markdown笔记中的图片，更新图片链接中的说明文字，提取图片中的文字内容并以引用格式置于图片下方，不修改原文件
  tags:
    - image
    - ocr
    - vision
    - notes
    - markdown
---

# 图片OCR识别工具 (image-ocr)

## 概述

本 skill 使用大模型的视觉能力（Read 工具）识别 Markdown 笔记中的图片内容。它读取笔记中的所有图片，用视觉模型"看懂"图片，然后：
1. 更新图片链接中的说明文字 `![说明](url)`，让说明准确反映图片实际内容
2. 提取图片中的文字，以引用格式 `>` 置于图片下方

**核心原则**：
- ✅ 使用大模型视觉能力识别图片（不依赖外部OCR工具）
- ✅ 更新 `![说明](url)` 中的说明部分，让说明准确描述图片内容
- ✅ 提取图片中的文字，用引用格式放在图片下方
- ✅ 保存为新文件，源文件不变
- ✅ 图表/关系图等无文字图片：简要描述结构，不强行编造内容
- ❌ 不修改图片URL
- ❌ 不移动图片位置

## 何时使用

- 笔记中包含PPT截图、图表、关系图等图片链接，需要提取图片中的文字
- 图片的 `![说明]()` 部分是占位符或不准确的描述，需要用视觉模型重新描述
- 需要在整合STT语音文稿之前，先完成图片文字提取

## 工作流程

### 第一步：提取图片URL

从笔记文件中提取所有图片链接：

```python
import re
with open('笔记文件.md', 'r', encoding='utf-8') as f:
    content = f.read()

# 匹配 ![说明](url) 格式
img_pattern = r'!\[([^\]]*)\]\((https?://[^)]+)\)'
images = re.findall(img_pattern, content)
# images = [('说明文字', 'https://...png'), ...]
```

### 第二步：下载图片到本地

Read 工具需要本地文件路径，所以先将图片下载到临时目录：

```bash
# 为每张图片创建临时文件
mkdir -p /tmp/image-ocr
curl -sL "https://example.com/image.png" -o /tmp/image-ocr/img_001.png
```

**注意事项**：
- 使用 `.png` 或 `.jpg` 后缀，与原始格式保持一致
- 下载失败（超时、404等）时跳过该图片，不中断流程
- 建议按图片在笔记中的出现顺序编号（img_001, img_002...）

### 第三步：使用视觉模型识别图片

对每张下载好的图片，使用 `Read` 工具查看：

```
Read /tmp/image-ocr/img_001.png
```

Read 工具会返回图片的视觉内容描述。**对每张图片需要获取两类信息**：

#### 3.1 图片描述（用于更新 `![说明]()`）
简要描述图片内容（1-2句话），例如：
- PPT标题页 → "《人物志》体别十二材概览表"
- 流程图 → "行为预判的三层模型流程图"
- 纯文字PPT → "三重简化模型的核心概念定义"

#### 3.2 文字提取（用于引用块）
逐行提取图片中的完整文字，保持原有层级结构（标题、列表、表格等）。

### 第四步：更新笔记文件

将识别结果写入笔记，更新两个地方：

#### 4.1 更新图片说明文字
将 `![旧说明](url)` 替换为 `![新说明](url)`，新说明来自视觉模型的图片描述。

#### 4.2 在图片下方添加文字引用
在图片Markdown链接后面插入引用格式的文字内容：

```markdown
![三重简化模型的核心概念](https://example.com/img.png)

> ## 三重简化模型
> - 第一重：情境简化
> - 第二重：认知简化
> - 第三重：行为简化
>
> 人类通过三重简化机制来应对复杂环境
```

**注入格式规范**：
- ✅ 图片与引用文字之间空一行
- ✅ 每行文字前加 `>` 引用标记
- ✅ 保留原文的层级结构（标题、列表、段落）
- ✅ 引用块结束后空一行再接后续内容
- ❌ 不加 "📝 图片文字提取" 等标注（引用格式本身已说明来源）

**从后往前处理**（避免位置偏移）：

```python
matches = list(re.finditer(img_pattern, content))
matches.reverse()  # 从后往前

for match in matches:
    desc, url = match.group(1), match.group(2)
    local_path = url_to_path[url]

    # 读取视觉识别结果
    new_desc = vision_results[url]['description']
    ocr_text = vision_results[url]['ocr_text']

    # 1. 更新说明文字
    old_img_md = f'![{desc}]({url})'
    new_img_md = f'![{new_desc}]({url})'
    content = content[:match.start()] + new_img_md + content[match.end():]

    # 2. 在图片后插入引用文字
    ocr_block = '\n\n> ' + '\n> '.join(ocr_text.split('\n')) + '\n'
    insert_pos = match.end()  # 图片链接结束位置
    content = content[:insert_pos] + ocr_block + content[insert_pos:]
```

### 第五步：保存输出

```python
output_path = original_path.replace('.md', '_ocr.md')
with open(output_path, 'w', encoding='utf-8') as f:
    f.write(content)
```

输出文件命名：`{原文件名}_ocr.md`

## 批量处理策略

对于多图片的笔记文件，建议分批处理：

1. **提取阶段**：一次性提取所有图片URL
2. **下载阶段**：并行下载（每次5-10张），失败的跳过
3. **识别阶段**：逐张用 Read 工具查看
4. **注入阶段**：统一从后往前注入所有结果

**大文件处理**（50+张图片）：
- 先处理前30张，保存中间结果
- 再处理后续图片
- 避免上下文过载

## 特殊情况处理

| 情况 | 处理方式 |
|------|---------|
| 图片URL无法下载 | 跳过，保留原样，不强行脑补 |
| 图片是纯图表/无文字 | `[说明]` 描述图表结构，引用块简要说明图表类型 |
| 图片是照片/人物 | `[说明]` 描述画面内容，引用块留空省去 |
| 同一图片出现多次 | 只在第一次出现时注入OCR内容 |
| 原说明已准确 | 保留原说明不改，只添加引用文字 |

## 与 notes-supplement 的协作

本 skill 是笔记补充工作流的**第一步**：

```
image-ocr（本skill）          →       notes-supplement
提取图片文字、更新说明              整合STT语音、格式化排版
输出: xxx_ocr.md                   输入: xxx_ocr.md
                                   输出: xxx_出版级.md
```

先运行 image-ocr 处理图片，输出的 `_ocr.md` 文件再交给 notes-supplement 做后续整合。

## 使用示例

```
用户输入：
- 笔记文件：/path/to/CH1人性单元.md
- 要求：识别所有PPT图片中的文字

预期操作：
1. 提取 CH1人性单元.md 中所有图片URL（共15张）
2. 下载图片到 /tmp/image-ocr/
3. 逐张用 Read 工具查看：
   - img_001.png → PPT标题页 → 更新说明为"人性单元课程封面"
   - img_002.png → 文字PPT → 更新说明并提取完整文字
   - ...
4. 从后往前注入所有结果到文件
5. 写入 CH1人性单元_ocr.md
```

## 注意事项

- 确保原始笔记文件的 frontmatter（YAML 头部）被完整保留
- Read 工具支持 PNG、JPG、GIF、WebP 等常见图片格式
- 图片下载到 `/tmp/image-ocr/`，处理完成后可手动清理
- **宁可不识，不要识错**：不确定的文字宁可不提取，也不要编造
- 原笔记中已有的手写内容不修改、不覆盖
