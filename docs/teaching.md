# Stagehand 项目教学文档：自顶向下理解、质量评估与本地部署

> 本文面向第一次接触本仓库的开发者。它不逐行讲代码，而是用“产品目标 → 架构分层 → 功能流 → 部署与模型接入 → 质量判断”的顺序，帮助你快速判断 Stagehand 能做什么、怎么跑起来、适合什么场景。

## 1. 一句话定位

Stagehand 是一个 **AI 浏览器自动化框架**：它把 Playwright/CDP 一类确定性的浏览器控制能力，与大模型的自然语言理解能力组合起来，让开发者可以在“写代码”和“写自然语言指令”之间自由切换。

它的核心价值不是“让 AI 完全接管浏览器”，而是：

- 已知、稳定的步骤用代码或缓存动作执行；
- 页面结构复杂、经常变化、难以手写 selector 的步骤交给模型理解；
- 模型给出动作后，框架再把它转为确定性的浏览器操作；
- 后续通过缓存、自愈、日志和指标降低成本并提升可复现性。

## 2. 仓库整体结构

这是一个 pnpm + Turborepo 的 TypeScript monorepo。根目录 `package.json` 提供统一的构建、测试、lint、文档和示例入口，Node 版本要求为 `^20.19.0 || >=22.12.0`，包管理器固定为 pnpm 9.15.0。

主要包的职责如下：

| 包 | 角色 | 你应该关注什么 |
| --- | --- | --- |
| `packages/core` | 核心 SDK | `Stagehand`/`V3`、浏览器启动、`act`、`observe`、`extract`、agent、模型客户端、缓存 |
| `packages/server-v3` | 本地/服务端 REST API | 给非 JS 客户端或服务化场景使用的 Fastify API，封装 session/start/act/extract/observe 等路由 |
| `packages/cli` | CLI 工具 | 项目创建、命令行体验、脚手架和辅助能力 |
| `packages/docs` | 官方文档站 | Mintlify 文档项目 |
| `packages/evals` | 评测体系 | 自动化能力、模型能力、回归质量的评测脚手架 |

从使用者视角看，最短路径通常是：

1. 直接使用 `@browserbasehq/stagehand` SDK；
2. 如果你要跨语言/服务化，启动 `packages/server-v3`；
3. 如果你要贡献或验证质量，跑 core/server/cli/evals 的测试与 lint。

## 3. 核心抽象：Stagehand 如何工作

Stagehand V3 的高层编排器是 `V3` 类。它负责：

- 解析初始化选项；
- 创建默认或自定义 LLM client；
- 启动本地 Chrome 或连接 Browserbase 云浏览器；
- 创建 CDP 上下文；
- 初始化 `ActHandler`、`ExtractHandler`、`ObserveHandler`；
- 维护 metrics、history、event bus、cache 和生命周期清理。

### 3.1 两种运行环境

Stagehand 的浏览器运行环境分两类：

#### LOCAL

适合本地开发、内网页面测试、低成本调试。

- 可以直接启动本地 Chrome；
- 也可以通过 `localBrowserLaunchOptions.cdpUrl` 连接已有浏览器；
- 支持设置 viewport、下载路径、user data dir、keepAlive 等。

#### BROWSERBASE

适合生产、托管浏览器、反爬/验证码/会话可观测等场景。

- 需要 Browserbase API key 和 project id；
- 可创建或恢复 Browserbase session；
- 可通过 Stagehand API client 把部分 `act/extract/observe` 请求交给后端；
- 提供 session URL/debug URL，便于远程观察。

## 4. 三个最重要的 SDK 能力

### 4.1 `observe()`：先看页面，找可操作对象

`observe()` 根据当前页面的可访问性树/DOM 快照和自然语言目标，返回候选动作。它适合用在“不确定该点哪个元素”的场景。

典型用法：

```ts
const actions = await stagehand.observe("找到登录按钮");
```

返回值不是直接执行结果，而是结构化 action，可被 `act()` 复用。这也是 Stagehand 可控性高于纯 agent 的地方：你可以先观察、审阅，再执行。

### 4.2 `act()`：把自然语言变成确定性浏览器动作

`act()` 是“理解 → 动作”的入口。它会：

1. 等待页面 DOM/网络稳定；
2. 捕获页面 hybrid snapshot；
3. 把页面结构和用户指令发给模型；
4. 模型返回目标元素和操作方式；
5. Stagehand 执行确定性浏览器操作；
6. 必要时进行二阶段动作或 self-heal。

这类设计的好处是，模型不是直接操作浏览器，而是先产出“结构化意图”，再由框架执行。

### 4.3 `extract()`：从页面提取结构化数据

`extract()` 既可以无参数返回页面文本，也可以配合 Zod schema 抽取结构化数据。

典型用法：

```ts
const result = await stagehand.extract(
  "提取商品标题和价格",
  z.object({
    title: z.string(),
    price: z.string(),
  }),
);
```

默认情况下，`extract()` 主要基于页面的 a11y/DOM snapshot。它也支持 `screenshot: true`，但这有明确限制：只有 AI SDK 类型的 LLM client 支持截图输入。

## 5. Agent 与 CUA：什么时候使用？

Stagehand 同时提供普通 agent 和 Computer Use Agent（CUA）能力。

- 普通 `act/observe/extract` 更适合生产自动化：步骤可控、容易缓存、容易调试；
- `agent.execute()` 更适合多步探索型任务；
- `mode: "cua"` 适合需要模型“看屏幕并操作”的任务，例如复杂 UI、非标准控件、视觉信息很重要的页面。

但 CUA 的成本和不确定性更高。生产建议是：

1. 用普通代码处理稳定路径；
2. 用 `observe + act` 处理 selector 难点；
3. 只有视觉强依赖或步骤高度开放时才使用 CUA。

## 6. 模型接入：能不能用模型 API？

可以，而且这是本项目的核心前提之一。Stagehand 支持三种主要方式。

### 6.1 直接配置模型 API key

初始化时传入 `model`：

```ts
const stagehand = new Stagehand({
  env: "LOCAL",
  model: {
    modelName: "openai/gpt-4.1-mini",
    apiKey: process.env.OPENAI_API_KEY,
  },
});
```

如果你不显式传入 apiKey，Stagehand 会按 provider 从环境变量中加载。默认模型是 `openai/gpt-4.1-mini`。

### 6.2 使用 provider/model 格式

推荐使用 `provider/model` 格式，例如：

- `openai/gpt-4.1-mini`
- `anthropic/claude-sonnet-4-5-20250929`
- `google/gemini-3-flash-preview`
- `ollama/llama3.2`（取决于本地 Ollama 与 AI SDK provider 可用性）

项目通过 Vercel AI SDK 的 provider 体系接入多种模型供应商，包括 OpenAI、Anthropic、Google、Azure、Groq、Cerebras、Together、Mistral、DeepSeek、Perplexity、Ollama、Vertex、Gateway 等。

### 6.3 自定义 LLM client

也可以传入 `llmClient`，例如 `CustomOpenAIClient`。这种方式适合你已有统一网关、审计代理、私有 baseURL 或内部模型服务的场景。

需要注意：传入自定义 `llmClient` 时，Stagehand 会禁用内置 API 远程路径，更偏向本地直接推理调用。

## 7. 需不需要给模型 API 上传图片？

结论：**不总是需要；只有特定能力需要。**

### 7.1 普通 `act()` / `observe()` 通常不需要图片

普通动作和观察主要依赖 DOM/a11y hybrid snapshot，把页面结构、文本、元素关系交给模型。优点是：

- token 和带宽通常比截图低；
- 更容易生成可复现 selector/action；
- 对视觉模型的依赖较小。

### 7.2 `extract({ screenshot: true })` 会上传当前视口截图

当页面上的信息主要是视觉呈现、canvas、图片、复杂布局或 OCR 场景时，可以给 `extract()` 开启 `screenshot: true`。此时 Stagehand 会截取当前 viewport 的 PNG，并连同 a11y snapshot 一起交给模型。

限制：当前代码明确要求截图提取只支持 AI SDK client；非 AI SDK client 会报参数错误。

### 7.3 CUA/视觉 agent 会频繁使用截图

CUA 模式的核心就是“模型看屏幕并操作”。OpenAI/Anthropic CUA client 都有截图 provider，会在工具调用后捕获截图并回传给模型。因此：

- 如果你使用 CUA，基本可以认为会把页面截图发给模型 API；
- 如果页面包含隐私数据，应先评估脱敏、遮罩、测试账号和合规策略；
- 如果你只做 DOM 型自动化，通常不需要上传截图。

## 8. 如何在本地部署框架 + 模型

下面给出两条路径：SDK 本地运行，以及 REST API 本地运行。

### 8.1 本地 SDK 方式：最快开始

```bash
git clone https://github.com/browserbase/stagehand.git
cd stagehand
pnpm install
pnpm run build
cp .env.example .env
# 编辑 .env，至少填一个模型供应商 API key，例如 OPENAI_API_KEY / ANTHROPIC_API_KEY / GEMINI_API_KEY
pnpm run example
```

本地运行的最小代码：

```ts
import { Stagehand } from "@browserbasehq/stagehand";

const stagehand = new Stagehand({
  env: "LOCAL",
  model: {
    modelName: "openai/gpt-4.1-mini",
    apiKey: process.env.OPENAI_API_KEY,
  },
});

await stagehand.init();
const page = stagehand.context.pages()[0];
await page.goto("https://example.com");
await stagehand.act("点击 More information 链接");
await stagehand.close();
```

### 8.2 本地 REST API 方式：给其他语言或服务调用

```bash
cd packages/server-v3
pnpm install
cp .env.example .env
pnpm dev
```

服务端 API 以 Fastify 提供 `/v1/sessions/start`、`/act`、`/extract`、`/observe`、`/end` 等能力。启动 session 时可以指定：

- `browser.type = "local"`：用本地浏览器；
- `browser.cdpUrl`：连接已有 CDP 浏览器；
- `browser.launchOptions`：让 server 启动本地浏览器；
- `modelName` / `options.model.apiKey` / `x-model-api-key`：传入模型配置。

### 8.3 本地模型是否可行？

可行，但要分清能力边界：

- 文本/DOM 型 `act/observe/extract`：可以尝试 `ollama/...` 或指向本地 OpenAI-compatible endpoint 的 `baseURL`；
- 结构化输出质量会显著影响稳定性，模型越弱越容易选错元素或返回不合规 JSON；
- CUA/截图场景需要模型具备视觉和工具调用能力，本地小模型通常不如云端旗舰模型可靠；
- 如果使用 `screenshot: true` 或 CUA，本地模型还必须支持图像输入。

建议路线：

1. 开发期先用云端强模型验证流程；
2. 固化稳定动作并启用缓存；
3. 再替换成成本更低的模型或本地模型做 A/B；
4. 对关键流程保留更强模型作为 fallback。

## 9. 项目亮点分析

### 9.1 架构亮点

- **抽象边界清晰**：V3 编排器、handlers、LLM provider、CDP context、cache、server API 分工明确。
- **LOCAL/BROWSERBASE 双运行时**：同一 SDK 同时支持本地浏览器和云浏览器。
- **模型接入面广**：provider/model 格式直接接入 AI SDK provider 生态，也保留旧式原生 client。
- **可控的 AI 自动化**：`observe` 可预览动作，`act` 可执行结构化 action，比纯 agent 更适合生产。
- **缓存与自愈**：`ActCache` 和 `AgentCache` 体现了“第一次用 AI，后续尽量确定性执行”的产品方向。
- **可观测性意识较强**：metrics、history、flow logger、event store、session debug URL 都有设计。

### 9.2 工程亮点

- 使用 TypeScript、Zod、OpenAPI、Vitest、ESLint、Prettier、Turborepo，工程栈现代；
- server-v3 有单元测试和集成测试目录；
- root scripts 覆盖 build/lint/test/evals/server/cli；
- `packages/evals` 独立存在，说明项目不仅靠单元测试，也关注模型驱动任务的效果评测。

## 10. 项目质量评估

综合来看，这是一个 **质量较高、目标明确、工程化程度不错，但仍受模型不确定性和多运行时复杂度影响的项目**。

### 10.1 优点

1. **产品定位清晰**：不是泛泛的浏览器 agent，而是面向生产自动化的“代码 + AI”混合框架。
2. **分层合理**：核心 SDK、server、cli、docs、evals 分包明确。
3. **模型兼容性强**：支持多 provider、per-call model override、自定义 client、API key 继承。
4. **生产意识强**：缓存、自愈、指标、日志、远程 session、shutdown supervisor 都是生产项目才会重视的点。
5. **安全/隐私有基本边界**：默认 DOM snapshot 路径不必上传截图；截图能力是显式选项或 CUA 场景。

### 10.2 风险与不足

1. **模型结果仍是主要不确定性来源**：页面复杂、指令模糊、模型能力不足时，`act`/`extract` 会不稳定。
2. **多 provider 兼容成本高**：不同模型对 tool calling、JSON、图像、reasoning 参数支持不一致。
3. **CUA 成本和隐私风险更高**：截图会进入模型上下文，不适合未经脱敏的敏感页面。
4. **本地模型可用但不等于可靠**：尤其是结构化输出、视觉理解、复杂多步浏览器操作。
5. **依赖链较重**：浏览器、CDP、Playwright 风格 API、AI SDK、各模型 SDK、Browserbase 服务都会影响部署复杂度。

### 10.3 适合的应用场景

- 网站后台自动化；
- 表单填写、数据录入、流程测试；
- 页面结构经常变化但业务目标稳定的自动化；
- 需要从网页抽取结构化数据；
- 需要 Browserbase 云浏览器能力的生产任务。

### 10.4 不太适合的场景

- 对每一步完全确定性、零模型误差要求的关键系统；
- 高敏感数据页面且无法脱敏、无法使用私有模型；
- 强实时、超低延迟任务；
- 大规模低成本爬取但页面非常标准的任务，此时传统 Playwright 可能更经济。

## 11. 推荐学习路径

1. 先读根 README，理解 Stagehand 的产品定位；
2. 跑 `pnpm install && pnpm run build`，确认仓库可构建；
3. 用 `env: "LOCAL"` 写一个最小 `act/extract` 示例；
4. 对同一个任务尝试：纯 Playwright、`observe+act`、`agent.execute`，比较稳定性；
5. 学会看 metrics/history/logs，理解模型消耗和失败原因；
6. 如果要服务化，再学习 `packages/server-v3`；
7. 如果要上生产，重点验证缓存、自愈、模型 fallback、隐私策略和失败重试。

## 12. 最终结论

Stagehand 的核心价值在于把“AI 的灵活理解”和“浏览器自动化的确定执行”结合起来。它不是让模型盲目控制浏览器，而是尽量让模型做判断、让框架做执行；让第一次探索可以智能，后续运行可以缓存和复现。

本地部署并不困难：Node + pnpm + 一个模型 API key 即可跑 SDK；如果需要跨语言或服务化，可以启动 `server-v3`。模型 API 完全可用，并且是推荐路径。至于是否上传图片：普通 DOM 型自动化通常不需要，`extract({ screenshot: true })` 和 CUA/视觉 agent 才会上传截图。生产场景中，应优先使用 DOM snapshot 路径，只有在视觉信息不可替代时才启用截图或 CUA。
