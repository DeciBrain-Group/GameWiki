# 交接说明：梳理六个游戏的环境接口（Environment Interfaces）

分支：`feat/env-interface-spec`　　样例：星际争霸 II（已完成）　　日期：2026-11-23

## 1. 要做什么

为 DSGBench 的六个游戏逐一梳理“封装链路”，写进 GameWiki 的 **API Reference → Environment Interfaces**。每个游戏说清七件事：

1. 原始观测空间（游戏引擎 / API 给出的完整状态）
2. 原始动作空间（原生环境接受的底层指令）
3. 封装后的观测空间（`env.step()` 返回给智能体的字段）
4. 大模型 Prompt 结构（系统提示词 + 用户输入）
5. 封装后的动作空间（离散动作表或结构化动作格式）
6. 大模型输出结构及解析方式（如何从输出中提取动作、失败时怎么办）
7. 评测数据（五个能力维度各自的数据是否已记录、需要标定还是缺失）

这些信息直接决定后续细粒度指标能不能算，所以第 7 项尤其要准。

## 2. 照着样例做

- 概览页：`docs/code_framework/gym_interface/gym_interface.md`（统一封装链路 + 六个游戏的对照表，目前只有星际争霸 II 一行填好）
- 样例详情页：`docs/code_framework/gym_interface/starcraft2.md`
- 空白模板：`templates/interface_template.md`（复制到 `docs/code_framework/gym_interface/<game>.md`）
- 中文可视化参考：DocsBench 原型的“开发介绍”页（`DocsBench/app/dev.html`，数据在 `app/dev-data.js`），内容与样例一致

## 3. 信息从哪里来

DSGBench 开源仓库（本地归档）：`F:\1.Git-Repository\9_归档·Archive\DSGBench_2024.07-2025.03\DSGBench_OpenSource\DSGBench`

| 游戏 | 说明文档 | 代码（重点看） |
|------|---------|---------------|
| StarCraft II | `games/starcraft2/readme.md` | 环境 `agent_vs_computer_starcraft_env.py`、`bot/Protoss_bot.py`；智能体 `agent_manager/agents/starcraft2_agent/`；提示词 `agent_manager/prompts/starcraft2_prompt.py`；解析 `utils/action_extractor.py` |
| Civilization | `games/civ/readme.md` | `games/civ/`、`agent_manager/agents/` 与 `agent_manager/prompts/` 下对应文件 |
| Street Fighter III | `games/streetfight3/readme.md` | `games/streetfight3/` 及对应 agent、prompt |
| Diplomacy | `games/welfare_diplomacy/readme.md` | `games/welfare_diplomacy/` 及对应 agent、prompt |
| Werewolf | `games/werewolf/readme.md` | `games/werewolf/` 及对应 agent、prompt |
| Stratego | `games/stratego/readme.md` | `games/stratego/` 及对应 agent、prompt |

找字段的方法：在环境类里看 `observation_space`、`action_space`、`step()` 返回的 `state`；在 agent 里看调用大模型的频率（如 `action_interval`）、拼接 prompt 的方式、解析输出的函数。

## 4. 写作规则

- 每个事实都要能在 README 或代码里找到，并在页面末尾“Source code”表里写出文件路径；查不到写 `TODO`，不要猜。
- Prompt 与输出示例从 README 或实际运行日志里原样粘贴，可以节选，节选处用 `...` 标出。
- “Evaluation data”的状态只用四种：`✅ Available`、`⚠️ Partial`（或 `⚠️ Needs probes`）、`❓ To confirm`、`➖ Not applicable`，并写清数据来自哪个字段、缺什么。
- 网站是公开的，只写 DSGBench 已公开的五个能力维度，不写内部未发表的指标名称。
- 语言与站点一致，用英文。

## 5. 每完成一个游戏

1. 新建 `docs/code_framework/gym_interface/<game>.md`（按模板填写）。
2. 在 `mkdocs.yml` 的 `Environment Interfaces` 下加一行导航。
3. 把概览页对照表里该游戏那一行的 `TODO` 换成摘要，并把游戏名改成指向详情页的链接。
4. 在该游戏的专题页（`docs/environments/<game>/<game>.md`）合适位置加一个 `!!! tip "Interface specification"` 链接（参考星际争霸 II 专题页 Part IV）。
5. 本地预览：`pip install mkdocs-material` 后运行 `mkdocs serve`，浏览器打开 http://127.0.0.1:8000 检查表格和折叠块显示正常；`mkdocs build --strict` 无报错。
6. 每个游戏单独提交一次，提交信息如 `docs: interface spec for Civilization`。

## 6. 验收标准

- 六个游戏都有详情页，概览表没有 `TODO`（确实查不到的写 `TODO` 并在 PR 说明中列出）。
- 每页“Source code”表的路径都真实存在。
- 每页“Evaluation data”五个维度都有状态和说明。
- `mkdocs build --strict` 通过。
- 完成后发起合并请求到 `master`，由项目负责人审阅后合并；不要直接推送到 `master` 或 `gh-pages`。
