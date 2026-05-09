---
name: local-image-ocr
description: >-
  Use when recognizing text and content from images in markdown notes using
  local OCR engines — uses minicpm-v (Ollama) as primary engine for image
  description and text extraction, with Tesseract as fallback
metadata:
  author: redzhx
  title: 本地图片OCR识别工具
  description_zh: >-
    使用minicpm-v（5.5B多模态模型，通过Ollama）识别Markdown笔记中的图片，更新图片链接说明文字，提取文字内容以引用格式置于图片下方，按语义合并断行。Tesseract兜底。完全本地，不修改原文件。
  tags:
    - image
    - ocr
    - local
    - vision
    - notes
    - markdown
    - minicpm-v
    - ollama
  version: 1.0.0
  license: MIT
---

# 本地图片OCR识别工具 (local-image-ocr)

## 概述

使用 **minicpm-v**（5.5B 多模态模型，通过 Ollama）作为主引擎，一次调用同时完成图片描述和文字提取。Tesseract 作为兜底。

**核心优势**：5.5B 参数量布局理解强，无重复幻觉，无需清洗流水线，自动按语义合并断行。

## 核心原则

- ✅ 完全本地处理，无数据外传
- ✅ 更新 `![说明](url)` 中的说明（≤12字，聚焦知识点）
- ✅ 提取图片中的文字，用引用格式 `>` 放在图片下方
- ✅ 保存为新文件 `{原文件名}_ocr.md`，源文件不变
- ✅ 引擎降级：minicpm-v → Tesseract
- ❌ 不修改图片URL
- ❌ 不移动图片位置

## 环境准备

### 1. 安装 Ollama

```bash
brew install ollama
```

### 2. 启动 Ollama 服务

```bash
# 启动服务（如未运行）
ollama serve > /tmp/ollama.log 2>&1 &

# 验证服务就绪
curl -s http://localhost:11434/ | head -c 200
```

### 3. 拉取 minicpm-v 模型（首次使用，约 5.5GB）

```bash
ollama pull minicpm-v:latest
# 已验证可用
```

### 4. 可选兜底引擎

```bash
# Tesseract（最后防线，用于 minicpm-v 失败时）
tesseract --list-langs | grep chi_sim
# 如无中文语言包：
brew install tesseract-lang
```

## 工作流程

### 第一步：提取图片URL

```python
import re
with open('笔记文件.md', 'r', encoding='utf-8') as f:
    content = f.read()

img_pattern = r'!\[([^\]]*)\]\((https?://[^)]+)\)'
images = [(m.group(1), m.group(2), m.start(), m.end())
          for m in re.finditer(img_pattern, content)]
```

### 第二步：并行下载图片到本地

下载到项目 `.tmp/ocr/` 目录。**建议使用多线程并行下载**，尤其是图片数量多时：

```python
import subprocess
from concurrent.futures import ThreadPoolExecutor, as_completed

def download_one(args):
    idx, url = args
    ext = '.png'  # 根据URL扩展名调整
    local_path = f'.tmp/ocr/img_{idx:03d}{ext}'
    subprocess.run(
        ['curl', '-sL', '-o', local_path, '--max-time', '30', url],
        capture_output=True
    )
    return (idx, local_path)

with ThreadPoolExecutor(max_workers=10) as pool:
    futures = [pool.submit(download_one, (i, url)) for i, (_, url, _, _) in enumerate(images, 1)]
    for f in as_completed(futures):
        pass  # 追踪进度
```

### 第三步：minicpm-v 识别（主引擎）

通过 Ollama 的 OpenAI 兼容 API 调用。**关键优化**：
- 提示词要求模型自动检测布局类型（双栏/表格/象限图等）
- 要求按阅读顺序提取文字
- 要求标注所有分栏/分区标题（解决双栏漏标题问题）
- minicpm-v 本身不会产生重复幻觉，无需清洗

```python
import base64, json, urllib.request

PROMPT_TEMPLATE = (
    '请完成三件事：\n'
    '1. 【布局】判断图片的布局类型：单栏正文/双栏并列/三栏并列/'
    '象限图/表格/流程图/环绕图\n'
    '2. 【描述】用不超过12字概括图片主题。直接说知识点，'
    '例如："洛克同一性理论"、"自传体记忆系统"、"埃里克森八阶段"、"叙事性对比"。\n'
    '3. 【文字提取】提取图中所有正文文字：\n'
    '   - 按自然阅读顺序提取\n'
    '   - 保留所有章节标题和分区标题\n'
    '   - 双栏/多栏：先读完一栏全部内容，再读下一栏\n'
    '   - 象限图：按左上→右上→左下→右下\n'
    '   - 表格：逐行读取\n'
    '   - 按语义合并断行，不要保留图片的列宽换行\n'
    '   - 不要提取PPT模板文字（角落Logo、网址、水印、页码）\n\n'
    '严格按此格式输出：\n'
    'LAYOUT: <布局类型>\n'
    'DESC: <12字>\n'
    'TEXT:\n'
    '<完整文字>'
)

def ocr_with_minicpm(image_path):
    with open(image_path, 'rb') as f:
        img_b64 = base64.b64encode(f.read()).decode()

    data = json.dumps({
        'model': 'minicpm-v',
        'messages': [{
            'role': 'user',
            'content': [
                {'type': 'text', 'text': PROMPT_TEMPLATE},
                {'type': 'image_url', 'image_url': {
                    'url': f'data:image/png;base64,{img_b64}'
                }}
            ]
        }],
        'options': {'temperature': 0}
    }).encode()

    req = urllib.request.Request(
        'http://localhost:11434/v1/chat/completions',
        data=data, headers={'Content-Type': 'application/json'}
    )
    resp = urllib.request.urlopen(req, timeout=180)
    result = json.loads(resp.read())
    return result['choices'][0]['message']['content']
```

建议**分批并行处理**（每批2-3张并行，minicpm-v 约 11s/张）。

### 第四步：引擎降级策略

```python
import subprocess

def ocr_image(image_path):
    """minicpm-v → Tesseract"""
    try:
        output = ocr_with_minicpm(image_path)
        layout, desc, text = parse_output(output)
        if text or (desc and desc != "N/A"):
            return desc, text
    except Exception as e:
        print(f"  minicpm-v 失败: {e}")

    # Tesseract（最后防线）
    try:
        r = subprocess.run(
            ['tesseract', image_path, 'stdout', '-l', 'chi_sim+eng'],
            capture_output=True, text=True, timeout=60
        )
        ocr_text = r.stdout.strip()
        return (ocr_text[:12] if ocr_text else "图片"), ocr_text
    except Exception as e:
        print(f"  Tesseract 失败: {e}")

    return "图片", ""
```

输出解析：

```python
import re

def parse_output(output):
    """解析 LAYOUT / DESC / TEXT"""
    lm = re.search(r'LAYOUT:\s*(.+?)(?:\n|$)', output)
    dm = re.search(r'DESC:\s*(.+?)(?:\n|$)', output)
    tm = re.search(r'TEXT:\s*\n(.*)', output, re.DOTALL)

    layout = lm.group(1).strip() if lm else "unknown"
    desc = dm.group(1).strip() if dm else ""
    text = tm.group(1).strip() if tm else ""

    if text in ('[无文字]', '[无]', ''):
        text = ""

    # 如果模型没按格式输出，降级处理
    if not desc and not text:
        lines = output.strip().split('\n')
        first = lines[0].strip()
        if len(first) <= 20:
            return layout, first, '\n'.join(lines[1:]).strip()
        return layout, first[:20], output.strip()

    return layout, desc, text
```

**引擎对比**：

| 引擎 | 能力 | 速度 | 中文精度 | 布局理解 | 模型大小 |
|------|------|------|---------|---------|---------|
| **minicpm-v** | 描述 + 连贯文字，布局感知 | ~11s/张 | 优秀 | 好 | 5.5GB |
| PaddleOCR-VL-1.5 | 描述 + 文字（需清洗） | ~4-5s/张 | 好（但重复幻觉严重） | 差 | 1.7GB |
| Tesseract | 仅文字(粗糙) | ~0.5s/张 | 中 | 无 | ~50MB |

### 第五步（可选）：布局优化预处理——图片物理切分

**适用场景**：minicpm-v 对大多数单栏/双栏/表格布局已能正确处理。但在以下情况仍需物理切分：
1. **三栏或更多栏**：模型可能在栏间跳跃
2. **象限图**：四象限阅读顺序难以保证
3. **复杂表格**：行列对应关系可能错乱

**原理**：检测布局的空白间隙，将多栏/象限图物理切分成多个子图，分别 OCR 后再按阅读顺序拼接结果。

```python
from PIL import Image
import numpy as np

def split_image_layout(image_path):
    """检测布局类型并返回子图路径列表"""
    img = Image.open(image_path)
    arr = np.array(img.convert('L'))  # 灰度
    h, w = arr.shape

    # 垂直投影：检测列间隙
    white_cols = np.sum(arr > 240, axis=0)
    col_gap = white_cols > h * 0.7

    gaps = []
    in_gap = False
    gap_start = 0
    for x in range(w):
        if col_gap[x] and not in_gap:
            gap_start = x
            in_gap = True
        elif not col_gap[x] and in_gap:
            if x - gap_start > w * 0.03:
                gaps.append((gap_start, x))
            in_gap = False

    # 水平投影：检测行间隙
    white_rows = np.sum(arr > 240, axis=1)
    row_gap = white_rows > w * 0.7
    h_gaps = []
    in_gap = False
    gap_start = 0
    for y in range(h):
        if row_gap[y] and not in_gap:
            gap_start = y
            in_gap = True
        elif not row_gap[y] and in_gap:
            if y - gap_start > h * 0.03:
                h_gaps.append((gap_start, y))
            in_gap = False

    col_count = len([g for g in gaps if g[1] - g[0] > w * 0.01])
    row_count = len([g for g in h_gaps if g[1] - g[0] > h * 0.01])
    sub_images = []

    if col_count >= 2 and row_count >= 2:
        # 象限图
        layout = "象限图"
        splits_x = [0] + [g[1] for g in gaps] + [w]
        splits_y = [0] + [g[1] for g in h_gaps] + [h]
        if len(splits_x) >= 3 and len(splits_y) >= 3:
            regions = [(splits_y[0], splits_y[1], splits_x[0], splits_x[1]),
                       (splits_y[0], splits_y[1], splits_x[1], splits_x[2]),
                       (splits_y[1], splits_y[2], splits_x[0], splits_x[1]),
                       (splits_y[1], splits_y[2], splits_x[1], splits_x[2])]
            for yi, (y0, y1, x0, x1) in enumerate(regions):
                sub = img.crop((x0, y0, x1, y1))
                sub_path = image_path.replace('.png', f'_quad_{yi}.png')
                sub.save(sub_path)
                sub_images.append((sub_path, yi))
            return layout, sub_images

    elif col_count >= 2:
        # 多栏
        layout = f"{col_count + 1}栏"
        splits = [0] + [g[1] for g in gaps] + [w]
        for ci in range(len(splits) - 1):
            x0, x1 = splits[ci], splits[ci + 1]
            if x1 - x0 < w * 0.05:
                continue
            sub = img.crop((x0, 0, x1, h))
            sub_path = image_path.replace('.png', f'_col_{ci}.png')
            sub.save(sub_path)
            sub_images.append((sub_path, ci))
        return layout, sub_images

    elif row_count >= 2:
        layout = f"表格（约{row_count + 1}行）"
        splits = [0] + [g[1] for g in h_gaps] + [h]
        for ri in range(len(splits) - 1):
            y0, y1 = splits[ri], splits[ri + 1]
            if y1 - y0 < h * 0.03:
                continue
            sub = img.crop((0, y0, w, y1))
            sub_path = image_path.replace('.png', f'_row_{ri}.png')
            sub.save(sub_path)
            sub_images.append((sub_path, ri))
        return layout, sub_images

    return "单栏", [(image_path, 0)]


def ocr_with_layout_split(image_path):
    """先检测布局切分，再逐块OCR，最后拼接"""
    layout, sub_images = split_image_layout(image_path)

    if layout == "单栏" or len(sub_images) <= 1:
        raw = ocr_with_minicpm(image_path)
        _, desc, text = parse_output(raw)
        return desc, text

    print(f"   检测到{layout}，切分为{len(sub_images)}个子区域")
    all_text = []
    for sub_path, order in sorted(sub_images, key=lambda x: x[1]):
        try:
            raw = ocr_with_minicpm(sub_path)
            _, _, text = parse_output(raw)
            if text:
                all_text.append(text)
        except Exception as e:
            print(f"   子图OCR失败: {e}")

    combined = '\n\n'.join(all_text)
    try:
        raw = ocr_with_minicpm(image_path)
        _, desc, _ = parse_output(raw)
    except:
        desc = None

    return desc, combined
```

### 第六步：注入结果到笔记

```python
def inject_results(content, images, results):
    """从后往前注入，避免位置偏移。每张图对应独立的blockquote"""
    for desc, url, start, end in reversed(images):
        if url not in results:
            continue

        new_desc, clean_text = results[url]
        if not new_desc and not clean_text:
            continue

        # 1. 更新说明文字（仅当原说明为空或无意义时）
        if new_desc and (not desc or desc.strip() in ("", "图片")):
            old_md = f'![{desc}]({url})'
            new_md = f'![{new_desc}]({url})'
            content = content[:start] + new_md + content[end:]
            end = start + len(new_md)

        # 2. 插入引用文字
        if clean_text:
            lines = clean_text.strip().split('\n')
            ocr_block = '\n> ' + '\n> '.join(lines) + '\n'
            content = content[:end] + ocr_block + content[end:]

    return content
```

### 第七步：保存输出

```python
output_path = original_path.replace('.md', '_ocr.md')
with open(output_path, 'w', encoding='utf-8') as f:
    f.write(content)
```

## 完整执行脚本

```python
#!/usr/bin/env python3
"""
local-image-ocr: 本地OCR处理 Markdown 笔记中的图片

引擎：minicpm-v 5.5B（通过 Ollama），Tesseract 兜底

改进要点（v3.0.0）：
1. 从 PaddleOCR-VL-1.5 (llama-server:8081) 换为 minicpm-v (Ollama:11434)
2. 布局感知提示词：自动检测双栏/象限图/表格并指定阅读顺序
3. 要求标注所有分栏标题，避免双栏漏标题
4. minicpm-v 无重复幻觉，无需清洗流水线
5. 保留图片物理切分策略作为可选优化
"""

import re, os, subprocess, sys, base64, json, time, urllib.request
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor, as_completed

NOTEBOOK_FILE = sys.argv[1]
TMP_DIR = Path('.tmp/ocr/')
OUTPUT_FILE = NOTEBOOK_FILE.replace('.md', '_ocr.md')

OLLAMA_ENDPOINT = 'http://localhost:11434/v1/chat/completions'

PROMPT = (
    '请完成三件事：\n'
    '1. 【布局】判断图片的布局类型：单栏正文/双栏并列/三栏并列/'
    '象限图/表格/流程图/环绕图\n'
    '2. 【描述】用不超过12字概括图片主题。直接说知识点。\n'
    '3. 【文字提取】提取图中所有正文文字：\n'
    '   - 按自然阅读顺序提取\n'
    '   - 保留所有章节标题和分区标题（尤其是双栏/多栏的栏目标题）\n'
    '   - 双栏/多栏：先读完一栏全部内容，再读下一栏\n'
    '   - 按语义合并断行，不要保留图片的列宽换行\n'
    '   - 不要提取PPT模板文字（角落Logo、网址、水印、页码）\n\n'
    '严格按此格式输出：\n'
    'LAYOUT: <布局类型>\n'
    'DESC: <12字>\n'
    'TEXT:\n'
    '<完整文字>'
)


def ocr_one(img_path):
    """调用 minicpm-v (Ollama) 进行OCR"""
    with open(img_path, 'rb') as f:
        b64 = base64.b64encode(f.read()).decode()

    data = json.dumps({
        'model': 'minicpm-v',
        'messages': [{
            'role': 'user',
            'content': [
                {'type': 'text', 'text': PROMPT},
                {'type': 'image_url', 'image_url': {'url': f'data:image/png;base64,{b64}'}}
            ]
        }],
        'options': {'temperature': 0}
    }).encode()

    req = urllib.request.Request(
        OLLAMA_ENDPOINT,
        data=data,
        headers={'Content-Type': 'application/json'}
    )
    out = json.loads(urllib.request.urlopen(req, timeout=180).read())
    return out['choices'][0]['message']['content']


def parse_output(output):
    """解析 LAYOUT / DESC / TEXT"""
    lm = re.search(r'LAYOUT:\s*(.+?)(?:\n|$)', output)
    dm = re.search(r'DESC:\s*(.+?)(?:\n|$)', output)
    tm = re.search(r'TEXT:\s*\n(.*)', output, re.DOTALL)

    layout = lm.group(1).strip() if lm else "unknown"
    desc = dm.group(1).strip() if dm else ""
    text = tm.group(1).strip() if tm else ""

    if text in ('[无文字]', '[无]', ''):
        text = ""

    # 降级处理：未按格式输出
    if not desc and not text:
        lines = output.strip().split('\n')
        first = lines[0].strip()
        if len(first) <= 20:
            return layout, first, '\n'.join(lines[1:]).strip()
        return layout, first[:20], output.strip()

    return layout, desc, text


def ocr_with_tesseract(img_path):
    """Tesseract 兜底"""
    r = subprocess.run(
        ['tesseract', img_path, 'stdout', '-l', 'chi_sim+eng'],
        capture_output=True, text=True, timeout=60
    )
    ocr_text = r.stdout.strip()
    return ocr_text


def download_one(args):
    """下载单张图片"""
    idx, url = args
    ext = '.jpg' if ('.jpg' in url or '.jpeg' in url) else '.png'
    if '.webp' in url:
        ext = '.webp'
    local_path = TMP_DIR / f'img_{idx:03d}{ext}'
    r = subprocess.run(
        ['curl', '-sL', '-o', str(local_path), '--max-time', '30', url],
        capture_output=True
    )
    return (idx, str(local_path) if r.returncode == 0 else None)


# ── 启动 ──
with open(NOTEBOOK_FILE, 'r', encoding='utf-8') as f:
    content = f.read()

img_pattern = r'!\[([^\]]*)\]\((https?://[^)]+)\)'
images = [(m.group(1), m.group(2), m.start(), m.end())
          for m in re.finditer(img_pattern, content)]

if not images:
    print("未发现图片，无需处理")
    sys.exit(0)

print(f"发现 {len(images)} 张图片")

# ── 并行下载 ──
TMP_DIR.mkdir(parents=True, exist_ok=True)
paths = {}

with ThreadPoolExecutor(max_workers=10) as pool:
    futures = [pool.submit(download_one, (i, url))
               for i, (_, url, _, _) in enumerate(images, 1)]
    for f in as_completed(futures):
        idx, path = f.result()
        if path:
            paths[idx] = path
            print(f"  [{idx}/{len(images)}] 下载成功")
        else:
            print(f"  [{idx}/{len(images)}] 下载失败，跳过")

if not paths:
    print("所有图片下载失败，退出")
    sys.exit(1)

# ── 分批 OCR（2张并行，minicpm-v 约11s/张）──
results = {}
batch_size = 2

for batch_start in range(0, len(images), batch_size):
    batch = list(enumerate(images[batch_start:batch_start + batch_size], batch_start + 1))

    def process_item(item):
        idx, (desc, url, _, _) = item
        if idx not in paths:
            return (url, None, None)

        try:
            output = ocr_one(paths[idx])
            _, new_desc, text = parse_output(output)
            print(f"  [{idx}/{len(images)}] OK: {new_desc or ''}")
            return (url, new_desc, text)
        except Exception as e:
            print(f"  [{idx}/{len(images)}] minicpm-v 失败: {e}")
            # Tesseract 兜底
            try:
                text = ocr_with_tesseract(paths[idx])
                new_desc = text[:12] if text else ""
                print(f"  [{idx}/{len(images)}] Tesseract 兜底成功")
                return (url, new_desc, text)
            except Exception as e2:
                print(f"  [{idx}/{len(images)}] Tesseract 也失败: {e2}")
                return (url, None, None)

    with ThreadPoolExecutor(max_workers=batch_size) as pool:
        for url, d, t in pool.map(process_item, batch):
            if d or t:
                results[url] = (d, t)

    time.sleep(0.5)  # 批次间间隔

# ── 注入结果 ──
for desc, url, start, end in reversed(images):
    if url not in results:
        continue

    new_desc, clean_text = results[url]
    if not new_desc and not clean_text:
        continue

    # 更新说明文字
    if new_desc and (not desc or desc.strip() in ("", "图片")):
        old_md = f'![{desc}]({url})'
        new_md = f'![{new_desc}]({url})'
        content = content[:start] + new_md + content[end:]
        end = start + len(new_md)

    # 插入引用文字
    if clean_text:
        lines = clean_text.strip().split('\n')
        ocr_block = '\n> ' + '\n> '.join(lines) + '\n'
        content = content[:end] + ocr_block + content[end:]

# ── 保存 ──
with open(OUTPUT_FILE, 'w', encoding='utf-8') as f:
    f.write(content)

print(f"\n完成! 输出: {OUTPUT_FILE}")
print(f"  成功: {len(results)}/{len(images)} 张图片有OCR文字")
```

## 特殊情况处理

| 情况 | 处理方式 |
|------|---------|
| Ollama 服务未启动 | 提示启动 `ollama serve`，降级到 Tesseract |
| minicpm-v 返回空 | 降级到 Tesseract |
| 图片URL无法下载 | 跳过，保留原样 |
| 图片是图表/无文字 | 不添加引用块，仅更新描述 |
| 图片是照片/人物 | 描述画面内容，引用块留空 |
| 同一图片出现多次 | 只处理第一次出现 |
| 原说明已准确 | 保留原说明不改，只添加引用文字 |
| OCR 结果全为空 | 不修改该图片，保留原样 |
| 双栏布局 | 布局感知提示词指定先左后右 + 保留栏目标题 |
| 三栏/多栏布局 | 布局感知提示词指定阅读顺序，必要时物理切分 |
| 象限图 | 布局感知提示词指定左上→右上→左下→右下 |
| 表格 | 布局感知提示词指定逐行读取 |
| 流程图/思维导图 | 布局感知提示词指定主流程方向 |
| 布局感知仍不佳 | 使用图片物理切分策略（split_image_layout） |
| minicpm-v 速度慢 | 每批处理2张，11s/张可接受；图片量大时睡前运行 |

## 与 notes-supplement 的协作

```
local-image-ocr               →       notes-supplement
使用本地引擎提取图片文字             整合STT语音、格式化排版
输出: xxx_ocr.md                    输入: xxx_ocr.md
                                     输出: xxx_出版级.md
```

## 清理

```bash
# 清理临时图片
rm -rf .tmp/ocr/img_*.png .tmp/ocr/img_*.jpg
```

## 注意事项

- minicpm-v 首次使用需 `ollama pull minicpm-v:latest`，约 5.5GB
- Ollama 服务需保持运行，默认端口 11434
- Mac Apple Silicon 自动使用 Metal GPU 加速
- 图片下载到 `.tmp/ocr/`，不会被 git 追踪
- **宁可不识，不要识错**：所有引擎失败时保留原样
- **minicpm-v 约 11s/张**（5.5B 模型），有耐心。如追求速度可尝试 `minicpm-v:q4` 量化版
- **布局感知提示词**已内置双栏/象限图/表格的阅读顺序要求，无需额外操作
- **如双栏仍漏标题**：检查是否因栏目标题字体较小/颜色较浅，手动使用图片切分策略
