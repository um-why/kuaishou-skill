# 快手 Skill 深度分析与优化后档

| 字段 | 内容 |
| --- | --- |
| 日期 | 2026-11-04 |
| 范围 | `kuaishou-skill`（本仓库）+ 依赖后端 `D:\Git\guaikei` 快手链路 |
| 版本快照 | skill `1.1.0`；后端 `app/kuaishou` + `mcp/kuaishou` |
| 状态 | 待跟进（问题编号可勾选） |
| 目标读者 | 产品 / 后端 / Skill 作者；后续迭代按优先级消化 |
| 位置 | 仓库根目录 `todo.md` |

本文档是**本地后档**：列出问题、优化方向、多模型/多 Agent 适配建议。后续改动请在文末「跟进记录」追加，不要静默覆盖已关闭项的结论。

---

## 0. 项目画像（宏观）

```
用户 / AI Agent
    │
    ├─ Cursor / Codex / Workbuddy … 读 SKILL.md → 执行 node CLI
    │
    ▼
kuaishou-skill (纯 Node，无 npm 依赖)
  search-cli / post-cli / comment-cli
  create 任务 → 本地轮询 get info → stdout JSON 信封
    │
    ▼ HTTPS TOKEN header
www.guaikei.com
  /api/kuaishou/{search,post,comment}/{keyword|url|info}
    │
    ▼ 异步队列 + 爬虫侧更新
guaikei server (Go)  ← 另有 MCP 同源工具 kuaishou_search/post/comment
```

**定位**：把 guaikei 快手开放 API 包成「大众 + Agent」可调用的只读技能。  
**已有亮点**：统一输出信封、严格参数失败、路由表清晰、schema 齐全、与 MCP 侧行为大体对齐。  
**核心张力**：异步长轮询 + 魔法数字枚举 + 类型不一致，对弱模型与短超时 Agent 不友好；人类营销文案与 Agent 指令混杂。

---

## 1. 问题清单（全部）

优先级：`P0` 阻断正确性/可用性 · `P1` 显著影响 Agent 准确率或成本 · `P2` 体验/可维护 · `P3` 锦上添花。

### 1.1 与后端契约 / 正确性

| ID | 优先级 | 问题 | 证据 / 影响 |
| --- | --- | --- | --- |
| C-01 | P0 | ~~**包月 TOKEN 限额未写入 Skill**~~ ✅ | 已对齐 douyin-search-claw：文档套餐双上限 + `errcode 14`→`PLAN_LIMIT`（搜索≤30 / 作品≤50 / 评论≤60）。见 1.1.1。 |
| C-02 | P0 | **计费/余额不足路径未文档化** | `errcode 15` 等；Agent 可能当 `API_ERROR` 乱重试或改参数浪费额度。 |
| C-03 | P1 | **Skill CLI 不补全缺失 URL，MCP 会补** | `mcp/kuaishou/search.go` `processSearchResults` 会补视频/主页 URL；JS 直出后端原始 data。Agent 链式「搜→评/作品」时可能缺 `url`。 |
| C-04 | P1 | **关键词特殊字符规则与 MCP 不一致** | MCP 禁 `&`；Skill `isKeywordValid` 不禁。同一产品两通道行为分裂。 |
| C-05 | P1 | **关键词长度计量可能不一致** | Skill 用 `string.length`（UTF-16）；后端 `for range`（rune）。极端字符下 CLI 放行、后端 13，或相反。 |
| C-06 | P1 | **任务复用窗口表述偏粗** | Create 去重 ~20min；Get 结果窗口 search ~29min、post/comment ~30min。Skill 写「约 20 分钟」对 TIMEOUT 重试大体够用，但边界会「创建成功却长期 pending」。 |
| C-07 | P1 | **互动数字类型分裂（设计债）** | 搜索/评论：number；作品 post 字段：string。已在 SKILL 警告，但仍是 Agent 排序/比较第一大坑。 |
| C-08 | P2 | **字段命名混用 camelCase / snake_case** | 如 `collect_count` vs `recToFriendCount`、`forward_count` vs `forwardCount`。schema 已注明，模型仍易写错路径。 |
| C-09 | P2 | **无视频详情命令** | 已声明；用户要「解析这条视频」只能评论或搜。能力缺口需在 description 更醒目。 |
| C-10 | P2 | **后端非法枚举静默改写，Skill 严格失败** | 后端 Sort/Time/Duration 非法变 0；Skill 直接 `INVALID_ARGS`。对大众更安全，但直连 API 的其他客户端与 Skill 行为不同——需在官网文档统一说明「开放 API 侧也建议严格」。 |
| C-11 | P2 | **Create 限流 errcode 2 与「处理中」errcode 1 易混淆** | Skill 创建阶段对 2–9 可重试；查询只把 `1` 当 pending。文档对 Agent 说「十进制数字原样转告」正确，但缺少「常见码表」卡片。 |
| C-12 | P3 | **`omitempty` 导致 0/空字段缺失** | 比较「零赞」需区分缺失与 0；分析脚本易误判。 |

### 1.2 Agent / Skill 指令层（Cursor、Codex、Workbuddy 等）

| ID | 优先级 | 问题 | 证据 / 影响 |
| --- | --- | --- | --- |
| A-01 | P0 | **轮询最长约 15 分钟，远超多数 Agent 工具超时** | `QUERY_MAX_ATTEMPTS=20`，退避至 60s。Cursor/Codex shell 常 30s–120s 被杀 → 假超时、重复 create、撞限流。 |
| A-02 | P0 | **`sort` 语义在 search / post 相反且默认不同** | search：`1=最新`；post：`0=最新`，默认 `1=最热`。弱模型（及部分中等模型）高频调反。 |
| A-03 | P0 | **时间参数是「天数枚举」不是「近 N 天」** | `--time 2` 非法；一周必须是 `7`。文档有，但 description/路由表需「禁止发明中间值」更靠前。 |
| A-04 | P1 | **stderr 彩色 Banner + 进度刷屏污染上下文** | Agent 把 stderr 吸入上下文；ANSI、emoji、横幅浪费 token，干扰「只读最后一份 JSON」规则。 |
| A-05 | P1 | **缺少机器友好的静默/JSON-only 模式** | 无 `--quiet` / `KS_QUIET=1`；无单独 `--json`（实际已是，但 stderr 仍吵）。 |
| A-06 | P1 | **SKILL.md 信息密度高、偏中文长文** | ~220 行可接受但接近「全量灌进上下文」。英文优先 Agent（GPT/Claude/Gemini/Grok）对中文路由表遵循率下降；中文模型（GLM/DeepSeek/Kimi）相对更好但仍易漏「limit 0」「不要两个命令都试」。 |
| A-07 | P1 | **无 AGENTS.md / tools 清单 / 一页纸决策树给「只读 description」的宿主** | 部分宿主只注入 frontmatter description；当前 description 已不错，但缺「禁止事项」与「超时策略」压缩版。 |
| A-08 | P1 | **无 MCP 入口在本仓库** | guaikei 已有 MCP；大众用 CLI Skill。Cursor/Codex 更爱 MCP tool schema。双栈未说明「何时用 Skill、何时用 MCP」。 |
| A-09 | P1 | **Windows 引号与 PowerShell 执行差异** | 示例偏 bash；Codex/Cursor 在 Win 上易把中文关键词拆参。缺 PowerShell 示例与「优先 `--keyword=` 内联」硬规则。 |
| A-10 | P1 | **工作目录约定脆弱** | 「在技能根目录执行」；Agent 常在用户项目 cwd 跑相对路径失败。缺「解析 SKILL 所在目录」或 `package.json` `bin` + `npx`。 |
| A-11 | P2 | **成功时打印完整 pretty JSON** | limit 大时 stdout 巨大，Agent 二次摘要失败/截断；缺 `--fields` 或「摘要模式」。 |
| A-12 | P2 | **readme 营销向，易被 Agent 误读为指令** | 「1W 条」「趋势预测」等与真实能力夸大；若宿主索引全仓 md 会污染。 |
| A-13 | P2 | **metadata 多宿主字段混杂** | `openclaw` + 自研 `requires`/`schemas`；Cursor 标准 skill 字段更简。兼容性靠「忽略未知」，但缺少各宿主适配说明。 |
| A-14 | P2 | **错误退出码：多个语义挤在 exit 1** | TIMEOUT / NETWORK / INVALID_URL / API 同为 1；Agent 更应读 `error_code`，弱模型却只看 exit code。 |
| A-15 | P3 | **无「评测用例集」给 Agent** | 无 golden prompts（爆款/粉丝/评论/歧义 3x id），无法回归各模型路由准确率。 |

### 1.3 工程与质量

| ID | 优先级 | 问题 | 证据 / 影响 |
| --- | --- | --- | --- |
| E-01 | P1 | **无自动化测试** | validate/url、args、envelope、与后端 fixture 均无。回归靠人工。 |
| E-02 | P1 | **无 `engines`、无 `bin`、无 lockfile** | Node 版本承诺只在文档；不可 `npm i -g` 成命令。 |
| E-03 | P1 | **BASE_URL 写死，无环境覆盖** | 无法指向 staging；调试与灾备困难。 |
| E-04 | P2 | **包名/技能名/上报名略乱** | `package.json`/`SKILL` name=`kuaishou-data-miner`；`name.js` fallback=`kuaishou-skill`。统计与支持排查分裂。 |
| E-05 | P2 | **仅 HTTPS 模块，无代理/自定义 CA 说明** | 企业网络、部分海外 Agent 环境可能失败；错误只见 NETWORK。 |
| E-06 | P2 | **日志默认关，开后写临时目录** | 大众难找；Agent 更不会设 `KS_GUAIKEI_ENABLE_LOG`。排障弱。 |
| E-07 | P2 | **无 CI（lint/test）** | 对「普罗大众」分发缺质量门禁。 |
| E-08 | P3 | **依赖零是优点，但也无 schema 运行时校验** | 出参靠约定；后端加字段/改类型不会被发现。 |

### 1.4 产品、合规、大众化

| ID | 优先级 | 问题 | 证据 / 影响 |
| --- | --- | --- | --- |
| P-01 | P1 | ~~**微信手机号硬编码在 Skill/readme**~~ ✅ | 已改为仅官网入口；`SKILL.md` / `readme.md` 不再含微信手机号。未升版本号。 |
| P-02 | P1 | **能力表述偏营销** | 「趋势预测」「穿透数据层」等超出只读抓取；信任与合规风险。 |
| P-03 | P1 | **合规边界对 Agent 偏软** | 已有「公开只读」；缺：不得用于骚扰、批量爬取转售、绕过平台 ToS 的明确拒绝话术模板。 |
| P-04 | P2 | **TOKEN 获取摩擦高** | 未配置就劝开通正确；但无「沙箱/只读 demo keyword」降低试错成本。 |
| P-05 | P2 | **中英双语缺失** | 面向全球 Agent 运行时（Claude/GPT）缺英文 SKILL 或 bilingual description。 |
| P-06 | P2 | **与 guaikei 多平台技能生态未对齐说明** | 用户问抖音时「不要用本技能」有；未引导「姊妹 skill/MCP」。 |
| P-07 | P3 | **MIT + 商业 API TOKEN** | 开源客户端 + 闭源付费 API 模式需在 readme 一句说清，避免「开源即可白嫖数据」误解。 |

### 1.5 安全

| ID | 优先级 | 问题 | 证据 / 影响 |
| --- | --- | --- | --- |
| S-01 | P1 | **TOKEN 仅环境变量，无本地加密存储指引** | Agent 可能把 token 写进命令行历史或聊天；需「禁止回显 token」硬规则。 |
| S-02 | P2 | **请求头自定义 `TOKEN` 非 Bearer** | 非常规；代理/网关可能剥头；文档需强调。 |
| S-03 | P2 | **成功日志可含用户查询内容** | 开启 log 时关键词/URL 落盘；多用户机器注意。 |
| S-04 | P3 | **无请求签名/设备绑定** | 属后端；Skill 侧至少不要在错误里打印完整 token。当前未打印，保持。 |

---

## 2. 优化方向（专业宏观 + 可落地动作）

### 2.1 战略原则（先定调）

1. **Agent-first，人类文档第二**：CLI 的默认输出应对机器稳定；人类说明放 `readme.md` / 官网。  
2. **一条通道一种契约**：Skill CLI 与 guaikei MCP **共享同一套校验与结果规范化**（能抽共享规则表最好）。  
3. **短超时可完成**：任何「一次工具调用」应在 **30–90s 内有确定性终态**（成功 / empty / 明确 `PENDING` 续查句柄），而不是卡 15 分钟。  
4. **消灭魔法数字**：对外用字符串枚举（`sort=hot|latest|comprehensive`），对内再映射 0/1/2。  
5. **统一数值类型**：出参全部 number（或全部 string + 明确 parse 规则）；推荐 number。  
6. **分层披露**：`SKILL.md` ≤ 120–150 行决策核心；细节进 `references/`；schema 只在需要时读。

### 2.2 P0 必做（建议 1–2 个迭代）

| 方向 | 具体怎么做 |
| --- | --- |
| **异步任务可续跑** | ① create 返回 `task_id`（需后端配合）或 ② CLI 增加 `--wait-max-sec 45` 默认，超时输出 `error_code=PENDING` + `request` 原样 + 提示「同参重入」；③ SKILL 明确：宿主超时 ≠ 失败，必须同参重试。 |
| **限额与计费写入 Skill** | 在「参数」前增加卡片：包月 limit 上限；余额不足停止；禁止为「全部」设 10000。readme 删掉不设上限的「1W」话术或改为「上限视套餐」。 |
| **消歧 sort** | 短期：SKILL 顶部加「易错对照表」加粗。中期：CLI 接受 `--sort latest\|hot\|mixed`，数字保留兼容。 |
| **静默模式** | `KS_QUIET=1` 或 `--quiet`：无 Banner、无 ANSI、进度可一行或关闭；仅 stdout JSON。默认对非 TTY 自动 quiet。 |

### 2.3 P1 高价值

| 方向 | 具体怎么做 |
| --- | --- |
| **结果规范化层** | 对齐 MCP：补 `url`；作品互动字段 `Number()`；统一 `user_id` 字符串化建议。 |
| **校验对齐 MCP** | 禁 `&`；关键词长度改 rune/code point；单测对齐 Go。 |
| **安装与路径** | `package.json` 加 `bin`：`kuaishou-search` 等；SKILL 示例改为 `npx`/`node <绝对或技能根>`；提供 `scripts/resolve-root.js` 或在文档写「先 `cd` 到含 SKILL.md 的目录」。 |
| **错误码表** | `references/error-codes.md`：1/2/11–15/401…；SKILL 只留 Agent 动作列。 |
| **双栈说明** | `references/mcp-vs-cli.md`：有 guaikei MCP 用 MCP；无则用本 Skill。 |
| **评测集** | `docs/eval/prompts.jsonl`：20–50 条中英用户话术 + 期望 command/args；用各模型跑准确率。 |
| **测试** | Node 原生 test 或最小 assert：url 路由、args、keyword、envelope。 |
| **合规与联系方式** | Skill 内联系方式改为官网；手机号移出仓库公开文件。 |

### 2.4 体验与产品（大众化）

| 方向 | 具体怎么做 |
| --- | --- |
| **摘要输出** | `--format summary`：标题、作者、赞、链、时间；默认 full JSON 给流水线。Agent 默认 summary 可写进 SKILL。 |
| **英文 SKILL** | `SKILL.en.md` 或 description 中英并列 WHAT/WHEN；body 英文决策树 + 链到中文 options。 |
| **场景配方** | 「爆款选题」「竞品主页」「舆情评论」三套固定参数配方，减少枚举推理。 |
| **Demo** | 官网提供固定 demo token + 固定 keyword 只读配额。 |

### 2.5 后端协同（guaikei）

| 方向 | 具体怎么做 |
| --- | --- |
| **开放 API 严格模式** | 与 Skill 一致：非法枚举返回 11/12 而非改写，避免「同参不同结果」。 |
| **返回 task_id + status** | 支持短轮询与跨进程续查。 |
| **规范化出参** | 服务端统一 number + 补 URL，CLI/MCP 双受益。 |
| **Webhook/长轮询可选** | 企业与 Agent 平台更友好。 |

---

## 3. 多模型 × 多 Agent 适配策略

### 3.1 模型能力差异（对 Skill 设计的含义）

| 模型族 | 相对强项 | 相对弱项 | Skill 应对 |
| --- | --- | --- | --- |
| **Claude / GPT** | 长文档遵循、工具选择、自我纠错 | 超时后易「换参重试」；英文环境读中文表有损耗 | 英文关键路径；PENDING 同参；枚举改字符串 |
| **Gemini** | 多模态、长上下文 | 偶发跳步、忽略负向约束 | 负向规则前置（不要、禁止）短句列表 |
| **Grok** | 指令跟从快 | 易过度调用、limit 偏大 | 默认 limit 强调；配方固化 |
| **DeepSeek** | 中文路由、性价比、代码 | 长约束中后部遗忘 | 决策树置顶；细节外置 references |
| **GLM** | 中文业务理解 | 复杂枚举易混 | 中文别名表（最新/最热）映射 |
| **Kimi** | 长文、中文检索感 | 偶发「两个命令都试」 | 「命中即停」写进 description |

### 3.2 Agent 宿主差异

| 宿主 | 典型用法 | 适配要点 |
| --- | --- | --- |
| **Cursor** | 读 SKILL.md + shell | 短 SKILL；quiet；cwd；`bin` |
| **Codex** | 自主跑命令、偏英文 | 英文 description；PowerShell；超时策略 |
| **Workbuddy 等** | 可能只注入 description / MCP | 强化 frontmatter；提供 MCP 文档 |
| **OpenClaw 等** | 已有 metadata 字段 | 保持 `primaryEnv`；补 emoji 以外的 requires |

### 3.3 「更容易、更准确」的具体设计清单

1. **Description 模板（建议改写方向）**  
   WHAT + WHEN + NOT WHEN + 三命令名 + 环境变量 +「超时同参重试」+「禁止发明 time=2」。  
2. **决策树改成伪代码**（弱模型更稳）：

   ```
   if 要评论 and 有视频 → comment
   else if 要博主/粉丝/作品 → post
   else if 有关键词 → search
   else ask
   ```

3. **参数用自然语言别名**（CLI 层）：`--sort hot`、`--time week`。  
4. **输出给用户 vs 给模型分离**：SKILL 规定「对用户只摘 5 字段」；stdout 仍完整（或 summary）。  
5. **Eval 门禁**：发版前跑中英各 20 prompt，DeepSeek/GLM/Claude/GPT 路由准确率阈值。

---

## 4. 建议落地路线图

| 阶段 | 周期感 | 交付 |
| --- | --- | --- |
| **M1 止血** | 短 | quiet 默认非 TTY；限额写入 SKILL/readme；易错 sort/time 置顶；联系方式改官网；PENDING/wait-max |
| **M2 对齐** | 中 | 出参规范化；校验对齐 MCP；error-codes.md；bin + engines；单测 |
| **M3 Agent 化** | 中 | 字符串枚举；summary 格式；英文 SKILL；eval 集；与后端 task_id |
| **M4 生态** | 长 | 官方推荐 MCP；多平台姊妹 skill 导航；CI |

---

## 5. 问题索引（便于勾选跟进）

**契约**：C-01 … C-12  
**Agent**：A-01 … A-15  
**工程**：E-01 … E-08  
**产品**：P-01 … P-07  
**安全**：S-01 … S-04  

合计 **46** 项。关闭时请在「跟进记录」写：`ID`、日期、提交/PR、备注。

---

## 6. 跟进记录

| 日期 | ID | 动作 | 备注 |
| --- | --- | --- | --- |
| 2026-11-04 | — | 创建本后档 | 初版深度分析，未改业务代码 |
| 2026-11-04 | — | 迁至仓库根目录 `todo.md` | 原 `docs/archive/` 副本已移除 |
| 2026-11-04 | C-01 | 落地套餐双上限 + `PLAN_LIMIT` | 对照 `douyin-search-claw`；版本 1.1.1 |
| 2026-11-04 | P-01 | 去掉微信手机号 | 仅保留 guaikei.com；不升版本号 |

---

## 7. 参考路径

- Skill：`SKILL.md`、`scripts/**`、`assets/*.schema.json`
- 后端开放 API：`guaikei/server/app/kuaishou/{search,post,comment}.go`
- 后端 MCP：`guaikei/server/mcp/kuaishou/*.go`
- 路由：`guaikei/server/service/router.go`（`utility` 组）
