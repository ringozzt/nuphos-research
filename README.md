# Nuphos 深度技术调研

> **一句话**：Nuphos 是 Zeabur Inc. 的产品——把 Claude Code、Codex 从「一个人在一台笔记本的一个终端里」搬进团队共享的工作区：会话是多人可加入的「房间」，不绑定某台机器，可在本地 / 远程 / 托管云 runtime 间迁移，团队共享技能、工具连接与记忆。它与 paperclip 的区别是品类性的：paperclip 是坐在 agent 之上的「治理控制面」，Nuphos 是会话本身居住、被多人实时共同驾驶的「执行面」。
>
> - 原项目：[nuphos/nuphos](https://github.com/nuphos/nuphos)（Apache-2.0）· 官网 [nuphos.ai](https://nuphos.ai/)
> - 调研时间：2026-10-11 · 标注说明：= 仓库文件 / 官方站点一手来源；= 媒体、目录站等二手来源
> - 本报告为独立第三方调研，与原项目无关
>
> 交互版本：
> - 📄 [report.html](./report.html)——完整交互调研页面（可直接浏览器打开）
> - 🗺️ [mindmap.html](./mindmap.html)——可交互折叠思维导图

## TL;DR（5 条硬发现）

1. **许可比标签干净**：逐字核对为真 Apache-2.0（根 LICENSE 标准全文、无附加条款）；GitHub 的 "Other (NOASSERTION)" 只是许可证探测失败（推断源于 runtime 镜像捆绑 Claude Code 等非开源成分）。闭源只在仓库外：Android 客户端、托管运维层、商标（名字/logo）。
2. **产品不年轻、仓库才年轻**：托管产品 2026 年夏已以「AI 原生 DevOps 工作区」面世（前身代号 Atlas，代码、提示词、环境变量中至今残留），2026-10-01 只是开源仓库创建与重新定位之日。主体是 Zeabur Inc.（NOTICE 署名），PitchBook 称其累计融资约 $2.3M。
3. **真正的赌注是把 ACP 网络化**：后端经 OpenAB 网关以 ACP-over-WebSocket 驱动远程容器 runtime——把 ACP 从「编辑器 ↔ 本地 stdio 子进程」改造成团队 runtime 总线。已支持 5 种 agent：Claude Code、Codex、Grok Build、Antigravity、OpenCode。
4. **与 paperclip 是执行面 vs 治理面之分**，不是同类竞品：Nuphos 无组织图/预算/心跳，paperclip 不托管会话本身。四点差异化（会话即房间实时共驾、会话与机器解耦三态迁移、工作区即运维台直操客户云账号、跨厂商 + 团队技能/记忆共享）同时具备者，在已核竞品中没有第二个。
5. **最大风险在权限与条款，不在代码**：一个团队密钥 + 一份个人订阅，后面站着能操作你 AWS/GCP 全账号的 agent；共享会话即共享权限；而「多人共驾一份个人订阅」的合规定性，连 Anthropic 官方文本都未直接回答。

## 1. 项目与公司

- **产品定义**：面向团队的 agent 工作区——在多人、多机器、多工具间运行、管理、协作编程 agent。核心理念：编程 agent 默认是单人模式，Nuphos 让团队来运行它们。官网自称 SOC 2 Certified，有 Contact sales，无定价入口。
- **公司主体**：NOTICE 署名 Copyright 2026 Zeabur Inc.；TRADEMARK.md 称名称与 logo 为 Zeabur Inc. 商标。创始人为 Zeabur 创始人兼 CEO Yuanlin Lin（林沅霖，GitHub yuaanlin——其主页 bio 自述 Zeabur Founder & CEO、浙江大学计算机毕业，Product Hunt 本人留言自称 founder of Zeabur and now building Nuphos；一手核实）。PitchBook 称 Zeabur 2022 年成立、累计融资约 $2.3M（500 Global、Baidu Ventures、MiraclePlus 等），未见 Nuphos 单独融资——应理解为 Zeabur 的产品线。
- **时间线**：前身仓库 zeabur/atlas 自述为 Nuphos 托管产品源码，产品原名 Atlas（后端提示词文件至今叫 atlas-agent-system.md）；EveryDev 2026-08 收录（DevOps 定位）；Product Hunt 2026-08 上线；开源仓库 2026-10-01 创建，Desktop v0.73.1 自称公共仓库的第一个完整 Desktop 发布。
- **托管 vs 开源**：托管版（Nuphos Cloud）提供多团队单实例、托管 runtime 置备、备份、运维工具、Google 登录；开源仓库提供 backend / desktop / runtime / iOS / CLI 全部主要代码，自托管为单团队部署。

## 2. 架构

| 路径 | 内容 |
|---|---|
| apps/backend | Bun/Hono API：账号、团队、会话、runtime 注册表、集成 |
| apps/desktop | Electron/Vite/React 客户端，即工作区本体 |
| apps/runtime | Claude Code / Codex 容器镜像、适配器、独立发版 |
| apps/ios | SwiftUI 客户端，离开桌面后接续会话 |
| apps/cli | README 表未列：`nuphos agent` 可把本机变成团队的本地 agent |

**数据栈**：MongoDB（用户/团队/JWT，单节点副本集）+ S3 兼容存储（自托管用 RustFS）；托管版另有 Redis（无 Redis 时 cron、后台队列、审计封存被禁用）。后端自带一个内置 agent（atlas-agent-system.md），经 SKILL.md + 脚本操作团队连接的系统；后端自身的标题/洞察类模型调用走 AWS Bedrock。

**runtime：一个 turn 如何执行（技术核心）**：
- runtime 是长寿命容器，内含恰一个 agent；镜像三层：(1) **OpenAB 网关**（外部开源项目，以 submodule 引入）拥有容器，经网络接受 ACP、监督 agent、在 agent 重启后保持会话存活，端点 `ws(s)://<host>:8080/acp`；(2) agent CLI + ACP 适配器——Claude 用 claude-agent-acp、Codex 用 codex-acp，**适配器构建时打补丁并以 SHA-256 钉住上游产物**，上游升级会使构建失败直至人工复核；(3) Nuphos 工具集层（git/gh/kubectl 常驻，aws/gcloud 等为首次用时下载钉版的 shim）。
- Grok Build / Antigravity / OpenCode 原生说 ACP，只加 shim；Grok/Antigravity 为专有软件不在镜像内，首次使用从厂商下载。
- **会话状态在后端（MongoDB），runtime 是可替换的执行器**——故会话可在本地机 / 自有远程 / 托管云间迁移并继续，移动后模型选择器切换为新 agent 的模型列表；关盖续跑、iOS 接续都建立在此解耦上。
- **认证**：runtime 内由用户本人完成 `claude auth login` / Codex 设备码登录，凭据只留在 runtime 卷（~/.claude / CODEX_HOME），Nuphos 不收集；用用户自己的编程套餐订阅，**无额外模型费**。镜像内无凭据无提示词，系统提示词、技能与每回合凭据由后端经 ACP 运行时下发。/acp 端点拒绝无团队密钥的请求（401），bindings 只存密钥的 SHA-256 摘要。
- **本地 agent 的关键语义**：只有机主能在本机发起会话；但会话一旦被移上这台机器，团队任何人都能续聊——且**他们的消息会以机主身份在这台机器上执行命令**（apps/cli/README.md 明示）。

**Desktop / iOS**：Desktop 每 6 小时检查更新，macOS 签名+公证，Windows/Linux 未签名；分屏快捷键、内置浏览器标签。iOS 已上架 App Store，独立版本线。Android 客户端存在但尚未开源。

## 3. 核心产品理念

- **(a) 会话即房间**：像 Notion 页面一样邀请——多人对同一 agent 说话、看同一批工具调用、共同转向。会话级权限已产品化（Desktop v0.77.0）：仅受邀 / 全队可看 / 全队可看可回；受邀者分 Can reply / Can manage 两级，仅所有者改权限，Can manage 者可迁移会话与换凭据。
- **(b) 会话比机器长寿**：三种 runtime 形态（本地 / 自有远程 / 托管）互迁，关盖续跑，手机接续。
- **(c) 团队共享技能与工具**：授权一次，全队每个 agent 同步获得，不再人人往自己机器粘 token；runtime 侧 nuphos-sync-skills 原子替换。
- **(d) 团队记忆**：跨队友、跨 agent、跨会话内置，无需另装 MCP 服务器；记忆以团队为作用域、存于后端。**其数据表示（自由文本 / 结构化 / 向量检索）公开文档未披露，未能核实。**
- **(e) 触发器**：cron 定时或 webhook 事件可自主启动回合（自托管 Compose 栈无 Redis 时 cron 被禁用）。
- **(f) Slack / Discord**：一步把同一 agent 放进团队频道，后续版本做了话题与会话互绑、发言者归属显示。
- **(g) 工作区即运维台**：Kubernetes、Grafana、GitHub、Linear 为一等 GUI，另有 GitLab、AWS（含 ECS）、GCP、Cloudflare、Linode、MongoDB/SQL。**面板不是截图——agent 操作面板背后的账号本身**：AWS 跨账号 assume-role、GCP 服务账号 impersonation（BYOC，信任主体是客户自己部署的身份）。官方设计原则：只读起步、受保护变更先出计划等人工批准、作用域会话、保留审计。

## 4. 许可（逐字核对）

- 根 LICENSE 为标准 Apache-2.0 全文，无附加条款、无例外段落。NOTICE 仅 323 字节：Zeabur 署名 + 商标声明。TRADEMARK.md 明言「代码 Apache-2.0、仅名称/logo 保留」——描述性使用免许可，分发修改版作自有产品须改名。CONTRIBUTING 亦称全仓 Apache-2.0。**未发现任何 BSL / ELv2 / Commons Clause 等 source-available 第一方条款。**
- GitHub "Other (NOASSERTION)" 的合理解释（推断，官方未说明）：许可证探测因多许可证成分无法自动归类——runtime 镜像捆绑非开源的 Claude Code / Agent SDK（按 Anthropic 商业条款分发），Grok/Antigravity 运行时从厂商下载。
- **真正不开的清单**：Android 客户端（尚未开源）；托管 runtime 置备 / 备份 / 运维工具；多团队单实例；Google 登录（仅 Nuphos Cloud）；厂商 agent 本体；商标。
- 结论：准确的定性是「**Apache-2.0 开源 + 商标保留 + 托管运维层闭源**」的标准 open-core。

## 5. 自托管

三条路径：Zeabur 一键按钮（数据栈与 runtime 分项目部署）、Railway（CLI 指南为主）、Docker Compose（MongoDB 8 + RustFS + backend + 一个 runtime 一条命令起全套）。

| 能力 | 自托管 | 托管 Cloud |
|---|---|---|
| backend/desktop/iOS/runtime 代码 | ✔ Apache-2.0 | ✔ |
| 团队数 / 部署 | 1（硬约束） | 多 |
| 托管 runtime 置备、备份、运维 | ✘ | ✔ |
| Google 登录 | ✘（邮箱 OTP） | ✔ |
| cron 触发（Compose 栈） | ✘（无 Redis） | ✔ |

Compose 栈自列限制：无 Bedrock 则后端标题/洞察不可用；NODE_ENV=development 勿公开暴露；managed agent 跑不了（需 K8s 置备器）；runtime 镜像目前仅 amd64。

## 6. 热度与生态

- 仓库数据（2026-10-11）：创建 2026-10-01；**45 stars / 6 forks / 152 commits / 21 releases / 25 tags**。10 天约 2 release/天，分组件版本线：Desktop v0.77.0、Backend v0.86.0、Runtime v0.1.19、iOS v0.8.0、CLI v0.3.0，**全线 <1.0**，Desktop 发布说明附组件兼容矩阵。
- 协作形态：Issues 开放数 0；PR 13 开放 / 169 已关闭，多为 nuphos[bot] 与 dependabot——开发明显高度 agent 驱动，**外部社区贡献尚无证据**。未检索到 Hacker News 实质讨论。
- 定价：nuphos.ai/pricing 实时打开为 **404**，官网仅 Download 与 Contact sales。EveryDev 保存的旧价（Free / Dev $29 / Team $99 / Business $299 / Scale $999）属旧托管用量模式，与现行 BYO 订阅定位冲突，**不可作现行价引用**。

## 7. 竞争格局

| 产品 | 形态 | 与 Nuphos 的关键差异 |
|---|---|---|
| paperclip | 开源（MIT）agent「公司」控制面：组织图、预算、审批、心跳；只编排不执行 | Nuphos 是会话本身的实时共驾 + 跨机迁移 + 生产 GUI。量级也不同：paperclip 第三方统计约 97k stars |
| Multica | 开源托管 agent 平台：issue 分给 agent 自行认领汇报 | 团队论最接近，但中心是看板异步协作，非实时房间 |
| Conductor | 免费 macOS 应用，并行多 agent + worktree（闭源） | 单人单机，无团队会话、无云端续跑 |
| Vibe Kanban | 开源看板编排 | 公司 Bloop 2026-04 关停，项目 sunsetting 中 |
| Sculptor | MIT 桌面应用，容器 + IDE 配对 | 本地单人导向，无团队会话 |
| Terragon（已关停） | 云端后台 agent 编排，BYO 订阅 | 形态最像的历史玩家，2026-01 关停——该品类商业化难度的直接先例 |
| 厂商云功能 | Claude Code 云会话 2026-09 转正；Codex 云任务 | 单厂商围墙：不跨厂商共享技能/记忆/房间 |

**ACP 生态**：ACP 由 Zed 于 2025-08 发起、JetBrains 共维，被喻为 agent 界的 LSP，注册表约 50 个 agent。多数采用者是「编辑器 ↔ 本地 stdio 子进程」；**Nuphos 把 ACP 经 OpenAB 变成网络协议（WebSocket），用作远程 runtime 与后端之间的团队化会话总线**——这是它真正的协议赌注。

## 8. 批评与风险

- **年轻与 churn**：开源仅 10 天、外部贡献无证据；全线 <1.0、约 2 release/天、组件间需兼容矩阵；Atlas 代号在代码/提示词/环境变量中大量残留，显示改名仓促。但注意：产品自 2026 年夏已运营，勿把「仓库 10 天」误读为「产品 10 天」。
- **安全攻击面（最实质）**：
- runtime README 自警：**团队密钥是挡在持有你工作区云凭据的 agent 前面的唯一东西**；共享会话即共享权限，能 reply 就能以会话作用域凭据驾驶 agent，Can manage 还能迁移会话、换凭据。
- 本地 agent：队友消息以**机主身份**在机主机器上执行命令。
- agent 触达面 = 所连云账号 CLI 的全部能力；Slack/Discord/Linear 等外部内容经 prompt injection 可达生产系统；团队共享记忆带来跨会话记忆投毒面。缓解项（只读起步、计划审批、审计）在官网博客多为原则性表述，强制执行细节披露有限；自托管 Compose 栈连审计封存都因无 Redis 被禁用。
- **厂商条款灰色地带**：Anthropic 官方文本一面禁止第三方以订阅 OAuth 在自家应用中中介请求，一面允许「在产品中运行 Claude Code」（若二进制不改、每用户自证、不代付转售）。Nuphos 形态贴后者设计，但托管形态需商业条款协议、**多人共驾一人订阅会话的定性未明**。适配器补丁钉 SHA，上游一升级即断构建，是持续维护负担。

## 思维导图

```mermaid
mindmap
root((Nuphos<br/>团队 agent 工作区))
是什么
Zeabur Inc. 的产品
会话 = 多人房间
会话比机器长寿
团队共享技能/工具/记忆
时间线
2026 夏 托管产品面世(Atlas)
Product Hunt 2026-08
2026-10-01 开源仓库 + 重新定位
45 stars / 21 releases
架构
backend: Bun/Hono
MongoDB + S3/RustFS
内置 agent(atlas 提示词)
desktop: Electron/React
runtime(核心)
长寿命容器 = 一个 agent
OpenAB 网关
ACP-over-WebSocket
适配器补丁钉 SHA
iOS: SwiftUI 接续
cli: nuphos agent 本地机
ACP 赌注
Zed 发起的编辑器协议
多数人用本地 stdio
Nuphos 改造成网络 runtime 总线
5 种 agent
Claude Code / Codex
Grok Build / Antigravity / OpenCode
认证
用户自有订阅登录 runtime
凭据只留在 runtime 卷
无额外模型费
产品理念
会话即房间(权限两级)
三态 runtime 迁移
技能授权一次全队同步
团队记忆(表示未披露)
cron/webhook 触发器
Slack/Discord 入频道
工作区即运维台(BYOC 直操云)
许可
真 Apache-2.0 逐字核对
Other 标签 = 探测失败
闭源在仓库外
Android 客户端
托管运维层
商标
竞争
paperclip: 治理面非同类
Multica: 看板异步协作
Terragon 已关停(先例)
厂商云功能: 单厂商围墙
风险
共享会话 = 共享权限
团队密钥是唯一屏障
队友消息以机主身份执行
多人共驾一份订阅: 条款未明
全线 <1.0 高 churn
```

> 交互版：[mindmap.html](./mindmap.html)

## 来源

**Primary（repo / 官方文档 / 官网）**：
- https://github.com/nuphos/nuphos（README，2026-10-11 读取）
- https://github.com/nuphos/nuphos/blob/HEAD/LICENSE · /NOTICE · /TRADEMARK.md · /CONTRIBUTING.md（许可逐字核对）
- https://github.com/nuphos/nuphos/blob/HEAD/apps/runtime/README.md（turn 执行、凭据、安全警告）
- https://github.com/nuphos/nuphos/blob/HEAD/deploy/compose/README.md（自托管限制）
- https://github.com/nuphos/nuphos/releases（版本线与兼容矩阵）
- https://nuphos.ai/ · https://nuphos.ai/blog/best-kubernetes-gui（BYOC 安全原则）
- https://github.com/zeabur/atlas（前身仓库自述）

**Secondary（媒体 / 目录站 / 第三方）**：
- https://pitchbook.com/profiles/company/533492-02（Zeabur 融资，字段陈旧需谨慎）
- https://www.everydev.ai/tools/nuphos（2026-08 收录、旧定价存档）
- https://www.producthunt.com/products/nuphos
- https://agentclientprotocol.com（ACP 生态）
- https://code.claude.com/docs/en/legal-and-compliance（Anthropic 条款原文核对）

## 未核实 / 开放问题

- 团队记忆的数据表示（自由文本 / 结构化 / 向量）在公开文档中未披露。
- 托管产品现行定价：官网无价格页（/pricing 为 404），旧价不可引用。
- 创始人个人页为 yual.in（其 GitHub 主页所挂）；X/Discord 具体账号与社区规模、YouTube 覆盖：本次未核实。
- OpenAI 侧对「容器内跑 Codex + 多人共驾订阅」的条款口径：未找到对等文本。
- GitHub "Other (NOASSERTION)" 的确切成因：官方未说明，本报告的解释为基于文件证据的推断。
- 本调研底稿第 8 节尾部（订阅条款合规细节）原文截断，相关结论以一手条款文本核对为准。

## 调研方法

1. 通读仓库 README / LICENSE / NOTICE / TRADEMARK / apps 各子目录文档 → 2. 核对官网与博客 → 3. 追溯前身仓库（zeabur/atlas）与目录站存档确认时间线 → 4. 竞品逐一核对形态与存续状态 → 5. 许可与条款逐字核对，区分一手/二手并标注未核实项。
数字精确到日期；star 数为 2026-10-11 快照。
