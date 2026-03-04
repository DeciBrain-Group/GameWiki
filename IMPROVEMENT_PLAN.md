# DSGBench GameWiki 改进方案

> 制定日期：2026-03-04

---

## 一、现状诊断

| 问题 | 详情 |
|------|------|
| 首页有"Under Construction"警告 | 第一印象极差，且内容已过时（只写DSGBench v1） |
| 绝大部分页面实际为空 | `evaluation_and_capabilities.md`、`code_framework.md`、所有`experimental`页面、`environments.md` — 全是1行空文件 |
| `mkdocs.yml` 引用了大量不存在的文件 | `performance.md`、`multi_agent_interaction.md`、`benchmarks.md`等6个文件根本不存在 |
| 没有DSGBench++内容 | Trajectory系统、RL训练、闭环架构 — 完全没有覆盖 |
| 视觉设计零定制 | 纯默认Material主题，没有品牌色、没有卡片布局 |
| 没有Leaderboard / 实验结果 | 研究benchmark网站最核心的内容缺失 |
| 没有Getting Started教程路径 | 用户不知道从哪里开始 |

---

## 二、标杆参考案例

| 网站 | 与本项目的关联 | 可借鉴的关键点 |
|------|--------------|--------------|
| **Gymnasium** (farama.org) | 同样是多游戏环境库 | 游戏卡片布局、环境分类徽章、Material主题深度定制 |
| **SWE-bench** (swebench.com) | 同类学术benchmark | Leaderboard表格、论文数据的可视化呈现、简洁的hero区 |
| **AgentBench** (llmbench.ai) | 最相近的定位（LLM Agent Benchmark） | 任务分类展示、评分维度说明、Paper链接突出 |
| **HELM** (Stanford) | 多维LLM评估 | 评估维度可视化、清晰的方法论说明 |
| **LangGraph Docs** | 复杂系统文档 | 系统架构图的呈现、Getting Started的组织方式 |

---

## 三、信息架构重组

```
现状（混乱、内容空）          建议（清晰、完整）
─────────────────────         ──────────────────────────────────
Home                          Home（品牌化Landing Page）
Environments                  Getting Started（快速上手）
  ├─ environments.md（空）      ├─ 安装配置
  ├─ Starcraft II              └─ 运行第一个评测
  ├─ Civilization            Games（6大游戏Wiki）
  ├─ Street Fighter III        ├─ StarCraft II
  ├─ Diplomacy                 ├─ Civilization
  ├─ Werewolf                  ├─ Street Fighter III
  └─ Stratego                  ├─ Diplomacy
Evaluation and Capabilities    ├─ Werewolf
  ├─ ...（空）                  └─ Stratego
Code Framework（空）         DSGBench（评测基准）
Experimental（空）             ├─ 评测指标体系
                               ├─ 实验结果 / Leaderboard
                               └─ 能力维度说明
                             DSGBench++（闭环系统）
                               ├─ 系统架构
                               ├─ Trajectory系统
                               └─ RL训练流程
                             API Reference（代码框架）
                             About / Citation
```

---

## 四、实施计划

### 第一阶段（核心重建）

| 任务 | 文件 | 状态 |
|------|------|------|
| mkdocs.yml 完全重写 | `mkdocs.yml` | ✅ |
| 主题CSS | `docs/stylesheets/extra.css` | ✅ |
| 首页改造 | `docs/index.md` | ✅ |
| Getting Started | `docs/getting_started/quickstart.md` | ✅ |
| Civilization游戏页 | `docs/environments/civ/civ.md` | ✅ |
| Street Fighter游戏页 | `docs/environments/street_fighter/street_fighter.md` | ✅ |
| Diplomacy游戏页 | `docs/environments/diplomacy/diplomacy.md` | ✅ |
| Werewolf游戏页 | `docs/environments/werewolf/werewolf.md` | ✅ |
| Stratego游戏页 | `docs/environments/stratego/stratego.md` | ✅ |
| 导航存根页面 | 多个 | ✅ |

### 第二阶段（深度内容）

| 任务 | 文件 | 状态 |
|------|------|------|
| DSGBench++专区 | `docs/dsgbenchpp/*.md` | 🔜 |
| Leaderboard（需实验数据） | `docs/dsgbench/leaderboard.md` | 🔜 |
| 评测指标详细说明 | `docs/evaluation_and_capabilities/` | 🔜 |
| API Reference | `docs/code_framework/` | 🔜 |

---

## 五、主题配置方案

### 品牌色

| 字母 | 颜色 | RGB |
|------|------|-----|
| **D** | 绿 | `rgb(0, 176, 80)` |
| **S** | 黄 | `rgb(250, 196, 34)` |
| **G** | 蓝 | `rgb(0, 176, 240)` |
| **++** | 紫 | `rgb(200, 80, 200)` |

### Material 主题特性

- `navigation.tabs` — 顶部Tab式主导航
- `navigation.sections` — 侧栏分组
- `navigation.instant` — SPA即时导航
- `navigation.top` — 回到顶部按钮
- `search.highlight` — 搜索结果高亮
- `content.code.copy` — 代码复制按钮
- 深色/浅色模式切换

---

## 六、游戏页面标准模板

每个游戏页面包含：
1. 游戏概述（通俗说明）
2. 游戏对 LLM 的核心挑战
3. 环境接口（Observation Space / Action Space / Reward）
4. 评测场景设置
5. 评测指标
6. 相关资源链接
