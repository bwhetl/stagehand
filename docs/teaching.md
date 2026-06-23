# Stagehand 教学文档：先把它想简单

> 这份文档只讲你真正需要理解的东西：Stagehand 是什么、适合干什么、模型怎么接、截图什么时候会传给模型，以及本地 REST API 到底是什么。更具体的部署步骤请看 [`docs/deployment-guide.md`](./deployment-guide.md)。

## 1. Stagehand 到底是什么？

一句话：**Stagehand 是给浏览器自动化加了一个“会看页面结构的模型助手”。**

传统 Playwright/Selenium 的问题是：你要自己写 selector。网页一改，脚本容易断。纯浏览器 Agent 的问题是：它太自由，生产环境里不好控。

Stagehand 夹在中间：

- 能写死的步骤，继续用代码；
- selector 难写、页面会变的地方，让模型帮你判断；
- 模型只负责“选哪个元素、做什么动作”；
- 真正点击、输入、跳转，还是 Stagehand 用浏览器自动化去执行。

**我的判断：Stagehand 的价值不是“让 AI 自动上网”，而是把不稳定的 selector 问题变成可控的模型判断问题。**这很实用，但不是魔法。

## 2. 仓库怎么看？

你主要看这几个目录就够了：

| 目录 | 干什么 | 你关心什么 |
| --- | --- | --- |
| `packages/core` | SDK 核心 | `Stagehand`、本地浏览器、模型调用、`act/observe/extract`、agent |
| `packages/server-v3` | REST API 服务 | 如果你想把 Stagehand 当一个本地 HTTP 服务给别的语言调用，看这里 |
| `packages/cli` | 命令行工具 | 脚手架和辅助命令 |
| `packages/evals` | 评测 | 看模型/自动化效果能不能稳定复现 |
| `packages/docs` | 官方文档站 | Mintlify 文档 |

根目录是 pnpm workspace + Turborepo，常用命令是 `pnpm install`、`pnpm run build`、`pnpm run example`。

## 3. 两种浏览器运行方式

### LOCAL

浏览器在你机器或服务器上跑。

适合：开发、内网页面、私有部署、低成本验证。

你可以让 Stagehand 启动 Chrome，也可以给它一个已有浏览器的 CDP 地址。

### BROWSERBASE

浏览器跑在 Browserbase 云端。

适合：生产托管、远程调试、需要云浏览器能力的任务。

代价是你要接 Browserbase 的 API key/project id，并且要接受浏览器会话在云端运行。

**你的场景：先本地接 DeepSeek API，不用截图；后面想在 Ascend A3 上跑 Qwen 3.5 397B。第一阶段建议用 LOCAL。除非你已经决定用 Browserbase 云浏览器，否则没必要先上 BROWSERBASE。**

## 4. 三个核心方法

### `observe()`：先问“页面上有哪些可操作目标？”

它不会直接点击，而是返回可能的动作。

```ts
const actions = await stagehand.observe("找到登录按钮");
```

适合你想先看模型判断得对不对，再决定是否执行。

### `act()`：让模型帮你选元素，然后执行动作

```ts
await stagehand.act("点击登录按钮");
```

大致流程：

1. Stagehand 抓页面结构，不一定截图；
2. 把结构和你的指令发给模型；
3. 模型返回该操作哪个元素；
4. Stagehand 用浏览器自动化执行。

**重点：普通 `act()` 主要靠 DOM/a11y 结构，不是靠截图。**这正好符合你第一阶段“不使用 screenshot”的需求。

### `extract()`：从页面拿结构化数据

```ts
const data = await stagehand.extract(
  "提取商品标题和价格",
  z.object({
    title: z.string(),
    price: z.string(),
  }),
);
```

默认也主要靠页面结构和文本。只有你主动开 `screenshot: true`，它才会把当前视口截图发给模型。

## 5. Agent / CUA 什么时候用？

先说结论：**能不用 CUA 就先不用 CUA。**

- `act/observe/extract`：适合生产化流程，步骤更可控；
- 普通 agent：适合多步探索，但不如手工编排稳；
- CUA：适合必须“看屏幕”的任务，比如 canvas、复杂图形 UI、远程桌面、页面结构抓不到的控件。

CUA 的问题也很直接：

- 会频繁截图；
- 成本更高；
- 对模型多模态能力要求更高；
- 行为更像“人在试”，稳定性要靠大量评测和兜底。

**你的路线建议：DeepSeek V4 Pro 先跑 DOM 型 `act/observe/extract`；Qwen 3.5 397B 上来以后，再单独做 CUA 试验。不要一开始就把核心流程押在 CUA 上。**

## 6. 自定义 LLM client 是什么意思？

不要被名字吓到。**自定义 LLM client 就是：你不走 Stagehand 内置的模型适配器，而是自己告诉 Stagehand“怎么调用模型”。**

比如你有一个 OpenAI-compatible 服务：

- DeepSeek API；
- 公司内部模型网关；
- vLLM/SGLang/其他推理服务暴露的 `/v1/chat/completions`；
- 本地代理服务。

你可以用 `CustomOpenAIClient` 包一层 OpenAI SDK：

```ts
import { CustomOpenAIClient, Stagehand } from "@browserbasehq/stagehand";
import OpenAI from "openai";

const stagehand = new Stagehand({
  env: "LOCAL",
  llmClient: new CustomOpenAIClient({
    modelName: "deepseek-v4-pro",
    client: new OpenAI({
      apiKey: process.env.DEEPSEEK_API_KEY,
      baseURL: "https://api.deepseek.com",
    }),
  }),
});
```

这和直接写：

```ts
model: {
  modelName: "deepseek/deepseek-v4-pro",
  apiKey: process.env.DEEPSEEK_API_KEY,
}
```

不是一个入口。前者是你自己给 Stagehand 一个 client；后者是让 Stagehand 通过 AI SDK provider 去创建 client。

**关键判断：**

- 第一阶段你不用截图，DeepSeek 走 `CustomOpenAIClient` 或 AI SDK provider 都可以；
- 如果后面要 `extract({ screenshot: true })`，不要用 `CustomOpenAIClient`，因为当前代码会拒绝非 AI SDK client 的截图提取；
- 如果后面要 CUA，重点不是普通 LLM client，而是 CUA client 是否支持你的模型、截图格式、工具调用协议。

## 7. 本地 Qwen 3.5 397B 能不能分析 screenshot？

**模型本身是否多模态，和 Stagehand 能不能把截图正确送进去，是两回事。**

你提到的 Qwen 3.5 397B，如果使用的是带视觉能力的 Qwen3.5-397B-A17B 这类多模态版本，理论上可以分析 screenshot。它适合作为你后续 CUA 的候选模型。

但要落地，你还需要同时满足三件事：

1. **推理服务支持图片输入**：OpenAI-compatible 接口里要能接 `image_url` 或等价格式；
2. **Stagehand 调用路径支持图片**：`extract({ screenshot: true })` 当前要求 AI SDK client；CUA 又有自己的 OpenAI/Anthropic/Microsoft 等 client 路径；
3. **工具调用协议能对上**：CUA 不是简单问答，它要模型返回点击、输入、截图请求等动作。

所以判断很简单：

- **做 DOM 型自动化**：Qwen 只要文本/结构化输出够稳就能试；
- **做 screenshot 提取**：需要 AI SDK client 路径支持你的本地服务图片输入；
- **做本地 CUA**：还要适配 CUA 工具协议，不是把模型部署起来就自动可用。

**我的建议：先别急着把 Qwen 3.5 397B 直接接 CUA。先用同一个本地模型服务跑普通 `act/observe/extract`，确认 JSON、工具调用、长上下文、延迟都稳定，再做 CUA 适配。**

## 8. “非 AI SDK client 会报参数错误”是什么意思？

这句话只针对一个具体场景：你调用 `extract()` 时传了 `screenshot: true`。

当前代码里有检查：如果 `screenshot: true` 且当前 LLM client 不是 `aisdk` 类型，就直接报错。

所以：

```ts
await stagehand.extract("读图里的价格", schema, {
  screenshot: true,
});
```

如果你用的是 `CustomOpenAIClient`，会报错。

但下面这些不受这个限制：

- 普通 `act()`；
- 普通 `observe()`；
- 不开 `screenshot` 的 `extract()`；
- CUA 自己的截图链路。

**你的第一阶段明确不用 screenshot，所以不用被这个限制卡住。**

## 9. 本地 REST API 方式是什么？

它不是模型 API，也不是 Browserbase。它只是：**把 Stagehand SDK 包成一个本地 HTTP 服务。**

为什么需要它？

- 你想用 Python/Java/Go 调 Stagehand；
- 你想把浏览器会话集中托管在一台机器上；
- 你想让多个业务服务通过 HTTP 调用同一个 Stagehand 服务。

大致是这样：

```text
你的业务服务 ──HTTP──> stagehand server-v3 ──CDP──> 本地 Chrome
                         │
                         └──HTTP──> 模型 API / 本地模型服务
```

如果你直接写 TypeScript，用 SDK 就够了；如果你想把它做成内部服务，再用 `packages/server-v3`。

**别误解：server-v3 不会帮你部署 DeepSeek 或 Qwen。它只负责把 Stagehand 的浏览器自动化能力变成 REST API。模型服务还是你自己接。**

## 10. 项目质量：直接判断

我会这样评价 Stagehand：

### 好的地方

- 方向是对的：生产里需要“代码 + AI”，不是全自动黑盒 Agent；
- 核心 API 简洁：`act/observe/extract` 很容易理解；
- LOCAL/BROWSERBASE 两条路都支持，试验和生产都有空间；
- 支持多模型 provider，也允许你接自定义模型网关；
- 有 cache、metrics、history、evals，说明它不是 demo 项目。

### 需要警惕的地方

- 模型返回错，Stagehand 也会错；
- CUA 很诱人，但生产稳定性最难；
- 多模型兼容会有坑，尤其是 tool calling、JSON、图片输入；
- 本地大模型部署成功，不等于能稳定做浏览器自动化；
- 如果页面有敏感信息，截图链路必须单独做合规评估。

**一句话判断：Stagehand 适合用来减少 selector 和页面变化带来的维护成本；不适合拿来替代严肃流程里的全部业务逻辑。你应该把它当“自动化框架 + 模型辅助”，不要当“万能网页机器人”。**

## 11. 推荐路线

按你的计划，我建议这样走：

1. **第一阶段：DeepSeek V4 Pro + 不使用截图**
   - 用 `env: "LOCAL"`；
   - 用 `act/observe/extract`；
   - 不开 `screenshot: true`；
   - 重点测 selector 稳定性、JSON 输出、延迟和失败率。

2. **第二阶段：把流程固化**
   - 能写代码的地方写代码；
   - 只有难定位的地方用 Stagehand；
   - 给关键步骤做日志和失败重试；
   - 记录哪些任务真的需要模型。

3. **第三阶段：Ascend A3 16 卡部署 Qwen 3.5 397B**
   - 先暴露 OpenAI-compatible API；
   - 先跑文本/DOM 型任务；
   - 再验证图片输入；
   - 最后才试 CUA。

4. **第四阶段：CUA 评测**
   - 用少量固定网页任务开始；
   - 每个任务记录成功率、耗时、截图次数、token/算力成本；
   - 没有 80% 以上稳定成功率，不要进生产主链路。
