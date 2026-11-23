# dsh superagent 可行性调研报告

> 调研日期:2026-11-23
> 调研对象:[deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype) 研究原型及其在本环境的实际部署
> 调研方式:阅读原型仓库文档与源码 + 在本部署上运行验证(本报告即由该系统的一条工单自动完成并通过 PR 交付)

## 1. 结论(TL;DR)

**可行,且已在本地得到实证。** dsh superagent 是一个打包为 DeepSeek Harness(DSH)bundle 的"主动式编码智能体":它监听 GitHub(可选飞书)事件,把被允许的人提出的工作请求变成工单(Ticket),由独立的 agent 会话完成实现、检查、提交 PR,并回到原对话交付。本次调研请求本身就是通过它受理并交付的——用户在飞书群里说"调研 dsh superagent 的可行性,把报告写进仓库",系统即开出工单,由 worker 会话完成本报告并以 PR 形式写入本仓库。这构成了对"可行性"问题最直接的回答。

同时它是明确的研究原型:README 自述"仅供参考的研究原型,非官方产品",需要运营者投入部署与看护成本,并以"尽量少占用人的注意力"为运营约束。

## 2. 它是什么

- **形态**:DSH profile 下的一个插件 bundle(本环境部署于 `~/.dsh/profiles/superagent`,与 DSH Web 面板同进程运行,面板地址 `127.0.0.1:3082`)。
- **职责**:对接一个 GitHub 仓库 + 一个飞书群,受理工作请求 → 开工单 → 派给 worker 执行 → 在原对话交付(确认、回复、PR、评审答复)。
- **设计哲学**(docs/000-philosophy.md):注意力经济学——每条消息、通知、提问、面板元素付出的注意力必须小于其给出的价值。agent 只在被明确请求或团队书面规则允许时行动;只在"有人必须行动或有人能用上结果"时说话。

## 3. 领域模型与架构(docs/001-domain.md、docs/007-architecture.md)

工作流为一条单向链:

```
Source 事件 → Trigger →(订阅扇出 / Eval 评估)→ Ticket(带不可变 Mandate 授权)
   → Assignment 绑定 Executor Session → Report 状态回传 → Channel 写回原对话
```

- **核心组件**:Engine(所有状态的唯一写者,每命令一个 SQLite 事务)、单一 SQLite 存储、持久副作用队列(Outbox,先落库后执行,指数退避)、GitHub 与飞书两个 Source 插件(新增 Source 有 11 步配方 docs/008-cookbook.md)、Web 面板(只回答"有什么需要我 / 系统健康吗 / 一件工作的历史")。
- **授权模型**(docs/002-authorization.md):两条准入路径——允许名单上的人直接 @ 触发 explicit 请求;其余消息按各 Source 的准入表 + 评估规则判定,默认 `observe` 模式只记录不执行。Mandate 记录 Policy 版本与授权范围,只有 Eval 创建、人的 stop、重校验失败三种操作能改变它。
- **安全边界**:prompt 文本改不了权限——消息、评论、引用文档永远只是决策输入;gate、Mandate、工具检查由代码强制。

## 4. 实际部署证据(本环境)

本部署完整走通了文档所述设计,以下为实地验证结果:

| 能力 | 证据 |
|---|---|
| 飞书受理 | 用户消息被转为 Trigger,工单自动创建并回执确认 |
| 独立执行 | 每工单一个 worker 会话 + 独立工作目录(`<stateDir>/work/`),本报告即在其中完成 |
| GitHub 交付 | worker 直接使用 `gh` CLI 操作目标仓库,开 PR 交付 |
| 双模型配置 | 通过 profile 配置接入 GLM 等自定义 LLM 提供方 |
| Web 面板 | 与 DSH Web 同端口,展示工单状态与需要人的事项 |
| 持久化 | 单文件 SQLite(`orchestrator.sqlite`),工单可跨会话恢复 |

## 5. 部署与运维成本(docs/006-deployment.md、SETUP.md)

**这是主要成本所在,文档自己承认"Setup has many steps"。** 需要:

- 独立 DSH profile + 专用 agent GitHub 账号(不能是 GitHub App,因为要读通知)+ 仅该账号登录的 `gh` wrapper;
- Node 22.19+/24、pnpm、git 2.32+、Python 3.8+、tmux、gh 2.48+;仅支持 github.com(不支持 GHES);
- 可选飞书自建应用 + bot 专属 `lark-cli`;
- 官方推荐的安装方式是让编码 agent 读 SETUP.md 陪同运营者逐步完成(一次一问、集中确认);
- 升级需等无 worker 在跑、备份 SQLite、重建重启;回滚依赖升级前备份。

## 6. 局限与风险

1. **原型成熟度**:README 明示可能含 bug、工程实践不一致,与团队论文中描述的内部版本不完全一致(部分设计未实现)。文档质量(编号文档、链接与索引有测试保证)远超典型原型。
2. **隔离不足**:`danger-full-access` 以运营者 OS 用户运行;wrapper 与目录只是"默认而非安全边界",worker 可按绝对路径执行程序、读取 home 目录。需要隔离时应使用专用 OS 账号。
3. **存储无界增长**:trigger/disposition/decision、已结束工单、outbox 回执均无清理机制(docs/007-architecture.md)。
4. **数据库前向不兼容**:新版迁移后的库旧版打不开,回滚必须依赖备份。
5. **通用模块仍耦合 GitHub/飞书**(如飞书字数限制硬编码在 Outbox),新增 Source 存在改造成本。
6. **网络环境依赖**:本环境实测 github.com 的 git smart-HTTP 端点被本地网络中间盒干扰(TLS 证书为 Sectigo 签发的 MITM 证书),git push/ls-remote 挂起;api.github.com 正常,只能改走 Git data API 交付。生产使用需为 agent 提供干净的 GitHub 网络路径。
7. **对人的判断仍有依赖**:默认需人手动合并 PR;auto-merge 仅在极窄条件下(改动 <200 行、审批覆盖 head commit)且绝不以 agent 账号启用。

## 7. 适用性建议

- **适合**:接入了明确仓库、请求以"具体代码改动"为主的团队;先以 `observe` 模式和小允许名单试运行;飞书群作为入口、GitHub PR 作为出口的组合已经闭环可用。
- **不适合现在就上生产**:需要强隔离(多租户/敏感代码)、需要 GHES、或无法接受数据库备份/升级运维的场景。
- **落地路径**:① 用专用 OS 账号与专用 GitHub 账号部署;② observe 模式观察一周准入判定;③ 逐步放开 `auto` 与仓库范围;④ 为 SQLite 建立定期备份;⑤ 为 agent 配置可直连 github.com 的网络出口。

## 8. 参考资料

- 原型仓库文档:`docs/000-philosophy.md` ~ `docs/008-cookbook.md`、`README.md`、`SETUP.md`(仓库 <https://github.com/deepseek-ai/dsh-superagent-prototype>)
- 本环境部署:`~/.dsh/profiles/superagent`(bundle 组合与模型配置)、`~/.dsh/state/superagent`(SQLite 状态与工单工作目录)
- 本报告的交付过程本身:工单由飞书消息触发,报告经 GitHub API 提交 PR —— 即第 4 节的实证案例
