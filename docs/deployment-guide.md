# Stagehand 部署指南：DeepSeek 先跑通，Qwen 再做本地 CUA

> 目标场景：你准备先接 `deepseek-v4-pro` API，不使用 screenshot；之后在 Ascend A3 16 卡服务器上部署 Qwen 3.5 397B，尝试提供本地模型服务，并进一步验证 CUA。

## 0. 先给结论

推荐路线：

1. **先用 Stagehand SDK + LOCAL 浏览器 + DeepSeek API 跑通 `act/observe/extract`。**
2. **不要一开始使用 screenshot，也不要一开始上 CUA。**
3. **等普通 DOM 型任务稳定后，再把模型切到本地 Qwen 服务。**
4. **最后单独验证 Qwen 的图片输入和 CUA 工具调用。**

原因很简单：浏览器自动化的问题很多，模型部署的问题也很多。两个复杂系统不要同时调。

## 1. 部署架构选择

### 方案 A：TypeScript SDK 直连模型 API（推荐第一阶段）

```text
你的 TS 脚本 ──> Stagehand SDK ──> 本地 Chrome
                  │
                  └──> DeepSeek API
```

优点：最少组件，最容易定位问题。

适合你第一阶段。

### 方案 B：本地 Stagehand REST API

```text
业务系统 ──HTTP──> Stagehand server-v3 ──> 本地 Chrome
                    │
                    └──> DeepSeek API / 本地 Qwen API
```

优点：可以给 Python/Java/Go 或多个服务调用。

缺点：多一个服务，多一层排错。

适合你后面要做内部平台化。

### 方案 C：本地 Qwen 模型服务 + Stagehand

```text
Stagehand ──HTTP──> 你的 Qwen OpenAI-compatible 服务 ──> Ascend A3 16 卡
```

这不是 Stagehand 自己提供的能力。Stagehand 只会调用一个模型 API；模型怎么在 Ascend 上跑，要由 vLLM/SGLang/MindIE/厂商推理栈或你自己的服务解决。

## 2. 环境准备

根目录安装：

```bash
cd /workspace/stagehand
pnpm install
pnpm run build
```

准备环境变量：

```bash
cp .env.example .env
```

第一阶段至少需要：

```bash
export DEEPSEEK_API_KEY="你的 DeepSeek API key"
```

如果你要用 Browserbase 云浏览器，才需要额外配置 Browserbase key。第一阶段不建议加这个复杂度。

## 3. 第一阶段：接 DeepSeek V4 Pro，不用 screenshot

### 3.1 推荐写法：AI SDK provider 路径

如果 DeepSeek provider 在当前依赖中可用，可以这样写：

```ts
import { Stagehand } from "@browserbasehq/stagehand";
import { z } from "zod";

const stagehand = new Stagehand({
  env: "LOCAL",
  model: {
    modelName: "deepseek/deepseek-v4-pro",
    apiKey: process.env.DEEPSEEK_API_KEY,
  },
  verbose: 1,
});

await stagehand.init();

const page = stagehand.context.pages()[0];
await page.goto("https://example.com");

await stagehand.act("点击 More information 链接");

const data = await stagehand.extract(
  "提取页面标题和主要说明",
  z.object({
    title: z.string(),
    summary: z.string(),
  }),
);

console.log(data);
await stagehand.close();
```

### 3.2 备选写法：OpenAI-compatible 自定义 client

DeepSeek API 兼容 OpenAI/Anthropic 格式。如果 AI SDK provider 路径不合适，可以用 OpenAI SDK 包一层：

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

这条路适合“不使用 screenshot”的第一阶段。

**不要这样写：**

```ts
await stagehand.extract("分析截图", schema, { screenshot: true });
```

因为 `CustomOpenAIClient` 不是 AI SDK client，当前截图提取路径会报参数错误。

### 3.3 第一阶段重点测什么？

不要只看 demo 能不能跑，要记录这些：

- `act()` 是否点对元素；
- `observe()` 返回的 action 是否可读、可复用；
- `extract()` 的 JSON 是否稳定；
- 同一个任务跑 20 次成功率多少；
- 页面轻微变化后是否还能成功；
- 延迟和 API 成本是否能接受。

**如果 DeepSeek 在 DOM 型任务上已经不稳，先别进入 CUA。CUA 只会更难。**

## 4. 第二阶段：使用本地 REST API

如果你想把 Stagehand 做成一个本地服务，而不是每个业务都写 TS SDK，可以启动 `server-v3`。

```bash
cd /workspace/stagehand/packages/server-v3
pnpm install
cp .env.example .env
pnpm dev
```

它的意义是：你通过 HTTP 创建浏览器 session、调用 act/extract/observe。

它不负责部署模型。模型仍然是：

- DeepSeek API；或
- 你的 Qwen 本地 API；或
- 其他 OpenAI-compatible 服务。

适合场景：

- 你要用 Python/Java/Go 调 Stagehand；
- 你要把浏览器会话集中在一台服务器上；
- 你要做内部自动化平台。

不适合场景：

- 你只是自己写 TypeScript 脚本；
- 你还在调模型效果；
- 你还没跑通最小链路。

## 5. 第三阶段：Ascend A3 16 卡部署 Qwen 3.5 397B

### 5.1 Stagehand 需要的不是“模型文件”，而是“模型 API”

Stagehand 不会直接加载 397B 权重。你需要先把 Qwen 部署成一个 HTTP 服务，最好兼容 OpenAI：

```text
POST /v1/chat/completions
```

然后 Stagehand 才能像调用 DeepSeek 一样调用它。

### 5.2 最小可用接口要求

普通 DOM 型 Stagehand 任务至少需要：

- 支持 chat completions；
- 支持 system/user/assistant messages；
- 支持稳定 JSON 输出；
- 支持足够长上下文；
- 延迟可接受；
- 最好支持 tool/function calling，后续 agent/CUA 会用到。

如果要做 screenshot/CUA，还需要：

- 支持图片输入；
- 接受 `image_url`、base64 image 或你的适配层能转换；
- 返回格式能被 Stagehand 当前 client 理解；
- 能稳定输出工具调用动作。

### 5.3 Qwen 3.5 397B 能不能做多模态？

如果你部署的是带视觉能力的 Qwen3.5-397B-A17B 类模型，它本身是多模态方向，可以作为 screenshot 分析和 CUA 的候选。

但落地风险在接口层：

- 推理服务是否真的打开了图片输入；
- OpenAI-compatible 协议是否和 Stagehand 期待一致；
- 图片尺寸、base64、URL、token 计费/显存占用怎么处理；
- CUA 的动作协议是否需要你写适配。

**一针见血：Qwen 多模态 ≠ Stagehand CUA 开箱即用。你还需要把“图片输入 + 工具调用 + 动作返回”这条链路接通。**

### 5.4 建议验证顺序

1. 用纯文本 prompt 调 Qwen API；
2. 用 OpenAI-compatible client 让 Stagehand 调 Qwen，先不开截图；
3. 跑 `extract()`，确认 JSON 稳定；
4. 跑 `observe()`，确认能识别按钮/输入框；
5. 跑 `act()`，确认能执行动作；
6. 单独测图片输入；
7. 最后才接 CUA。

## 6. CUA 本地化注意事项

CUA 不是“换一个模型名”那么简单。它需要模型像一个操作员一样工作：看截图、决定动作、执行后再看截图。

你要检查：

- Stagehand 当前 CUA client 支持哪些 provider；
- 你的 Qwen API 是否能模拟这些 provider 的返回格式；
- 点击、输入、滚动、截图请求的工具调用格式是否一致；
- 模型是否会输出可执行坐标/动作；
- 每一步截图的隐私风险和成本。

建议先做 5 个固定任务：

1. 打开网页并点击某个按钮；
2. 填一个简单表单但不提交；
3. 从复杂页面找一个价格；
4. 处理一个普通弹窗；
5. 在页面变化后重新定位目标。

每个任务至少跑 20 次。只看一次成功没有意义。

## 7. 不使用 screenshot 的安全边界

你第一阶段不使用 screenshot，意味着普通 `act/observe/extract` 主要把页面结构和文本发给模型，而不是图片。

但这不等于没有隐私风险：

- DOM 文本仍可能包含姓名、手机号、订单号；
- 表单 value 可能进入页面快照；
- 日志里可能记录指令和结果。

上线前要做：

- 测试账号；
- 脱敏页面；
- 不在 prompt 中放密钥；
- 控制日志落盘；
- 明确哪些页面禁止 CUA/截图。

## 8. 最终推荐配置

### 第一阶段推荐

- 浏览器：`env: "LOCAL"`；
- 模型：`deepseek-v4-pro`；
- 截图：关闭；
- API 入口：优先 SDK；
- 目标：验证 DOM 型自动化能不能稳定。

### 第二阶段推荐

- 把关键流程固定成脚本；
- 增加重试、日志、失败截图（本地保存，不一定发给模型）；
- 记录成功率和失败原因。

### 第三阶段推荐

- Qwen 先提供 OpenAI-compatible API；
- 先替换普通 LLM 调用；
- 再验证多模态；
- 最后适配 CUA。

**最终判断：你这条路线是合理的，但顺序要控制好。DeepSeek 用来验证 Stagehand 工作流；Qwen 用来验证私有化模型能力；CUA 放最后，不要一开始就把它当主方案。**
