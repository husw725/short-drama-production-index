---
name: short-drama-production-index
description: 为短剧项目（剧本/分镜/Prompts Markdown文件）生成结构化 JSON 和交互式 HTML 工作台，用于管理 AI 生图/视频流程、追踪进度、一键复制 Prompt
version: 2.2
author: Hermes Agent
metadata:
  hermes:
    tags: [Short-Drama, Production-Workflow, HTML-Workbench, Project-Index, Scene-Reference, Prop-Reference]
    related_skills: [hermes-short-drama-team, seedance2-short-drama-workflow]
---

# Short Drama Production Index Page — 交互式制作工作台 v2.2

为短剧项目（drama-team 输出的 Markdown 文件）生成结构化 JSON 和交互式 HTML 工作台。

**v2.0 新增**：视觉资产清单、好莱坞格式 Screenplay、Props 注入、场景 Prompt 注入
**v2.2 新增**：场景管理 + 道具管理 Tab（8 Tab），场景/道具 Reference 数据解析与 UI 展示，`project_data.json` 支持 `scenes[]` 和 `props[]` 顶级字段

## 触发条件

短剧项目已完成以下文件：
- `script/EP-XX.md` — 剧本
- `storyboard/EP-XX.md` — 分镜
- `prompts/EP-XX.md` — AI 提示词
- `characters/characters.md` — 角色设定
- `visual_assets/manifest.md` — 视觉资产清单

## 工作流

### Step 1: 创建 `generate_index.py`

放置于项目根目录，解析所有 Markdown 文件为 `project_data.json`。

```python
#!/usr/bin/env python3
import json, os, re

BASE = os.path.dirname(os.path.abspath(__file__))

def read(path):
    with open(os.path.join(BASE, path), 'r', encoding='utf-8') as f:
        return f.read()

def parse_manifest(text):
    """Parse visual_assets/manifest.md into structured data."""
    sections = {}
    current_section = None
    current_table = []
    for line in text.split('\n'):
        if line.startswith('## ') and not line.startswith('### '):
            if current_section and current_table:
                sections[current_section] = current_table
            current_section = line[3:].strip()
            current_table = []
            continue
        if line.startswith('### '): continue
        if line.strip().startswith('|') and current_section:
            cells = [c.strip() for c in line.strip().split('|')[1:-1]]
            # ⚠️ 必须同时过滤多种分隔线格式：---、------、-------- 等
            if len(cells) >= 2 and not all(c == '---' for c in cells) and not all(c == '------' for c in cells):
                current_table.append(cells)
    if current_section and current_table:
        sections[current_section] = current_table
    return sections

def parse_script(md):
    """Parse script EP-XX.md → {meta, scenes, dialogue, cliffhanger}"""
    scenes = re.findall(r'## Scene Breakdown\n(.+?)(?=##|$)', md, re.DOTALL)
    dialogue = re.findall(r'## Key Dialogue\n(.+?)(?=##|$)', md, re.DOTALL)
    cliffhanger = re.search(r'## Cliffhanger\n\n?(.+?)(?=$)', md, re.DOTALL)
    return {'scenes': scenes, 'dialogue': dialogue, 'cliffhanger': cliffhanger.group(1).strip() if cliffhanger else ''}

def parse_storyboard(md):
    shots = re.findall(r'## Key Frames\n(.+?)(?=##|$)', md, re.DOTALL)
    return {'shots': shots}

def parse_prompts(md):
    image = re.findall(r'### Frame (\d+): (.+?)\n\*\*Prompt:\*\*(.*?)(?=\n\n### |\Z)', md, re.DOTALL)
    video = re.findall(r'### Shot (\d+): (.+?)\n\*\*Prompt:\*\*(.*?)(?=\n\n### |\Z)', md, re.DOTALL)
    return {'imagePrompts': image, 'videoPrompts': video}

def main():
    data = {'episodes': [], 'characters': [], 'manifest': {}}
    
    # Parse manifest
    mp = os.path.join(BASE, 'visual_assets', 'manifest.md')
    if os.path.exists(mp):
        data['manifest'] = parse_manifest(read('visual_assets/manifest.md'))
    
    # Load screenplays (if convert_screenplay.py has been run)
    sp = os.path.join(BASE, 'project_screenplays.json')
    if os.path.exists(sp):
        with open(sp, 'r', encoding='utf-8') as f:
            data['screenplays'] = json.load(f)
    
    # Parse episodes
    episodes = []
    script_dir = os.path.join(BASE, 'script')
    for fname in sorted(os.listdir(script_dir)):
        if not fname.endswith('.md'): continue
        ep_id = fname.replace('.md', '')
        episodes.append({
            'id': ep_id,
            'script': parse_script(read(f'script/{fname}')),
            'storyboard': parse_storyboard(read(f'storyboard/{ep_id}.md')),
            'prompts': parse_prompts(read(f'prompts/{ep_id}.md')),
        })
    
    # Characters
    cf = os.path.join(BASE, 'characters', 'characters.md')
    if os.path.exists(cf):
        data['characters'] = [c.strip() for c in read('characters/characters.md').split('\n## ')[1:]]
    
    data['episodes'] = sorted(episodes, key=lambda e: e['id'])
    
    out = os.path.join(BASE, 'project_data.json')
    with open(out, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)
    
    total_img = sum(len(ep['prompts']['imagePrompts']) for ep in data['episodes'])
    total_vid = sum(len(ep['prompts']['videoPrompts']) for ep in data['episodes'])
    print(f"Parsed {len(data['episodes'])} episodes, {len(data['characters'])} characters")
    print(f"Total: {total_img} image prompts, {total_vid} video prompts")

if __name__ == '__main__':
    main()
```

### Step 2: 运行解析

```bash
cd /path/to/project
python3 generate_index.py
```

### Step 3: 创建 `build_html.py`

生成自包含 SPA（JSON 嵌入 HTML）。

**⚠️ 关键：不要用 f-string 模板！** JS 代码中的 `{}` 与 Python f-string 冲突，导致大量语法错误（尤其 JS 对象字面量如 `{type:'img',idx:i}`）。

**正确做法：纯字符串模板 + 占位符替换**：

```python
import json, os

# 读取 project_data.json
with open('project_data.json', 'r', encoding='utf-8') as f:
    data = json.load(f)
json_str = json.dumps(data, ensure_ascii=False)

# 用 r"""...""" 原始字符串（不解析 \n 等转义）+ 占位符
template = r"""<!DOCTYPE html>
...
<script>
const PROJECT = __JSON_PLACEHOLDER__;
...
</script>
</html>"""

# 替换占位符
html = template.replace('__JSON_PLACEHOLDER__', json_str)
with open('index.html', 'w', encoding='utf-8') as f:
    f.write(html)
```

**HTML 模板包含 8 个 Tab**：
1. **分镜 (storyboard)** — 分镜表 + 进度追踪
2. **剧本 (script)** — 场景分解 + 对白 + 悬念
3. **Prompt (prompts)** — Image/Video Prompts + 场景/道具引用标签
4. **场景管理 (scenes)** — 场景 reference prompt + 状态管理（v2.2 新增）
5. **道具管理 (props)** — 关键道具 reference prompt + 状态管理（v2.2 新增）
6. **视觉资产 (visual_assets)** — manifest.md 表格展示
7. **Screenplay (screenplay)** — 好莱坞格式剧本 + 集数切换
8. **角色 (characters)** — 角色设定一览

### Step 4: 运行生成

```bash
python3 build_html.py
# → 生成 index.html（自包含 SPA）
```

## 关键设计

- **双文件架构**：`generate_index.py` (解析) + `build_html.py` (渲染)
- **JSON 嵌入 HTML**：`build_html.py` 将 `project_data.json` 通过 `json.dumps` 注入 `PROJECT` 变量
- **纯字符串模板**：用 `r"""..."""` + `str.replace()` 注入数据，避免 f-string `{}` 与 JS 冲突
- **纯前端 SPA**：零依赖，单 HTML 文件，Tab 切换纯 JS 实现
- **localStorage**：状态持久化（进度、编辑、图片）

## build_html.py 新增 Tab 三步走

1. **HTML tab 按钮**：`<div class="tab" data-tab="xxx">标签</div>`
2. **HTML 内容区**：`<div class="tab-content" id="tab-xxx"><div id="xxxContainer"></div></div>`
3. **JS 渲染函数**：`function buildXxx() {{ ... }}` + 在 `init()` 中调用

## 已知 Pitfalls

### parse_table 最小列数
`parse_table` 的 `len(cells) < 3` 过滤会漏掉 2 列表格（如 Key Dialogue 的 `| EN | CN |`）。必须用 `len(cells) < 2` 才能兼容所有表格宽度。

### Cliffhanger 标题后缀
Cliffhanger 标题可能带后缀（如 `## Cliffhanger / 终局`），正则 `## Cliffhanger\\n` 不匹配。必须用 `## Cliffhanger[^\\n]*\\n` 匹配任意后缀。

### Manifest 分隔线过滤
Markdown 表格分隔线有多种形式（`---`、`------`、`--------` 等），parse_manifest 必须同时过滤：
```python
if not all(c == '---' for c in cells) and not all(c == '------' for c in cells):
```
仅检查 `---` 会漏掉长度不同的分隔线，导致它们被当作数据行。

### Manifest section 正则过宽 — 误吸脚本内容
`## [^#]` 的正则匹配会捕获 `## Scene Breakdown`、`## Key Dialogue` 等脚本内标题，导致剧本内容混入场景/道具数据。

**修复方法**：使用精确的模式匹配：
```python
def parse_manifest_sections(text):
    """Parse manifest.md sections into structured data."""
    # 使用精确匹配，避免误吸脚本标题
    section_pattern = r'## (Scene Visuals|Scene [A-Z]|Scene [0-9]|Prop [A-Z]|Prop [0-9])'
    
    scenes = {}
    props = {}
    
    # 解析场景：从表格行提取 Scene-XX ID
    scene_block = re.search(r'## Scene Visuals\n\n(.+?)(?=\n## |\Z)', text, re.DOTALL)
    if scene_block:
        for line in scene_block.group(1).split('\n'):
            cells = [c.strip() for c in line.strip().split('|')[1:-1]]
            if len(cells) >= 3:
                scene_id = cells[0]  # e.g. Scene-01
                name = cells[1]
                prompt = cells[2]
                scenes[scene_id] = {'name': name, 'prompt': prompt, 'images': []}
    
    # 解析道具：从表格行提取 P-XX ID
    prop_block = re.search(r'## Prop Reference Prompts\n\n(.+?)(?=\n## |\Z)', text, re.DOTALL)
    if prop_block:
        for line in prop_block.group(1).split('\n'):
            cells = [c.strip() for c in line.strip().split('|')[1:-1]]
            if len(cells) >= 3:
                prop_id = cells[0]  # e.g. P-01
                name = cells[1]
                prompt = cells[2]
                props[prop_id] = {'name': name, 'prompt': prompt, 'episodes': cells[3] if len(cells) > 3 else ''}
    
    return scenes, props
```

### 正则转义陷阱：`\\n` 与 `\n`
在 `generate_index.py` 的正则中，`\\n` (字面量反斜杠+n) 与 `\n` (换行符) 行为不同。

- **在 `.py` 源文件中**：`r'\n'` 表示真正的换行符（推荐），`\\n` 在某些上下文中可能表示字面量 `\n` 文本
- **错误表现**：正则 `## Scene Breakdown\\n` 有时匹配不到内容，因为 `\\n` 不等同于 `\n`
- **正确做法**：始终使用原始字符串 `r'...'` 配合 `\n`，如 `r'## Scene Breakdown\n(.+?)(?=##|$)'`
- **验证方法**：打印 `repr()` 确认正则中的换行符是正确的 `\n` 而非 `\\n`

### 场景/道具引用标准 (v2.2 Drama-Team 规范)
Prompts 中的场景/道具不应全文本内嵌，应使用引用标记：
- **场景引用**：`scene_ref: "S-01"` + 在 prompt 文本中使用 `[ref: S-01]`
- **道具引用**：`prop_refs: ["P-01", "P-02"]` + 在 prompt 文本中使用 `[ref: P-01]`
- **渲染时**：从 `scenes[S-01]['prompt']` 展开得到完整场景描述
- **改造时**：将 `scene: [Gothic Victorian bedroom...]` 替换为 `[ref: S-01]`，并添加 `scene_ref` 和 `prop_refs` 元数据字段

### renderTab 路由必须覆盖所有 Tab
`renderTab()` 函数中只处理了 storyboard/script/prompts 三个动态 Tab（随集数切换重新渲染），visual_assets/screenplay/characters/scenes/props 在 `init()` 中一次性构建，**不随集数切换重新渲染**。scenes/props 是全局资产（跨集共享），所以一次性构建是正确的设计。如需动态切换集数时刷新这些 Tab，需在 `renderTab` 中添加对应路由或在 `selectEp` 中手动调用。

### 好莱坞剧本 Word 转换
使用 `python-docx` 生成 .docx 时，注意：
- 角色名居中加粗
- 对白缩进 1.5 英寸左右
- 场景标题（INT./EXT.）大写加粗
- 每集之间分页

## Props 批量注入到帧级 Prompt

```python
# 从 manifest.md 解析道具清单（含出现集数）
# 构建 ep_num → props 映射
# 每帧 Prompt 用关键词匹配注入对应道具
# 注入位置：scene: [...] 描述之后
```

## 正则适配指南

| 解析目标 | 正则锚点 |
|----------|---------|
| 场景分解 | `## Scene Breakdown\n(.+?)(?=##|$)` |
| Key Dialogue | `## Key Dialogue\n(.+?)(?=##|$)` |
| 分镜表 | `## Key Frames\n(.+?)(?=##|$)` |
| Image Prompts | `### Frame (\d+): (.+?)\n\*\*Prompt:\*\*(.*?)` |
| Video Prompts | `### Shot (\d+): (.+?)\n\*\*Prompt:\*\*(.*?)` |

## 打包交付

```bash
tar --exclude='__pycache__' -czf /path/to/desktop/project-name.tar.gz -C /path/to project-name/
```

## 相关技能

- `hermes-short-drama-team` — 短剧编剧系统
- `seedance2-short-drama-workflow` — Seedance 2.0 工作流
- `short-drama-image-generation` — Dreamina 生图
