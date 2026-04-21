# AI 搜索 API 对比任务中 Twin 工具的低效行为复盘 — Case

> 日期：2026-04-21
> 关联 agent：`ai-search-api-comparison`（build + continue_build，共 2 次 run）
> 用户原始请求：「对比 Exa、Perplexity Sonar、Tavily 三家 AI 搜索 API，用简体中文写成 Markdown 报告发到一个公开 GitHub 仓库」

## 一句话

一次看似常规的「抓文档 + 对比」任务，两次 build 分别被 `deep_research` 120s 超时 和 `llm` 单字符串静默截断各绊倒一次；最终通过 **scrape + 分片 llm + 带 sha 的 put_github_file** 的组合把仓库补成四份完整文档，同时暴露出 TW 工具链里 6 处可被直接改进的低效行为。

## 任务时间线

1. 用户要求做 Exa / Perplexity Sonar / Tavily 三家 AI 搜索 API 的对比分析 → 第一次 build 在编排时首选 `deep_research` 做综合调研 → 120 s 超时失败 → 回退 `web_search` + `scrape` 完成任务并推到 GitHub。
2. 交付后用户指出 `docs/exa.md` 内容明显偏短，要求 chat 层直接读文件回显；orchestrator 调 `file.download_via_curl` 被拒（「unsupported file action for workspace chat」），诊断只能再推回 agent。
3. 第二次 build（continue_build）在 agent 侧拉取 GitHub 文件，确认 `docs/exa.md` 实际只有 **267 字符 / 537 字节**（标题 + 半句话）；第一次 build 的 `llm` 长文生成被静默截断，却以 SUCCESS 收尾。
4. 重写策略：对 Exa 官方文档再做 **8 次端点级 `scrape`**（`/reference/search`、`/reference/answer`、`/reference/rate-limits`、`/reference/quickstart`、`/pricing`、`/changelog`、`docs.exa.ai` 首页、landing），然后用 **三块分片 `llm` 生成**（概览 / 端点与参数 / 限流与计费 & 变更），拼接后写盘。
5. 更新三份正文（exa.md / perplexity.md / tavily.md）与 README；每次 update 都需要先 `get_github_file` 取 sha，再把 sha 回传给 `put_github_file` 完成更新。
6. 最终产出：README 14 482 字 / 19 小节、exa.md 30 922 字 / 47 小节、perplexity.md 34 083 字 / 61 小节、tavily.md 30 733 字 / 51 小节，commit 到公开仓库收尾。

## 用到的工具

| 工具 | 调用次数 | 成功 | 失败 | 备注 |
|---|---|---|---|---|
| deep_research | 1 | 0 | 1 | 120 s 超时，被迫降级 |
| web_search | 若干 | 全部 | 0 | 用于第三方对比 / benchmark 收集 |
| scrape | 20+ | 全部 | 0 | 第二次 build 仅 Exa 一家就 8 次端点 scrape |
| llm | 多次 | 部分 | 1 次静默截断 | 单字符串长中文生成被截到 267 字符 |
| get_github_file | 每次 update 1 次 | 全部 | 0 | 仅为拿 sha 而存在 |
| put_github_file | 多次 | 全部 | 0 | 需外部 get → put 两步 |
| file.download_via_curl（chat 层）| 1 | 0 | 1 | workspace chat 不支持 |

## 每个工具的观察

### `deep_research`

- **卡点：** 工具描述称「1-3 分钟」完成深度研究，实测在 **120 s 硬超时**处被中断，没有 partial / citations 返回。
- **Workaround：** 直接换 `scrape`（抓官方文档）+ `web_search`（拿第三方对比）+ `llm`（合成）三段式管线。对「有明确源 URL 的对比分析」这条管线比 `deep_research` 稳得多。
- **耗时 / 成本：** 多走一次失败重试路径 ≈ 2 min 浪费 + 额外 credit 消耗；更大的代价是 build 第一条路径被堵死，用户等待拉长。
- **原文引用：** 工具描述写「takes 1-3 minutes to complete」，但实际 timeout 定在 120 s。
- **建议：** 超时上限 ≥ 180 s、或开放 `timeout_ms` 让调用方声明、或支持「超时返回当前已收集的 partial 结果 + citations」。
- **观察等级：[已观测]**

### `llm`（长中文 Markdown 生成）

- **卡点：** 一次生成长 Markdown 单字符串字段时，输出 token 耗尽不是抛错，而是**静默截断**。具体表现：第一次 build 写出的 `docs/exa.md` 只有 **267 字符 / 537 字节**（仅剩「# Exa」标题 + 半句导语），无任何 warning，`finish_run` 正常 SUCCESS。截断是用户后续人工发现的。
- **Workaround：** 第二次 build 显式**分片生成**——把 exa.md 拆成「概览 / 端点与参数 / 限流与计费 & 变更」三段分别调 `llm`，再在 JS 里拼接写盘。拼接后恢复到 30 922 字 / 47 小节。
- **耗时 / 成本：** run 以「成功」收尾但交付物残缺，用户被迫充当 QA；二次 build 重做一整条管线的代价 ≈ 10+ 次工具调用。
- **原文引用：** 没有任何错误码或元信息提示 truncation。
- **建议：**（a）output token 耗尽应显式抛错或在结果上挂 `truncated: true` flag；（b）内置「长文模式」自动按章节分片并拼接；（c）每次返回附带 prompt / completion token 用量，方便调用方做完整性校验。
- **观察等级：[已观测]**

### `file.download_via_curl`（workspace chat 层）

- **卡点：** 用户贴来 `https://github.com/.../docs/exa.md` 的 URL 让 chat 直接读一眼，orchestrator 调 `file.download_via_curl` 被拒，提示「unsupported file action for workspace chat」。chat 层没有任何只读 fetch / peek 能力，只能再启一轮 build，把诊断推回 agent。
- **Workaround：** 新开一个 continue_build，在 agent 侧用 `get_github_file` 拿 base64，再本地 decode；等价于绕一圈去读一个公开 URL。
- **耗时 / 成本：** 诊断 → 修复循环多了一轮（≈ 1 次 build 启停 + 一轮用户等待）。
- **原文引用：** 工具返回固定字符串 `unsupported file action for workspace chat`。
- **建议：** chat 层开放只读 fetch / peek（至少对 `http(s)://` 公开 URL，大小上限可设），避免把「看一眼文件」升级成「启 build」。
- **观察等级：[已观测]**

### `put_github_file`（update 路径）

- **卡点：** 更新已存在文件必须先 `get_github_file` 拿 blob sha，再回传给 `put_github_file`。每次 update = 两次 API 往返；并发修同一文件时还需要自己处理 409 冲突重试。
- **Workaround：** 在 agent 侧显式编排 get → put 两步；一次 build 重写 4 份文档时串行跑了 4 轮 get + 4 轮 put。
- **耗时 / 成本：** 每份文件多一次 GET 往返；并发场景下必须自己加冲突重试逻辑，调用方代码复杂度上升。
- **原文引用：** GitHub Contents API 的 PUT 语义要求对已存在文件提供 `sha`；工具层没有封装。
- **建议：** 提供 `upsert_github_file`，内部自动拿 sha 并在 409 时重试；调用方只传 path + content + message 即可。
- **观察等级：[已观测]**

### write 完成后的完整性校验（agent 行为 + 工具层缺失）

- **卡点：** 第一次 build 把 267 字节的 exa.md 推上去之后直接 `finish_run` SUCCESS，没有任何「这个文件是否符合预期大小 / 小节数 / 必需小节」的后置断言。agent 自己根本不知道交付的是残次品。
- **Workaround：** 第二次 build 内建了 post-write validator——对每份 md 做最小字符数（≥ 8 000 字）+ 最小小节数（≥ 15）+ 必需小节清单匹配；不达标则回到生成步骤。
- **耗时 / 成本：** 没有 validator 的成本 = 一整轮用户发现问题 + 二次 build；加上 validator 的成本 = 每份文件末尾额外一次 readFile + 计数。
- **建议：** 平台侧提供可声明式配置的 post-write check（minChars、minSections、requiredSections），或至少在 `put_github_file` / `write_file` 工具结果里回传 bytes 与行数，方便 agent 自查。
- **观察等级：[已观测]**

### 工具选择偏差（对`deep_research`的首选）

- **卡点：** 第一次 build 面对「三家 API 深度对比」这类明显属于「有确定文档 URL 的结构化对比」的任务，第一反应直接挑 `deep_research`，结果 120 s 超时。实际 `scrape` + `web_search` + `llm` 的组合对这类任务更稳，第二次 build 也正是这条管线一次打穿。
- **Workaround：** 显式在 instruction 里写「对有明确文档 URL 的对比分析，优先使用 `scrape` + `web_search` + `llm`；`deep_research` 留给无明确源的探索性研究」。
- **耗时 / 成本：** 不是工具本身的 bug，但工具描述没有帮 agent 做出这个分流判断。
- **建议：** 在 `deep_research` 工具描述中明确区分两类场景——「无明确源的探索性研究」vs「有明确文档 URL 的对比分析」；后者引导去 `scrape` + `web_search`。
- **观察等级：[已观测]**

## 关联数据点

- **两次 build：** 一次初始 build + 一次 continue_build（具体 run_id 略）。
- **第一次 deep_research：** 120 s 超时失败。
- **exa.md 第二次 build 前：** **267 字符 / 537 字节**（标题 + 半句话）。
- **重写后（第二次 build 产出）：**
  - `README.md`：14 482 字 / 19 小节
  - `docs/exa.md`：30 922 字 / 47 小节
  - `docs/perplexity.md`：34 083 字 / 61 小节
  - `docs/tavily.md`：30 733 字 / 51 小节
- **补齐过程：** 8 次 Exa 端点文档 `scrape` + 三块分片 `llm` 生成（概览 / 端点与参数 / 限流与计费 & 变更） + 每份文件一轮 `get_github_file`（取 sha） + 一轮 `put_github_file`（带 sha 更新）。

## 最终产出

- 对比报告仓库（公开）：README + `docs/exa.md` + `docs/perplexity.md` + `docs/tavily.md`，合计约 110 000 中文字符，涵盖端点、参数、限流、计费、SDK、变更日志。
- 本案例文件：`cases/2026-04-21-ai-search-api-comparison.md`。

## 关联

- 工具页更新建议：`tools/deep_research.md`、`tools/llm.md`、`tools/github_http_tools.md`、`tools/scrape.md`。
- 新增 improvements 建议（与现有 IMP-007 / IMP-009 / IMP-012 关联）：
  - `improvements/IMP-xxx-deep-research-timeout-configurable.md`（deep_research 超时 ≥ 180 s 或可配）
  - `improvements/IMP-xxx-llm-truncation-explicit-error.md`（llm 输出 token 耗尽显式报错 + 长文分片模式）
  - `improvements/IMP-xxx-chat-layer-readonly-fetch.md`（workspace chat 开放只读 fetch）
  - `improvements/IMP-xxx-github-upsert-file-wrapper.md`（`upsert_github_file` 封装 get + put + 409 重试，延续 IMP-009 的方向）
  - `improvements/IMP-xxx-post-write-validator.md`（write 完成后内建完整性校验）
  - `improvements/IMP-xxx-tool-selection-hint-deep-vs-scrape.md`（`deep_research` 描述中明确 vs `scrape + web_search` 的分流）

## 自省（Agent 行为层面）

以下三条是本次执行中 agent 自己的问题，与工具本身的 bug 分开记录：

1. **第一次 build 没在 `deep_research` 超时后立刻做成果校验。** 120 s 超时回退到 `web_search` + `scrape` 之后，agent 专注于"把流程跑通到 finish_run SUCCESS"，没有对最终写入 GitHub 的每个文件做字符数 / 小节数断言；`llm` 静默截断产生的 267 字节残次品就这样带着 SUCCESS 发出去了。这是 agent 行为问题，不是 `llm` 工具本身的问题。[已观测]
2. **把「生成长中文 Markdown」这种显然容易撞 token 上限的子任务直接丢给单次 `llm` 调用。** 既没有预估 token、也没有先试探性分片。等到第二次 build 才被迫切成三片；如果第一次 build 就按章节分片，残缺交付可以完全避免。[已观测]
3. **对「工具返回 SUCCESS」与「产出符合预期」的差别反应迟钝。** `finish_run` 返回 SUCCESS 不等于交付物正确；本次正是把"工具链没报错"等同于"任务完成"，绕过了最基本的输出校验。后续应把 post-write 断言作为 finish 前的必走步骤。[已观测]
