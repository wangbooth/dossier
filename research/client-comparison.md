# NotebookLM 客户端对比：为 dossier skill 选底座

**调研日期**：2026-04-22
**调研人**：Claude（Opus 4.7）+ mimi
**决策目标**：给 `dossier` skill 选一个可靠的 NotebookLM 客户端作为底层 CLI/API

---

## 一、背景

最初选用 `icebear0828/notebooklm-client` 试点肌酸 dossier 场景，在真实使用中发现两个关键问题：

1. **版本不对齐**：npm 发布版 `0.2.0` 功能残缺（无 `source add` / `report` / `delete` 等），完整功能只存在于未发布的 GitHub main 分支，必须源码装
2. **不支持 CLI 创建空 notebook**：源码装完依然没有 `create notebook` 命令，只能依赖生成命令的 `--keep-notebook` 副作用或回退到网页手动创建

第 2 点对 dossier 场景特别致命——dossier 的典型流程是"开一个新课题 → 灌源 → 长期查"，必须能程序化地建空 notebook。

所以决定扩大选型，做一次正式的客户端对比。

---

## 二、搜索范围

### 使用的查询

- GitHub `q=notebooklm+client+in:name,description&sort=stars`
- GitHub `q=notebooklm+api&sort=stars`

### 初筛原则

- 排除 < 10 star 的冷门项目（维护风险高）
- 排除单一功能项目（只做播客、只做 PPT 反水印等）
- 排除 NotebookLM **Enterprise** 版客户端（如 `K-dash/nblm-rs`，和个人账号不兼容）
- 排除 Android/iOS 客户端等和 Agent 集成无关的

### 入围 decision round 的三家

| 项目 | ★ | 语言 | 形态 |
|---|---:|---|---|
| `teng-lin/notebooklm-py` | **11,491** | Python | CLI + Python library + Skill |
| `PleasePrompto/notebooklm-mcp` | 2,075 | TypeScript | MCP server |
| `icebear0828/notebooklm-client`（原选） | 181 | TypeScript | CLI + Skill |

---

## 三、维护活跃度与发布质量

| 维度 | teng-lin | PleasePrompto | icebear |
|---|---|---|---|
| ★ star 数 | 11,491 | 2,075 | 181 |
| 最近 push | 2026-04-21（昨天） | **2025-12-27（4 个月前）** | 2026-04-21（昨天） |
| Open issues | ~（很多但在处理） | 30 | 少 |
| 许可证 | MIT | MIT | MIT |
| 发布渠道 | **PyPI 正式版 + git tag 机制**（稳定版可信） | npm | 仅 main 分支稳定，**npm 落后** |
| 多平台 CI | **macOS / Linux / Windows 三平台跑 CI** | 未知 | 未知 |
| 文档数量 | CLI reference 925 行 + 架构/troubleshooting/stability 等 8 份 docs | 一份 README | README + 一份 SKILL.md |

**解读**：
- PleasePrompto **4 个月未 push** + 30 个 open issues，活跃度明显放缓，有停摆风险
- icebear 活跃但**发布流程不规范**（npm 版本和 main 功能集差异巨大，无 release tag）
- teng-lin **产品级发布流程**：PyPI 稳定版 + 推荐 `LATEST_TAG` 安装脚本 + main 分支明确标注为"实验版"

---

## 四、核心功能矩阵

| 功能 | teng-lin | icebear | PleasePrompto (MCP) |
|---|:---:|:---:|:---:|
| **Notebook 管理** | | | |
| `create` 空 notebook | ✅ `notebooklm create "..."` | ❌ | ❌ |
| `list` | ✅ | ✅ | ✅ |
| `rename` | ✅ | ❌ | ❌ |
| `delete` | ✅ | ✅ | ❌ |
| `use <id>` 有状态会话 | ✅ + `~/.notebooklm/context.json` | ❌ 每次传 id | 通过 MCP 状态 |
| **源管理** | | | |
| URL | ✅ | ✅ | ❌ |
| 本地文件（PDF/docx/etc.）| ✅ | ✅ | ❌ |
| YouTube | ✅ 原生 | ✅（当 URL）| ❌ |
| Google Drive | ✅ **原生** | ❌ | ❌ |
| 粘贴文本 | ✅ | ✅ | ❌ |
| Deep research + auto-import | ✅ `source add-research "..."` | ⚠️ 通过 `analyze --topic`（一次性） | ❌ |
| **提取源 fulltext**（web UI 没有）| ✅ | ❌ | ❌ |
| 刷新源 | ✅ | ❌ | ❌ |
| **对话** | | | |
| chat 单轮 | ✅ | ✅ | ✅ |
| 对话历史 | ✅ | ❌ | ✅ |
| 自定义 persona | ✅ | ❌ | ❌ |
| **保存 chat 到 notes**（web UI 没有）| ✅ | ❌ | ❌ |
| **内容生成** | | | |
| Audio podcast | ✅ 4 种格式 | ✅ 4 种 | ❌ |
| Video overview | ✅ | ✅ | ❌ |
| Slides → **PPTX**（web UI 只给 PDF）| ✅ PDF + **PPTX** | ✅ PDF | ❌ |
| Quiz → **JSON/MD/HTML**（web UI 只有交互）| ✅ 三种 | ✅ | ❌ |
| Flashcards | ✅ | ✅ | ❌ |
| Infographic | ✅ | ✅ | ❌ |
| Data table → CSV | ✅ | ✅ | ❌ |
| **Mind map → JSON**（web UI 没有）| ✅ | ❌ | ❌ |
| Report（study guide / briefing / blog / 自定义）| ✅ | ✅ | ❌ |
| **批量下载**（web UI 没有）| ✅ | 部分 | ❌ |
| **共享管理** | | | |
| 公开/私有链接 | ✅ | ❌ | ❌ |
| 用户权限（viewer/editor）| ✅ | ❌ | ❌ |
| **Agent 集成** | | | |
| 官方 Skill | ✅ `notebooklm` skill（583 行，极完整）| ✅ `notecraft`（~150 行） | N/A（MCP 本身就是集成）|
| 多账号 profile | ✅ `notebooklm profile create`、`-p` 切换 | ⚠️ 仅 `NOTEBOOKLM_HOME` 切换 | ? |
| 并行 Agent 安全 | ✅ **专门设计**，有三种隔离方案 | ❌ 无指导 | ? |
| CI/CD 友好（env var 注入 session）| ✅ `NOTEBOOKLM_AUTH_JSON` | ❌ | ? |
| Python API（非 CLI）| ✅ 完整 async API | ❌（只有 Node lib）| N/A |
| 给 Codex / Claude 打印各自 instruction | ✅ `agent show claude/codex` | ❌ | N/A |
| MCP 原生支持 | 社区有（基于 notebooklm-py 的 wrapper）| ❌ | ✅ **原生** |

---

## 五、三家的形态差异与定位

### teng-lin/notebooklm-py — 全能平台

- **定位**：把 NotebookLM 的**所有**功能暴露为 Python API + CLI + Agent Skill
- **强项**：功能完整度超越 web UI（batch download、mind map JSON、PPTX 导出、源 fulltext 提取等 web UI 没有的能力）；有完整的多账号 / 并行 agent / CI-CD 工程化设计
- **弱项**：Python 依赖（用户不介意就不是问题）；选项多学习曲线稍陡
- **迁移成本**：中等——需要卸载 npm 版，装 Python 版，重跑 `notebooklm login`

### PleasePrompto/notebooklm-mcp — 纯查询场景的极简方案

- **定位**：MCP 形态，把"chat with notebook" 暴露给 MCP-aware agent（Claude Code / Cursor / Codex / Gemini）
- **强项**：最原生的 agent 集成方式（不是 CLI → skill 调用，而是 MCP 工具直接暴露）；安装一行命令 `claude mcp add notebooklm npx notebooklm-mcp@latest`
- **弱项**：
  - **只做 chat**，不能 create notebook、不能 source add、不能生成 podcast 等制品
  - 假定 notebook 已经在 web UI 建好并灌完源——完全不覆盖 dossier 建库流程
  - 维护放缓（4 个月未更新）
  - 只能在 MCP client 用，shell 脚本 / CI 调用不了
- **适用场景**：已经有一批现成 notebook，只想让 Claude Code 查询它们。**不适合 dossier**

### icebear0828/notebooklm-client — TypeScript 版全能，但工程化差一截

- **定位**：类似 teng-lin 的全能 CLI，但 TypeScript 实现
- **强项**：TypeScript 生态；功能覆盖接近 teng-lin
- **弱项**：
  - **npm 发布流程不规范**——用户装到的是功能残缺的 0.2.0，main 才完整，且没 tag
  - **没有 `create` 命令**，空 notebook 要在网页建
  - Skill 质量较薄（150 行 vs teng-lin 583 行），多账号 / 并行没覆盖
  - 社区规模小 63 倍

---

## 六、对 dossier skill 的特定评估

dossier skill 的核心需求是什么？回到设计文档：

1. **程序化建新 notebook**（每个课题一个 dossier）→ 必须有 `create`
2. **灵活加源**：URL、PDF、YouTube、Google Drive → 全覆盖加分
3. **Deep research 作为路线 A 的支撑** → 必须有 `source add-research` 或等价
4. **Chat with history**（dossier 是多轮问答）→ 有历史加分
5. **长期跨 session 工作**（dossier 会被反复用）→ 多账号 / profile 隔离加分
6. **未来扩展**：生成 podcast / slides / quiz（第三、五部分的内容工厂模式）→ 全覆盖加分
7. **Agent 集成友好**：Autonomy rules、CI/CD、并行 → 专门设计加分

**按需求打分（1-5）**：

| 需求 | teng-lin | icebear | PleasePrompto |
|---|:---:|:---:|:---:|
| 1. 程序化建 notebook | 5 | 1 | 1 |
| 2. 多样化源 | 5 | 4 | 1 |
| 3. Deep research | 5 | 3 | 1 |
| 4. Chat with history | 5 | 3 | 4 |
| 5. 多账号/并行 | 5 | 2 | 2 |
| 6. 内容生成扩展 | 5 | 4 | 1 |
| 7. Agent 工程化 | 5 | 3 | 3 |
| **合计** | **35/35** | 20/35 | 13/35 |

---

## 七、决策：换到 teng-lin/notebooklm-py

### 理由

1. **唯一满足"程序化建空 notebook"**——这是 dossier 场景的硬门槛
2. 功能矩阵全面碾压
3. 工程化成熟（PyPI 稳定发布、三平台 CI、完整 docs、多账号/并行设计）
4. Agent 集成最深（官方 skill 583 行，覆盖 autonomy rules / CI/CD / 并行隔离）
5. 社区规模最大（11k+ star，63 倍于 icebear），长期可持续性好
6. 符合我们 `CLAUDE.md` 的语言偏好（Python 优先）

### 和我们 dossier skill 的关系

teng-lin 的 `notebooklm` skill 是 **底层工具 skill**（"怎么调每个 CLI 命令"），dossier skill 是 **方法论 skill**（"什么时候该用 NotebookLM、走三路线哪条、输出什么格式"）。两者**互补不冲突**——dossier skill 会在运行时调用 `notebooklm create / source add / source add-research / chat / ask` 等命令。

### 需要注意

- teng-lin 的 skill 名是 `notebooklm`（icebear 的是 `notecraft`），调用命令时要用 `notebooklm` 前缀
- `notebooklm login` 需要开浏览器（playwright chromium），要重新授权一次
- 多账号通过 `profile` 管理而非 `NOTEBOOKLM_HOME` 目录（更优雅）

---

## 八、迁移行动项

1. [ ] 卸载 icebear 版：`npm uninstall -g notebooklm-client` + 删掉 `/tmp/notebooklm-client`
2. [ ] 安装 teng-lin 版：`pip install "notebooklm-py[browser]"` + `playwright install chromium`
3. [ ] 重新授权：`notebooklm login`
4. [ ] 安装底层 skill：`notebooklm skill install`
5. [ ] 验证：`notebooklm list` 应能列出原有 notebook（同一 Google 账号）
6. [ ] 删除 icebear 相关的 skill（`~/.claude/skills/notecraft` 如有）避免和 `notebooklm` 冲突
7. [ ] 更新 dossier skill 设计：命令前缀改 `notebooklm`、利用 `create` / `use` / `ask` 等原语改写流程
8. [ ] 删除本文档中的 icebear 相关决策文字，保留本文作为**选型决策记录**

---

## 九、保留这份对比的理由

1. **未来发 GitHub 时**：README 里会被问"为什么选 teng-lin"，这份文档提供完整答案
2. **版本演化跟踪**：如果 teng-lin 停摆或 icebear 追上来，这份对比的维度可以重新跑一次
3. **给其他用户参考**：dossier skill 公开后，用户如果想换底层客户端，这份对比能帮他们评估

