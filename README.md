# 🎬 Short Drama Production Index

**交互式短剧制作工作台 — 管理 AI 生图/视频流程、追踪进度、一键复制 Prompt**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Version](https://img.shields.io/badge/Version-2.2-green.svg)](https://github.com/husw725/short-drama-production-index)

## ✨ 特性

- 📋 **8 大 Tab 管理面板** — 分镜、剧本、Prompt、场景、道具、视觉资产、好莱坞剧本、角色
- 🎯 **一键复制 Prompt** — Image/Video Prompts 直接复制，支持场景/道具引用展开
- 📊 **进度追踪** — localStorage 持久化，实时记录每集/每帧的 AI 生成状态
- 🖼️ **视觉资产管理** — 场景 Reference、道具 Reference、角色 Reference 统一管理
- 🎬 **好莱坞格式剧本** — 自动生成标准 Final Draft 格式 Screenplay
- 🔗 **场景/道具引用系统** — `[ref: S-01]`、`[ref: P-01]` 自动展开完整描述
- 💾 **零依赖单文件** — 纯前端 SPA，单个 `index.html` 即可运行

## 📁 目录结构

```
├── SKILL.md          # Hermes Agent 技能定义（主文件）
├── README.md         # 本文档
```

## 🚀 快速开始

### 1. 准备项目文件

确保你的短剧项目包含以下文件：

```
your-project/
├── script/
│   ├── EP-01.md      # 剧本
│   ├── EP-02.md
│   └── ...
├── storyboard/
│   ├── EP-01.md      # 分镜
│   └── ...
├── prompts/
│   ├── EP-01.md      # AI 提示词
│   └── ...
├── characters/
│   └── characters.md  # 角色设定
└── visual_assets/
    └── manifest.md    # 视觉资产清单
```

### 2. 安装技能

将 `SKILL.md` 放入 Hermes Agent 技能目录：

```bash
cp SKILL.md ~/.hermes/skills/creative/short-drama-production-index/SKILL.md
```

### 3. 生成工作台

```bash
cd your-project/

# Step 1: 解析 Markdown → JSON
python3 generate_index.py

# Step 2: 生成 HTML 工作台
python3 build_html.py

# Step 3: 打开浏览器
xdg-open index.html
```

## 📖 8 大 Tab 说明

| Tab | 功能 | 适用阶段 |
|-----|------|---------|
| 🎬 分镜 | 分镜表 + 进度追踪 | 分镜审核 |
| 📜 剧本 | 场景分解 + 对白 + 悬念 | 剧本审阅 |
| 🎨 Prompt | Image/Video Prompts + 一键复制 | AI 生成 |
| 🏛️ 场景 | 场景 Reference + 状态管理 | 资产生成 |
| 🎭 道具 | 道具 Reference + 状态管理 | 资产生成 |
| 🖼️ 视觉资产 | manifest.md 表格展示 | 全局管理 |
| 📝 Screenplay | 好莱坞格式剧本 + 集数切换 | 交付输出 |
| 👤 角色 | 角色设定一览 | 参考查询 |

## 🔧 技术架构

- **解析器**: `generate_index.py` — Markdown → JSON
- **渲染器**: `build_html.py` — JSON → 自包含 SPA HTML
- **前端**: 纯 Vanilla JS，零依赖
- **持久化**: localStorage 保存进度状态

## 🎯 场景/道具引用规范

```
# Prompts 中使用引用标记
scene_ref: "S-01"
prop_refs: ["P-01", "P-02"]

# Prompt 文本中
[ref: S-01] + [ref: P-01] → 自动展开完整描述
```

## 📋 依赖

- Python 3.8+
- 无需额外 Python 包（纯标准库）

## 🤝 相关项目

- [hermes-short-drama-team](https://github.com/husw725/hermes-short-drama-team) — 短剧编剧系统
- [seedance2-short-drama-workflow](https://github.com/husw725/seedance2-short-drama-workflow) — Seedance 2.0 工作流

## 📄 License

MIT
