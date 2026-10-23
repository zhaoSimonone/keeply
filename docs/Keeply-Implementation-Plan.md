# 收好 Keeply：可执行开发任务书

版本：v0.1；日期：2026-10-23。本文用于直接交给本地编码 Agent 实施。

## 0. 给执行 Agent 的指令

你是本项目的全栈工程师。目标是交付真实可运行的 Chrome 插件、网页管理端和后端，不是只交付界面原型或设计文档。按本文阶段逐步实现；每阶段完成实现、相关验证和进度记录，再继续下一阶段。一般工程选择自行判断，不必逐项询问。不要扩大到社区、付费、团队协作或全自动网络爬虫。

开始前读取当前仓库 README、AGENTS.md、已有代码及工具版本；已有项目优先沿用兼容技术栈，空目录则使用下述默认方案。不要覆盖用户未提交的修改。依赖选择实施时验证过的稳定版本并提交锁文件，不凭空使用“最新版本号”。缺少真实模型密钥时使用明确标识的测试适配器验证结构，真实调用的验收标为待验证，不能把 mock 当作上线功能。

每阶段更新 docs/PROGRESS.md，记录完成项、实际命令和结果、未完成项与下一步。最终提供启动命令、插件加载方法、截图、测试结果与已知限制。优先交付 P0，再实施 P1；P1 未完成不能宣称完整语义搜索已交付。

## 1. 产品定义与范围

名称：收好 Keeply。标语：随手收好，需要时找到。

定位：用户在浏览器插件配置自己的模型 API Key，把网页、链接、文本和文件收进来；系统异步生成摘要、标签和集合归属；网页统一浏览、管理和检索。

默认单用户、自部署、本地开发优先，但服务接口必须鉴权。后续可部署到个人服务器；本版不提供公开注册或多租户。数据同步到用户自己的 Keeply 后端，不是承诺所有内容只留浏览器本地。

硬约束：

- 模型 API Key 只存插件本机，不上传 Keeply 后端、不进入网页端、不使用 chrome.storage.sync。
- 模型调用由扩展可信上下文发起。内容会发送给用户所配置的模型服务，设置页应清楚说明。
- 浏览器退出后 AI 整理暂停；重启恢复。网页在插件离线时仍能浏览和关键词搜索。
- 收藏先持久化，再异步整理；模型失败不能导致原收藏丢失。
- 用户手动修改的标题、标签、集合不得被自动任务覆盖。
- 一条资料可属于多个集合。收件箱是“尚无已确认集合”的视图，不是强制文件夹。
- 来源内容、AI 结果、用户笔记分开存储；不能把摘要冒充原文。

### P0：首先交付的可用版本

1. Chrome MV3 扩展：当前页收藏、右键选文/链接收藏、侧边栏粘贴文本/链接和拖入文件。
2. 网页：同样支持粘贴和文件拖入；列表、详情、集合、标签、搜索、编辑、回收站、导出。
3. 文件：UTF-8 TXT、Markdown、可提取文字的 PDF。默认单文件上限 20MB；扫描 PDF 提示“需要 OCR，当前未支持”。DOCX 留到 P1。
4. 模型：自定义 OpenAI 兼容 Chat Completions 接口，手动填写模型 ID，测试连接。
5. AI 整理：中文摘要、3—6 个标签、已有集合建议；低置信度不强制归类。
6. 关键词检索：覆盖标题、原文、摘要、标签、备注，支持中文及类型/集合/日期过滤。
7. 可恢复队列、错误原因、手动重试、URL/内容去重。

### P1：核心体验增强

独立 Embedding 配置、语义混合搜索、DOCX、浏览器书签 HTML 导入。P1 语义搜索有明确的在线条件，见第 8 节。

### 暂不做

视频/音频理解、OCR、跨平台账号同步、付费墙绕过、复杂网页自动登录、知识图谱、公开社区、订阅收费、MCP、跨资料生成式问答。这些不阻塞第一版。

## 2. 默认技术架构

采用 TypeScript monorepo，pnpm workspace：

| 模块 | 默认选择 | 职责 |
| --- | --- | --- |
| apps/web | React + Vite + Tailwind | 管理页面、上传、关键词搜索 |
| apps/extension | WXT + React，Manifest V3 | 采集、配置、模型调用、任务执行 |
| apps/api | Fastify + Zod | 鉴权、收藏 CRUD、文件处理、任务租约、检索 |
| packages/contracts | Zod schema + TypeScript 类型 | 接口与模型输出契约 |
| packages/core | 纯函数模块 | URL 规范化、分类合并、分块、去重 |
| packages/model-client | fetch 适配器 | Chat/Embedding 调用、错误归一化 |
| 数据库 | PostgreSQL + SQL migrations | 收藏、集合、任务；P1 使用 pgvector |
| 文件 | 后端 data/uploads 持久卷 | 原始文件、净化后的正文快照 |

前端与 API 在部署时同源；开发用 Vite proxy。生产构建后可由 API 服务静态网页，降低部署复杂度。提供 Docker Compose 和本地开发两种启动方式，数据库和上传目录均挂载持久卷。

不引入 Redis、Kafka、独立向量数据库或微服务。后台文件解析采用数据库任务队列和独立 worker 进程，避免阻塞 API 事件循环。AI 队列由扩展领取执行；后端 worker 不持有模型 Key。

建议目录：apps/{web,extension,api}、packages/{contracts,core,model-client}、migrations、tests/fixtures、docs、docker-compose.yml、.env.example。

## 3. 核心数据模型

所有主键用 UUID，所有时间存 UTC，UI 按用户时区显示。正文较大时单独表存储。

| 表 | 关键字段 |
| --- | --- |
| items | id, type, source_url, canonical_url, source_title, display_title, user_note, content_hash, capture_status, content_version, user_revision, created_at, updated_at, deleted_at |
| item_contents | item_id, extracted_text, sanitized_snapshot_path, extraction_method, extraction_warning |
| assets | id, item_id, original_name, storage_key, mime, size, sha256 |
| collections | id, name, description, normalized_name |
| item_collections | item_id, collection_id, origin(auto/manual), locked |
| tags / item_tags | normalized_name；关联记录 origin、locked |
| ai_results | item_id, content_version, model_profile_fingerprint, prompt_version, generated_title, summary, suggested_tags, suggested_collections, partial, created_at |
| jobs | id, item_id, kind, content_version, status, step, attempts, next_run_at, lease_owner, lease_token, lease_expires_at, result_json, last_error_code |
| client_devices | id, name, token_hash, last_seen_at, revoked_at |
| export_jobs | id, status, output_path, expires_at |
| chunks（P1） | id, item_id, content_version, ordinal, text, embedding_profile_id, embedding |

模型设置仅存在扩展：profile_id、base_url、chat_model、api_key、timeout、output_limit、可选 embedding 配置。后端只可接收无密钥的能力摘要、模型指纹与在线状态。

标题呈现优先级：用户手动标题 > 有效来源标题 > AI 建议标题 > URL/文件名。AI 不覆盖原始标题。修改内容增加 content_version；用户编辑增加 user_revision。删除时取消任务，旧任务返回不得复活内容。

## 4. 采集与解析

### 4.1 当前网页

用户主动点击收藏后，通过 activeTab + scripting 注入提取器，在页面 DOM 副本上使用 Readability 类正文提取。保存 title、url、选文、正文和采集时间。只读取当前授权页面，不扫描浏览历史或后台标签。

模型 Key 不进入 content script。content script 仅返回受 schema 校验的采集结果。不得采集表单输入值、密码或隐藏字段；用户自己的登录页面也需主动选择收藏，不能自动抓取。

正文快照采用净化 Markdown/文本，不保存可执行 HTML。默认不批量抓取图片；无封面用站点名和类型图标。

### 4.2 粘贴/拖入链接

只接受 http/https 链接，立即保存元数据。后端尝试抓取公开 HTML 并解析正文；登录页面、动态渲染失败时状态为 link_only，提示“打开网页后用插件补全正文”。不能只靠标题生成看似读过全文的摘要。

抓取器需验证 DNS 解析后的地址及每一次重定向，拒绝 loopback、私网、链路本地和云元数据地址，限制重定向次数、响应大小和超时。考虑 DNS rebinding，使用验证过的解析地址建立连接或等价安全网络层。这个限制用于网页抓取，不妨碍用户显式配置 localhost 模型接口。

不要删除全部 URL query 参数；仅去除明确追踪参数，保留影响内容的查询参数。PDF 链接走文件限制和解析路径。

### 4.3 文本与文件

粘贴为单个 URL 时按链接处理，否则按文本处理；多链接批量导入后续增强，P0 不自动误拆普通文本。

文件先验证扩展名、真实 MIME 和大小，再写入随机 storage_key；禁止用用户文件名作为磁盘路径。原文件可下载；提取正文单独保存。加密 PDF、扫描 PDF、损坏文档分别报告原因。空正文不调用模型编摘要。PDF 可使用 PDF.js 的服务端文本提取能力，实施时验证所选版本兼容性。

### 4.4 去重与离线

同一规范化 URL 提示“已收藏”，提供查看/补全/刷新，不直接覆盖旧内容。不同 URL 但正文 hash 相同仅提示相似，不自动合并。重复点击用 client_request_id 幂等处理。

后端不可用时，扩展用 IndexedDB 持久化待同步采集包；明确提示“已暂存本机，待同步”。不可把它显示成已同步成功。配额不足需保留输入并提示失败。web 端网络失败同样不能假报成功。

## 5. 自动整理与调用契约

Base URL 约定是 API 根路径，例如 https://host.example/v1；客户端追加 /chat/completions，避免重复 /v1。不要求服务支持 /models。P0 接受标准兼容接口；不把任何厂商名字等同于兼容承诺。

测试连接发最小真实请求；页面提示可能产生少量用量。不输出或记录 Authorization。支持模型不接受 temperature、response_format 等可选参数时的显式兼容降级；不盲目重复付费调用。

输入：用户收藏备注、正文、已有集合 id/name/description、标签词表。正文作为不可信资料，不允许其中的“指令”改变系统任务或索取密钥；模型无工具权限。

期望输出 JSON：

```json
{
  "title": "建议标题",
  "summary": "80—150字中文摘要",
  "tags": ["浏览器插件", "信息管理", "产品设计"],
  "collectionSuggestions": [{"id": "已有集合UUID", "confidence": 0.91, "reason": "与集合用途匹配"}],
  "newCollectionSuggestion": null,
  "language": "zh",
  "partial": false
}
```

用 Zod 验证长度、字段和已有集合 ID，禁止模型伪造集合。无有效匹配时保持收件箱。模型建议新集合只显示为建议，用户确认才创建。confidence 只是启发式信号，不是校准概率；初始阈值 0.85，高于阈值且未被人工锁定时可自动归类，用户可关闭。阈值后续用实际样本调校。

标签做 trim、Unicode 规范化和大小写归一；优先复用已有标签。自动整理结果写 ai_results，再按人工锁定规则合并。提交结果必须带 content_version 和读取时的 user_revision；正文版本不一致作废，人工修订变化则保留 AI 建议但不覆写人工字段。

长文先分块摘要，再聚合；每块处理结果持久化，失败只重跑当前块。可配置上下文预算，留出输出空间。超出单篇预算时保留全文用于搜索，对 AI 处理范围标 partial 并在 UI 说明，不静默截断。不要宣称 token 估算对所有模型精确。

## 6. 队列、生命周期与恢复

采集状态和 AI 状态独立显示：例如“正文已保存 / 等待插件整理”。不要用一个 status 混合全部流程。

AI job 状态：queued → leased → running → succeeded；失败转 retry_wait 或 failed；删除内容转 cancelled。等待 Key 或缺权限显示 blocked，不能无限消耗重试次数。

后端原子领取（事务、行锁或等价方式），颁发每次唯一 lease_token，有效期初始 120 秒；运行时按能力续约。默认并发 1。完成/失败接口验证 token、租约、item 未删除和 content_version。过期租约重新排队，旧执行器返回 409，不再覆盖结果。短任务每步独立领取或检查点保存。

Chrome service worker 可被终止，不能依赖全局变量、setInterval 或 popup 存活。用 chrome.alarms、启动事件和用户事件唤醒调度，任务数据持久化；启动时确保 alarm 存在。后台 AI 子请求采取保守超时，首版建议不超过 20—25 秒并限制输出；超时允许有限重试，不保证慢模型都能后台完成。

为慢模型提供“打开整理任务页”模式：扩展独立页面成为执行器，可处理较长请求；与后台共用原子租约，避免双重执行。关闭任务页会中断当前请求，租约过期后恢复。不要滥用 offscreen 页面或保活循环伪装永久后台。

429/5xx/网络超时有限指数退避并加抖动，遵守有上限的 Retry-After；例如最多 3 次自动重试。401/403、模型不存在、格式错误需分类提示；JSON 格式可修复一次，仍失败显示原因。保持收藏可读。

模型调用是 at-least-once：如果提供商已计费但扩展在落库前崩溃，无法保证零重复计费。可避免重复落库，但不得宣传 exactly-once 调用。存模型 usage（若返回），不凭估算显示精确费用。

## 7. 网页与插件界面

视觉：中文优先、简洁温暖，浅色背景加绿色强调色；不照搬参考产品资产。Logo 首版用书签或收纳盒几何图形。桌面先行，网页适配窄屏。

网页：左侧全部收藏/收件箱/集合/回收站；顶部搜索和新增；主区卡片/列表；点击后右侧详情抽屉。详情包含原文、摘要、个人备注、标签、集合、来源、处理状态与重试。支持批量归类和软删除。用户可恢复；永久删除需确认，清理原文件、正文、向量和任务。

插件 popup：收藏当前页、最近 5 条、打开侧边栏/管理端/设置。side panel：粘贴框、文件拖拽、处理队列。options：后端地址和配对、模型配置与连接测试、自动分类开关、预算、备份/删除配置。密钥掩码显示，默认不导出。

空状态、loading、错误状态、未配置模型、插件离线、仅链接、部分摘要都必须有明确文案。不要以假数据冒充用户收藏；演示数据只能由显式命令导入。

## 8. 检索：先可靠，再语义化

P0 不依赖 AI 查询：PostgreSQL pg_trgm 对中文标题/正文/摘要/备注等做检索，短词使用参数化 substring 查询兜底；组合标题和标签优先的排序。不要直接套英文 FTS 分词器并声称已支持中文。返回匹配字段、片段、来源和日期。设计中英文测试语料验证实际结果。

P1 为 Embedding 单独设置 base_url、key、model 和维度；不得默认 Chat 模型支持 embeddings。Key 仍仅插件保存。文本分块初始约 800—1200 中文字符、重叠约 150 字，后续评估调整。

模型指纹包括供应商地址、模型、维度与分块版本；文档和查询必须使用相同配置。换模型建立新索引并切换，不能混用维度或不同模型空间。服务端检查向量长度、有限数值和模型指纹。

网页语义查询流程：先立即返回关键词结果，同时创建短时查询任务；在线扩展领取、生成 query embedding、回传向量；服务端进行向量检索，与关键词结果做 RRF 融合。前端轮询最多约 8 秒，超时保留关键词结果并提示“语义搜索需配置模型的插件在线”。查询任务过期即取消，不在第二天继续生成旧查询向量。

此桥接不把 Key 给网页，也不开放网页任意调用模型的通道。插件只领取固定 schema 的任务，后端只接受已配对设备。其他设备没有在线插件时，本版只保证关键词搜索。未来若要求随时全端语义搜索，需另行设计服务端 Embedding 或浏览器本地 Embedding，不静默改变 BYOK 边界。

## 9. API 契约与鉴权

统一 /api/v1，错误结构包含 code、message、retryable、request_id。输入全部 schema 校验，列表 cursor 分页，默认 30、最大 100。

| 接口 | 用途 |
| --- | --- |
| POST /auth/login, POST /auth/logout | 单用户登录、登出；密码仅存 hash，限速 |
| POST /devices/pairing-codes | 网页登录后生成短效一次性配对码 |
| POST /devices/pair | 插件换取设备 token；后端只存 token hash |
| DELETE /devices/:id | 撤销设备 |
| POST /items | 保存 URL/文本/采集正文，带幂等 request_id |
| POST /items/upload | multipart 文件上传，返回已落盘状态 |
| GET /items, GET /items/:id | 列表/详情 |
| PATCH /items/:id | 乐观并发版本校验；人工编辑 |
| DELETE /items/:id, POST /items/:id/restore | 回收/恢复 |
| DELETE /items/:id/permanent | 永久删除 |
| CRUD /collections | 集合管理；重名校验 |
| POST /items/:id/reprocess | 新建整理任务；保留人工锁定 |
| POST /jobs/claim | 扩展原子领取任务 |
| POST /jobs/:id/heartbeat, /complete, /fail | 租约续期/完成/失败 |
| GET /search | P0 关键词搜索 |
| POST /search/semantic, GET /search/semantic/:id | P1 异步语义请求及结果 |
| POST /exports, GET /exports/:id | 导出 JSON/Markdown ZIP，不包含密钥 |

web 使用 HttpOnly session cookie；生产 Secure、合适的 SameSite，所有写请求校验 Origin/CSRF。扩展使用可撤销 device token，作用域限自身数据和任务。CORS 仅允许配置的前端和扩展来源；不能把 CORS 当鉴权。配对码短效、单次、限速。P0 单用户，不用硬编码公开管理员 token。

扩展权限优先 activeTab、scripting、storage、alarms、contextMenus、sidePanel；自定义模型与后端域名用 optional_host_permissions 按用户配置请求授权。不默认获取所有网站永久权限。Key 放 chrome.storage.local 并设置 TRUSTED_CONTEXTS；这并非操作系统加密保险箱，不做“加密存储”虚假承诺。仅可信扩展代码可读。模型请求拒绝跨源重定向，避免密钥转发；远端必须 HTTPS，本机开发接口可显式允许 HTTP loopback。

UI 对模型输出和抓取内容做净化/转义；导入 HTML、Markdown、文件名不能触发脚本。日志不含完整正文、密钥、Authorization 或配对码；私有下载与快照必须鉴权。

## 10. 开发阶段与完成标准

### M0：骨架与真实持久化

建立 monorepo、数据库迁移、鉴权、配对、契约、开发环境；实现网页新增一条文本，刷新和重启后仍存在。提供 docker compose、env 示例及健康检查。不要先搭大面积空壳页面。

### M1：端到端采集

实现插件加载、配对、当前页/选文/链接采集、侧边栏拖文件、网页上传、解析 worker、来源预览、离线暂存和去重。验收：插件收藏一篇中文文章，网页读到正文；TXT/MD/PDF 成功或给出真实原因。

### M2：BYOK 自动整理

实现 profile、权限申请、测试连接、model-client、输出校验、租约队列、重试、慢模型任务页、摘要标签与分类合并。验收：真实模型产生结果；杀死 service worker/重启浏览器后可恢复；编辑或删除后旧任务不能覆盖。

### M3：管理检索与 P0 发布包

集合编辑、卡片/列表、中文关键词搜索、筛选、详情、批量操作、回收站、导出；清理敏感日志。提供生产构建、扩展加载目录或 ZIP、README、测试报告和备份恢复说明。

M0—M3 完成即 P0。每阶段完成继续推进下一阶段，不等用户逐项确认。

### M4：P1 语义搜索

实现 Embedding 独立配置、分块/向量索引、查询桥接、模型更换重建、RRF、离线降级。再加 DOCX 和书签导入，分别验收。若资源不足，先交付已验证 P0 并清楚记录 P1 尚未完成，不伪装语义搜索。

## 11. 必须验证的场景

1. 未配置 Key 也可收藏并关键词搜索，AI 等待配置。
2. 中文文章、英文文章、纯文本、MD、文本 PDF 的内容可查看和搜索。
3. 扫描 PDF/加密 PDF/损坏文件/超限上传显示不同错误，不生成假摘要。
4. 401、429、5xx、慢响应、断网分别触发预期行为，收藏不丢。
5. 关闭 popup 不取消整个收藏；终止 worker、关闭浏览器重开后任务可恢复。
6. 两个扩展执行器争抢任务，仅一个租约有效；旧租约完成请求被拒绝。
7. 人工编辑分类期间 AI 返回，人工修改保留；删除期间任务完成不能复活数据。
8. 同 URL 连点和网络重试不重复新增；正文刷新不悄悄覆盖个人备注。
9. 恶意网页指令、脚本、伪集合 ID、异常 JSON 被当数据处理，不执行。
10. 内网 URL、重定向到 localhost、非法文件路径、未认证下载被拒绝。
11. 从网络请求与后端数据库确认模型 Key 不进入 Keeply 服务；配置导出不含 Key。
12. 重启服务后文件和数据仍在；导出包含来源、正文/摘要、备注、标签与集合关系。
13. P1：断开插件自动退回关键词检索；混合模型向量被拒绝；换模型不破坏旧索引。

测试方法：纯函数/契约用 Vitest；接口/租约用真实测试 PostgreSQL 的集成测试；网页用 Playwright。插件生命周期、跨域权限和真实模型必须在 Chrome 加载扩展做人工验证，不能只凭 Node 单测通过。至少准备 20 条中英混合资料，建立 10 个精确查询和 10 个模糊查询的人工相关性样本，记录结果，不杜撰准确率。

## 12. 交付清单与运行约定

- 完整源代码、迁移、锁文件、.env.example，不包含真实密钥。
- README：环境要求、初始化管理员、启动后端/网页、构建与加载插件、配对、配置模型、常见错误。
- pnpm dev、pnpm build、pnpm test、pnpm lint、pnpm typecheck 与 pnpm db:migrate 等统一脚本；具体实现可按工具调整，README 必须与实际一致。
- Docker Compose 可启动持久化服务；仅开发环境可绑定 loopback 简化 HTTPS，外网部署必须写明 HTTPS 和鉴权配置。
- docs/ARCHITECTURE.md、docs/API.md、docs/PROGRESS.md、docs/TEST-REPORT.md。
- 展示真实成功路径和失败状态的截图；测试环境数据明确标注。
- 备份步骤覆盖 PostgreSQL 和 data/uploads；至少验证一次恢复。JSON/Markdown 导出是可携带资料，不冒充完整系统备份。

## 13. 官方实现参考

下列是已核查的 Chrome 关键约束；架构与阈值是本任务书的设计选择。执行 Agent 安装其他依赖时仍需核对其官方 API，不凭本文推断版本兼容性。

- Service worker 生命周期与中断恢复：https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle
- storage 与可信上下文访问范围：https://developer.chrome.com/docs/extensions/reference/api/storage
- 扩展跨域请求和 host permissions：https://developer.chrome.com/docs/extensions/develop/concepts/network-requests

项目成功标准：用户真正能够“配置自己的模型 → 收藏网页或文件 → 自动整理 → 在网页找回”，并且失败可解释、数据不丢、密钥不外泄到 Keeply 后端。
