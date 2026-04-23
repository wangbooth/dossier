# dossier

[English](README.md) · [中文](README.zh-CN.md)

> 一个通用 [Skills](https://agentskills.io/) 格式的 skill —— 兼容 [Claude Code](https://claude.com/claude-code)、Codex、ChatGPT 等支持 Skills 的 AI agent。让你的 AI agent 把"需要引用的研究工作"委托给 [Google NotebookLM](https://notebooklm.google.com/) 处理，语料不进 agent 的对话窗口。

## 为什么要用它

当你让 Claude"研究一下 X 话题"时，通常会出三类问题：

1. **Token 烧得飞快。** 把 20 份 PDF 塞进对话 —— 上下文爆炸、账单也爆炸。
2. **回答会幻觉。** 貌似合理但没出处的说法，混进真实事实里不易察觉。
3. **工作无法复利。** 下周同一个话题、同一批资料，又得从头再来。

`dossier` 把任务分工：

| 角色 | 谁来做 | 做什么 |
|------|-------|--------|
| 📚 语料库 + 检索 | **NotebookLM** | 永久保存你的资料，用 `[1][2]` 引用方式回答问题、引用能回溯到源文本、不外推 |
| 🧠 编排 | **AI agent**（Claude Code / Codex / ChatGPT / …）| 筛选权威源、判断何时委托给 dossier、写代码、跑实验、合成结论 |
| 🎯 决策 | **你** | 确认源清单、判断最终结论 |

结果就是：**一个话题一个可复用的、带引用的语料库**。重复问答只花几美分而不是几美元，每条事实都能追溯到源文本。

## 适用场景

- 文献综述（多篇论文、跨论文对比）
- 公司/基金/行业尽职调查
- 跨文档的合规/法律/监管审阅
- 竞品情报收集
- 长期个人知识库（Obsidian / Readwise / 会议纪要）
- 任何**需要引用**且**同一批语料会被反复查询**的场景

## 不适用场景

- 一次性网页查询 → 直接问 Claude
- 代码库探索 → agent 自己的文件工具（Read / Grep 等）更强
- 实时数据 → NotebookLM 是静态的
- 小语料（< 5k tokens）→ 直接贴 prompt 里
- 速度比引用质量更重要 → NotebookLM 比直接对话慢约 3 倍

## 工作原理 —— 三条路线

每个新 dossier 都从一个问题开始：源怎么进来？

| 路线 | 适合 | 谁来挑源 |
|------|------|---------|
| **A · NotebookLM 自动研究** | 快速了解、探索 | NotebookLM 自己的研究 agent（通过 web search） |
| **B · Claude 策展** ⭐ | 需要权威源、长期复用 | Claude 提出清单，**你确认**，Claude 导入 |
| **C · 你指定清单** | 你自己已经知道信任哪些源 | 你给清单，Claude 只负责导入 |

Skill 的工作铁律：**绝不默默做选择 —— 把权衡摆出来，让用户决定。**

## 安装

### 1. 装底层工具

`dossier` 构建在 [teng-lin/notebooklm-py](https://github.com/teng-lin/notebooklm-py)（Python NotebookLM CLI）之上。Skill 会在首次使用时提示你装，你也可以现在手动装：

```bash
pip install "notebooklm-py[browser]"
python3 -m playwright install chromium
python3 -m notebooklm login          # 一次性浏览器 Google OAuth 登录
python3 -m notebooklm skill install  # 安装底层 `notebooklm` skill
```

> **macOS PATH 注意：** `pip install` 经常把 `notebooklm` 可执行文件放到 `~/Library/Python/<版本>/bin/` —— 这不在 `$PATH` 里。不用自己修 PATH，skill 会自动回退到 `python3 -m notebooklm` 形式。

### 2. 装这个 skill

`dossier` 遵循 [Skills 格式](https://agentskills.io/specification) —— 带 YAML frontmatter 的 markdown 文件，任何兼容 Skills 的 agent 都能加载。安装路径按你用的 agent 不同：

| Agent | 安装方式 |
|-------|----------|
| **Claude Code** | `git clone https://github.com/wangbooth/dossier ~/.claude/skills/dossier` |
| **Codex** | `git clone https://github.com/wangbooth/dossier ~/.agents/skills/dossier` |
| **多 agent 通用（Open Skills）** | `npx skills add wangbooth/dossier` |
| **ChatGPT / 其他** | 查你 agent 的文档找到 skill 目录，然后把 repo clone 进去 |

如果已经 clone 到别处，软链过去即可，不用重复 clone：

```bash
ln -sfn /path/to/dossier <你的-agent-skill-目录>/dossier
```

### 3. Jina Reader key —— 现在不用管

某些网站（最典型的是 PubMed）会激进地屏蔽 NotebookLM 的后端抓取。这时 skill 会降级到 [Jina Reader](https://jina.ai) —— 一个为 LLM 友好格式化网页的代理服务。

**你现在不用做任何事。** 第一次真的需要 Jina 时，Claude 会让你粘贴一个 key（免费，去 <https://jina.ai>，不用信用卡），然后自动帮你存到 `~/.config/dossier/config.json`（`chmod 600`，只有你自己能读）。以后全自动 —— 不用 `export`、不用改 shell 配置、也不用在终端里敲 key。

高级用户想提前配置的话，两种都行：设 `$JINA_API_KEY` 环境变量（环境变量优先级最高），或者直接写配置文件：

```json
// ~/.config/dossier/config.json
{ "jina_api_key": "jina_xxxxxxxx" }
```

## 快速开始

在你的 agent 会话里（Claude Code、Codex 等），直接描述你的研究意图：

> "我想研究下长期服用肌酸对肾功能的影响"

Claude 会识别意图、加载 skill、引导你走三路线选择。典型的 B 路线（推荐）流程：

1. Claude 扫描你已有的 notebook —— 如果有相关 dossier 就复用。
2. 否则 Claude 提出一份权威源清单（你过目/编辑）。
3. Claude 导入源、校验是否被反爬拦截（必要时走 Jina Reader 降级）。
4. 你开始提问。回答带 `[1][2]` 引用，每个引用映射到特定源的特定文本片段。
5. 下周同话题就直接复用 dossier，跳过 1-3 步。

## 示例

参考 [examples/creatine-research.md](examples/creatine-research.md) —— 一次真实的 8 源肌酸 dossier 建设（约 15 分钟），含 3 轮实测问答，展示引用深度。

## 原理

Skill 本身是纯 prompt 工程 —— 没有代码。就是一个 [Skills 格式](https://agentskills.io/specification) 的 markdown 文件，教任何兼容 Skills 的 agent：

- 何时触发（靠 description 字段）
- 怎么跑 preflight（装包检测、认证检测、PATH 回退）
- 推荐哪条路线、怎么分岔
- 九条工作铁律（清单确认、靠 `--json` 解析引用、"源外推断"分层、切换子主题前清上下文，等等）
- 交付格式（`## Dossier says / ## What I did / ## Conclusion / ## Not covered`）

所有命令都走 `notebooklm` CLI（来自 notebooklm-py）。

## 为什么选 notebooklm-py 而不是其他客户端

选型前对比了 3 个活跃的 NotebookLM 客户端。完整决策记录见 [research/client-comparison.md](research/client-comparison.md)。简要结论：

- **teng-lin/notebooklm-py**（11k★，Python）—— 最终选择。唯一有真正 `create notebook` 命令的客户端、agent 集成生态最完整、PyPI 有稳定发布。
- icebear0828/notebooklm-client（181★，TypeScript）—— 没 `create` 命令，npm 发布版落后主线。
- PleasePrompto/notebooklm-mcp（2k★，TypeScript MCP）—— 只能 chat，不管理源和 notebook；4 个月未更新。

## 已知限制

- **非官方 Google API。** NotebookLM 没有公开 API。`notebooklm-py` 用的是逆向出来的内部接口，Google 改后端时可能会挂。
- **反爬限制是真的。** 尤其是 PubMed，对云端抓取极度不友好。Skill 以 Jina Reader 作为降级方案。
- **Session token 会过期。** 开始返回认证错误时重跑 `notebooklm login` 即可（一般是几周到几个月后）。
- **源数量上限。** 免费档 50 源/notebook，Pro 档 300。规划 dossier 范围时留意。
- **生成命令慢。** `notebooklm generate audio` / `video` 每次要 5-10 分钟。

## 参与贡献

欢迎 bug report 和 PR。Skill 就一个 markdown 文件（[SKILL.md](SKILL.md)）—— 改进往往来自跑真实 dossier 时发现的"规则没覆盖到的角落"。

## 许可证

MIT —— 详见 [LICENSE](LICENSE)。
