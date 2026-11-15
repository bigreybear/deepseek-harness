# DSH 用户消息处理链路 —— 阅读笔记

本笔记沉淀自一次关于"用户消息发送给 agent 后，哪些模块参与处理"的分步阅读会话。
包含两部分：我对阅读方式提出的要求，以及最终确认过的技术结论。

## 一、我对阅读方式的要求

- 一步步来，每一步都引导我去看相关代码，不要一次性铺开全部内容。
- 每一步给出可以直接跳转查看的具体代码位置（文件 + 函数/行），而不是只讲抽象概念。
- 解释机制时**禁止用类比**，要用准确的专业术语和真实的调用关系描述。
- 不要为了"证明"一个简单概念去跑一堆命令（比如现搭 demo 包验证 npm bin 机制）；能直接讲清楚的就直接讲。
- 关注"进程"和"文件"这个粗粒度视角：哪些是数据/配置文件，哪些是会被执行的代码文件，它们之间的接力关系是什么。
- 对调用链上的每个接力点，要明确写出"谁在哪个文件的哪个函数里调用了谁"。

## 二、项目基本定位

DeepSeek Harness（仓库 `bigreybear/deepseek-harness`）是一个基于 Cordis 框架的全插件化 AI Agent 运行时。
核心信息来自 [AGENTS.md](/e:/repos/deepseek-harness/AGENTS.md) 和 [docs/architecture.md](/e:/repos/deepseek-harness/docs/architecture.md)：

- 没有"特权内核"：模型适配器、工具注册表、会话日志、agent loop 本身都是插件，都可被配置替换。
- Monorepo 用 pnpm workspaces，`packages/` 下是 `@deepseek-ai/dsh-<name>` 系列包；`apps/cli` 只是启动器外壳。
- 除 TypeScript 外还有 `python/` SDK 和 `native/` 原生插件；`apps/` 下还有 `web`、`desktop`。

## 三、profile / bundle / plugin 的层级关系

```
profile（命名配方，纯配置，位于 $DSH_HOME/profiles/<name>/，无代码）
  └─ bundle（代码包，贡献一份 cordis.patch.yml，一份 patch 里可以同时插入/覆盖多行插件配置）
       └─ plugin（单个插件包，真正被 Cordis 挂载，提供服务/工具/事件处理逻辑）
```

- 一个 bundle **不是**对应一个插件，而是对应"一批"插件挂载指令。
  例：[dsh-base 的 cordis.patch.yml](/e:/repos/deepseek-harness/packages/bundle/base/cordis.patch.yml) 一份 `insert` 里列了几十个插件（`llm`、`session`、`timer`……）。
- `headless` profile = `dsh-base` bundle + `dsh-headless` bundle，定义在
  [packages/boot/app-boot/src/profile.ts](/e:/repos/deepseek-harness/packages/boot/app-boot/src/profile.ts:184) 的 `PROFILE_TEMPLATES`：
  ```ts
  headless: { bundles: ['@deepseek-ai/dsh-base', '@deepseek-ai/dsh-headless'] }
  ```
- `dsh-headless` 的 patch 文件 [packages/bundle/headless/cordis.patch.yml](/e:/repos/deepseek-harness/packages/bundle/headless/cordis.patch.yml)：
  覆盖 `system-prompt`/`tools`/`hmr` 三行已有配置，并 `insert` 两个新插件：`headless-startup`、`headless-runner`。

## 四、npm/pnpm 的 `bin` 字段机制（去类比版）

两个**完全独立**的步骤，容易被混淆：

| 步骤 | 执行者 | 产物 |
|---|---|---|
| 编译/构建 | `tsc` / `tsdown` | 把 `src/bin.ts` 翻译成 `lib/bin.js`（文件内容） |
| 安装/链接（`pnpm install`） | pnpm | 读 `package.json` 的 `"bin"` 字段，在 `node_modules/.bin/` 生成一个**转发脚本**（Windows 下是 `.cmd`/`.ps1`，POSIX 下是带 shebang 的 shell 脚本），固定指向 `lib/bin.js` 的路径 |

运行 `dsh ...` 时：shell 按 `PATH` 命中这个**已经在安装阶段生成好**的转发脚本 → 转发脚本里再起一个 `node` 进程执行 `lib/bin.js`。
`package.json` 本身**不会在运行时被再次读取**，它只在安装那一刻被 pnpm 读过一次，用来决定转发脚本里写哪个目标路径。

## 五、从终端到 `bin.ts` 的两条路径

**路径 A：已安装 CLI**（如 `npm i -g` 后，或走 `node_modules/.bin`）
```
shell 按 PATH 命中 node_modules/.bin/dsh(.ps1/.cmd)
  → 转发脚本执行 node <apps/cli 绝对路径>/lib/bin.js
    → lib/bin.js（编译产物，源码为 apps/cli/src/bin.ts）
      → import.meta.main 为真 → 调用 runCli()
```
依据：[apps/cli/package.json](/e:/repos/deepseek-harness/apps/cli/package.json) 的 `"bin": { "dsh": "lib/bin.js" }`。

**路径 B：仓库内开发用的 `pnpm dsh "任务"`**
```
pnpm 读根 package.json 的 scripts.dsh 字段
  → "node --import tsx/esm apps/cli/src/bin.ts"
    → node 直接执行 src/bin.ts 源文件（tsx 钩子让 node 能跑 .ts）
      → import.meta.main 为真 → 调用 runCli()
```
依据：根 [package.json:204](/e:/repos/deepseek-harness/package.json)。两条路径最终落到同一段逻辑（`bin.ts` 的 `runCli()`），区别只是"读编译产物"还是"读源文件"。

## 六、完整调用链（终端 → `run()`），逐接力点标注代码位置

| # | 调用方（文件:位置） | 被调用方 | 调用方式 |
|---|---|---|---|
| 1 | [bin.ts](/e:/repos/deepseek-harness/apps/cli/src/bin.ts) 的 `runCli()`，`case 'profile'` 分支 | `profile-boot.ts` 的 `runProfile` | 动态 `import('./profile-boot.ts')` |
| 2 | [profile-boot.ts](/e:/repos/deepseek-harness/apps/cli/src/profile-boot.ts) 的 `runProfile` | `app-boot/index.ts` 的 `boot` | 静态顶部 `import`（跨包） |
| 3 | `profile-boot.ts` 的 `composeProfile`/`prepareProfile` | [app-boot/profile.ts](/e:/repos/deepseek-harness/packages/boot/app-boot/src/profile.ts:788) 的 `loadProfile` | 静态顶部 `import`（跨包），读 `$DSH_HOME/profiles/<name>/package.json` 的 `dsh.profile.bundles` 及各 bundle 的 `cordis.patch.yml` |
| 4 | [app-boot/index.ts](/e:/repos/deepseek-harness/packages/boot/app-boot/src/index.ts:975) 的 `boot()` | `profile-boot.ts` 传入的 `prepare` 回调 | 回调函数调用，`await prepare?.(ctx)` |
| 5 | `app-boot/index.ts` 的 `boot()` | 同文件 [`mountRootInclude`](/e:/repos/deepseek-harness/packages/boot/app-boot/src/index.ts:540) | 同文件函数调用 |
| 6 | `mountRootInclude` | vendor Cordis 的 `Loader`/`Include`（`@deepseek-ai/cordis-plugin-loader`/`cordis-plugin-include`，位于 `vendor/`） | `ctx.loader.create(rootInclude)`，`rootInclude.config.patches` 携带第 3 步解析出的"待挂载列表" |
| 7 | vendor `Include`（遍历配置里的每个包名） | 各插件包的 `apply` 导出（如 [headless/src/startup.ts](/e:/repos/deepseek-harness/packages/bundle/headless/src/startup.ts) 和 [headless/src/index.ts](/e:/repos/deepseek-harness/packages/bundle/headless/src/index.ts)） | 动态 `import(包名)` |
| 8 | `headless/src/index.ts` 的 `apply(ctx, config)` | 同文件 `run(ctx, config, io)` | 同文件函数调用，`void run(...).catch(...)`（fire-and-forget） |

**关键区分**：第 1-5 步、第 6 步前半段都只是在"组装一份待挂载的插件清单"（读 YAML/JSON 纯数据，不执行插件代码）；真正第一次执行插件代码是第 6 步末尾 `ctx.loader.create(...)` 触发 vendor `Include` 对清单逐行做动态 `import()` 之后（第 7 步）。

## 七、`run()` 内部：Agent 的创建与 `followup`

文件：[packages/bundle/headless/src/index.ts](/e:/repos/deepseek-harness/packages/bundle/headless/src/index.ts)

```
run(ctx, config, io)
  await ctx.get('loader')?.await()            等整棵插件树挂载完
  agents.create({ sessionId, meta, agentOptions, setup })
    → AgentRegistry.create (core/agent/src/index.ts:391)
      → 转调已注册工厂 AgentLoop.createAgent (core/agent-loop/src/index.ts)
          → sessions.prepare(...) 准备 Session（事件日志）
          → new ReactLoopAgent(loopCtx, id, options, session)  (core/agent-loop/src/agent.ts:98)
          → setup(agentCtx) 挂载模型选择
          → 发布到 AgentRegistry，启动 driver
  agent.followup(createUserMessage({ content:[...], source:{kind:'user'} }))
  await agent.whenIdle()
  （读取 session 日志，打印结果，io.exit）
```

### `followup` 的准确语义

`followup` **不是**通用"追问"语义，而是"排一个新的 turn"：

```ts
// core/agent-loop/src/agent.ts
followup(input: UserMessage): void {
  this.send(input, 'next-turn', true)   // target='next-turn', wakeup=true
}
steer(input: UserMessage): void {
  this.send(input, 'next-step', true)   // 插入当前运行的下一步
}
inject(input: UserMessage): void {
  this.send(input, 'next-step', false)  // 只放入，不唤醒 driver
}
```

`send()` 把消息塞进 `inbox`（`this.inbox.splice(...)`），`wakeup=true` 时额外调用 `wakeDriver()`。
特例：若 agent 正处于被取消状态（`abort.signal.aborted`），唤醒型输入会被强制改投到 `next-turn`，不能加入已中止的活动。

### `ReactLoopAgent` 结构（`core/agent-loop/src/agent.ts:98`）

| 成员 | 作用 |
|---|---|
| `id` / `session` | 同一个 id；`session` 是事件日志，模型上下文的唯一来源 |
| `inbox` | `ReactLoopInbox`，分 `next-turn`/`next-step` 两种投递边界 |
| `phase` | 状态机：`idle` / `maintenance` / 运行中，带 `AbortController` |
| `scope` / `ctx` | agent 专属插件注册作用域 |
| `dispatch` | 发布 `agent/*` 事件的分发器 |
| `systemPrompt` / `runtimeContext` | 提示词、运行时上下文的投影 |

## 八、Turn/Step 生命周期概览（尚未逐行深挖）

依据 [docs/architecture.md](/e:/repos/deepseek-harness/docs/architecture.md) "Turn flow" 和 [docs/agent-lifecycle.md](/e:/repos/deepseek-harness/docs/agent-lifecycle.md)：

- **step** = 一次模型请求 + 它调用的工具。
- **turn** = 零或多个 step：从认领第一个输入开始，到没有任何"欠账"（owed）为止结束。
- 粗流程：`turn/start` → 认领消息 → 组装 prompt（`system-prompt/assemble`）→ `agent/pre-step` 判定是否接受 → `step/start` → `agent/request` 确定路由 → 写 `system/message`/`user/message` → 流式调模型（`llm/stream`）→ 写 `assistant/message`/`assistant/attempt` → 执行工具（`tool/call`/`tool/result`）→ `step/end` → 有后续输入则继续下一 step，否则 `agent/turn-stopping` → `turn/end`。
- `turn/*`、`step/*`、`system/message`、`user/message`、`assistant/message`、`tool/*` 是**持久化到 session 日志**的事件；`agent/*` 是**进程内实时**的协调事件（状态、拦截、请求构造、纠偏、错误）。

**下一步待深挖**：driver 被 `wakeDriver()` 唤醒之后，如何认领 inbox 里的消息并写入 `turn/start`（对应 `core/agent-loop/src/agent.ts` 中 `wakeDriver` 之后的驱动循环代码）。
