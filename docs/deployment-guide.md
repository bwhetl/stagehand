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

## 2. 第一阶段从零开始：源码、环境、文件放哪里

这一阶段你要做的事只有一个：**在本地跑 Stagehand，用 DeepSeek V4 Pro 做普通 DOM 型 `act/observe/extract`，不要开 screenshot。**

### 2.1 先 pull/clone 源码

如果你还没有源码：

```powershell
git clone https://github.com/browserbase/stagehand.git
cd stagehand
```

如果你已经有源码，在 Windows PowerShell 里进入你的本地目录再 pull。示例：

```powershell
cd C:\code\stagehand
git pull
```

如果你的目录不是 `C:\code\stagehand`，换成你自己的实际路径。后面所有命令默认都在这个仓库根目录执行。

### 2.2 准备 Node 和 pnpm

根目录 `package.json` 要求 Node 20.19+ 或 22.12+。先确认版本：

```powershell
node -v
pnpm -v
```

安装依赖并构建：

```powershell
pnpm install
pnpm run build
```

> 如果 `pnpm run build` 失败，先不要继续接模型。先把依赖、Node 版本、Playwright/Chrome 这些基础问题解决。

### 2.3 设置 DeepSeek API Key

推荐先直接用环境变量，不要把 key 写进代码：

```powershell
$env:DEEPSEEK_API_KEY = "你的 DeepSeek API key"
```

如果你想长期保存，可以复制 `.env.example`：

```powershell
Copy-Item .env.example .env
```

然后在 `.env` 里加：

```bash
DEEPSEEK_API_KEY=你的 DeepSeek API key
```

但注意：下面的示例脚本直接读 `process.env.DEEPSEEK_API_KEY`。在 Windows PowerShell 当前窗口里，最简单的办法就是先执行上面的 `$env:DEEPSEEK_API_KEY = "..."`，然后在同一个窗口继续运行示例。

### 2.4 你要写什么文件？

第一阶段不要改 Stagehand 核心源码。你只需要新增一个示例脚本。

建议文件位置：

```text
packages/core/examples/deepseek-local.ts
```

为什么放这里？因为 `packages/core/package.json` 已经有 example runner，可以直接跑 `packages/core/examples/*.ts`。根目录的 `pnpm run example -- <name>` 也会转到 core package 的 examples。

创建文件。Windows 下最稳的方式是直接用 VS Code：

```powershell
code packages/core/examples/deepseek-local.ts
```

然后把下面内容粘进去保存：

```ts
import { CustomOpenAIClient, Stagehand } from "../lib/v3/index.js";
import OpenAI from "openai";
import { z } from "zod";

async function main() {
  if (!process.env.DEEPSEEK_API_KEY) {
    throw new Error("Missing DEEPSEEK_API_KEY");
  }

  const stagehand = new Stagehand({
    env: "LOCAL",
    verbose: 1,
    // 第一阶段不用 screenshot，所以用 OpenAI-compatible client 最直接。
    llmClient: new CustomOpenAIClient({
      modelName: "deepseek-v4-pro",
      client: new OpenAI({
        apiKey: process.env.DEEPSEEK_API_KEY,
        baseURL: "https://api.deepseek.com",
      }),
    }),
  });

  await stagehand.init();

  try {
    const page = stagehand.context.pages()[0];
    await page.goto("https://example.com");

    const observed = await stagehand.observe("找到页面里的 More information 链接");
    console.log("observe result:", JSON.stringify(observed, null, 2));

    await stagehand.act("点击 More information 链接");

    const data = await stagehand.extract(
      "提取页面标题和一句话说明，不要使用截图",
      z.object({
        title: z.string(),
        summary: z.string(),
      }),
    );

    console.log("extract result:", JSON.stringify(data, null, 2));
  } finally {
    await stagehand.close();
  }
}

main().catch((error) => {
  console.error(error);
  process.exit(1);
});
```

这个文件就是你的第一阶段入口。它没有嵌入业务系统，也没有改 SDK，只是验证：Stagehand 能不能启动本地浏览器、能不能调用 DeepSeek、能不能完成 `observe/act/extract`。

### 2.5 怎么运行这个文件？

在仓库根目录运行：

```powershell
$env:DEEPSEEK_API_KEY = "你的 DeepSeek API key"
pnpm run example -- deepseek-local
```

如果你已经在当前 PowerShell 窗口设置过 `$env:DEEPSEEK_API_KEY`，后面只需要：

```powershell
pnpm run example -- deepseek-local
```

如果你想直接在 core 包里跑：

```powershell
cd packages/core
$env:DEEPSEEK_API_KEY = "你的 DeepSeek API key"
pnpm run example -- deepseek-local
```

### 2.6 运行成功应该看到什么？

你应该看到三类输出：

1. Stagehand 启动本地浏览器；
2. `observe result` 打印出模型认为可操作的元素；
3. `extract result` 打印出类似：

```json
{
  "title": "Example Domain",
  "summary": "This page is for use in illustrative examples in documents."
}
```

如果这一步跑通，说明第一阶段的最小链路通了：

```text
Stagehand SDK -> 本地 Chrome -> DeepSeek API -> DOM 型自动化
```

## 3. 第一阶段：接 DeepSeek V4 Pro，不用 screenshot

### 3.1 为什么推荐先用 `CustomOpenAIClient`？

第一阶段你的要求是不使用 screenshot，只跑普通 `act/observe/extract`。在这个前提下，`CustomOpenAIClient` 最直接：

- DeepSeek API 兼容 OpenAI 风格接口；
- 你可以明确写 `baseURL: "https://api.deepseek.com"`；
- 不需要先确认 AI SDK 的 DeepSeek provider 包装是否符合当前模型名；
- 不涉及 `extract({ screenshot: true })` 的限制。

所以第一阶段我建议先用上面的 `packages/core/examples/deepseek-local.ts`。

### 3.2 AI SDK provider 写法什么时候用？

如果你确认当前依赖里的 DeepSeek provider 可用，也可以改成：

```ts
const stagehand = new Stagehand({
  env: "LOCAL",
  model: {
    modelName: "deepseek/deepseek-v4-pro",
    apiKey: process.env.DEEPSEEK_API_KEY,
  },
  verbose: 1,
});
```

但第一阶段不是为了测试 provider 包装，而是为了测试 Stagehand 工作流。所以先用 `CustomOpenAIClient` 更少绕路。

### 3.3 怎么嵌入你自己的项目？

等 `deepseek-local.ts` 跑通后，再嵌入你的项目。不要一开始就嵌入业务系统。

嵌入方式有两种：

#### 方式 A：你的项目是 TypeScript/Node

安装 Stagehand 包，直接把示例里的初始化代码复制到你的项目里。核心就是：

```ts
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

然后在你的业务函数里：

```ts
await stagehand.init();
const page = stagehand.context.pages()[0];
await page.goto("你的业务页面");
await stagehand.act("你的操作指令");
await stagehand.close();
```

#### 方式 B：你的项目不是 Node/TS

先不要硬嵌 SDK。后面再用 `packages/server-v3` 把 Stagehand 包成本地 HTTP 服务，让 Python/Java/Go 通过 REST 调用。

### 3.4 第一阶段不要做什么？

先不要做这些：

- 不要开 `screenshot: true`；
- 不要上 CUA；
- 不要改 `packages/core/lib` 源码；
- 不要一开始就接 Ascend/Qwen；
- 不要直接嵌进复杂业务系统。

先跑通最小闭环，再逐步替换页面、指令、模型。

### 3.5 第一阶段重点测什么？

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

```powershell
cd C:\code\stagehand\packages\server-v3
pnpm install
Copy-Item .env.example .env
pnpm dev
```

上面的 `C:\code\stagehand` 仍然只是示例路径，换成你的 Windows 本地仓库路径。

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

## 6. 本地 CUA 的确定方案

你问的是“到底怎么搭本地 CUA”。确定回答如下：

**当前不建议用 `CustomOpenAIClient` 搭 CUA。当前最可落地的本地 CUA 路线，是把 Qwen 3.5 397B 服务伪装成 FARA/Microsoft CUA 兼容接口，然后让 Stagehand 走 `MicrosoftCUAClient`。**

原因：Stagehand 现有 CUA 路径只内置了 OpenAI、Anthropic、Google、Microsoft 四类 CUA client。其中 Microsoft 这条路最适合本地模型，因为它本来就是 OpenAI-compatible Chat Completions 调用，而且动作协议是 prompt 里的 XML tool call，不强依赖某个云厂商的专用 Responses API。

### 6.1 你要实现的接口

你的 Qwen 服务需要提供：

```text
POST http://你的-qwen-服务:8000/v1/chat/completions
```

请求里会有：

- `model`：你配置的模型名，比如 `qwen3.5-397b`；
- `messages`：包含 system prompt、用户指令、截图 `image_url`；
- `temperature`：通常为 0。

返回要兼容 OpenAI Chat Completions：

```json
{
  "choices": [
    {
      "message": {
        "content": "思考内容\n<tool_call>\n{\"name\":\"computer_use\",\"arguments\":{\"action\":\"left_click\",\"coordinate\":[420,300]}}\n</tool_call>"
      }
    }
  ],
  "usage": {
    "prompt_tokens": 1000,
    "completion_tokens": 100,
    "total_tokens": 1100
  }
}
```

Stagehand 会解析 `<tool_call>...</tool_call>` 里的 JSON，然后把这些动作转成浏览器操作。

### 6.2 Stagehand 侧配置

```ts
const stagehand = new Stagehand({
  env: "LOCAL",
  verbose: 2,
  localBrowserLaunchOptions: {
    viewport: { width: 1288, height: 711 },
    deviceScaleFactor: 1,
  },
});

await stagehand.init();

const agent = stagehand.agent({
  mode: "cua",
  model: {
    modelName: "qwen3.5-397b",
    apiKey: process.env.QWEN_LOCAL_API_KEY ?? "local",
    baseURL: "http://你的-qwen-服务:8000/v1",
    provider: "microsoft",
    maxImages: 3,
    temperature: 0,
  },
});

await agent.execute({
  instruction: "打开当前页面，找到第一个商品的价格",
  maxSteps: 10,
});
```

这里必须有 `provider: "microsoft"`。否则 Stagehand 会根据模型名判断是否是内置 CUA 模型；`qwen3.5-397b` 不在内置 CUA 模型列表里，会被拒绝。

### 6.3 Qwen 要会输出哪些动作？

至少要稳定输出这些 action：

- `left_click`：点击坐标；
- `type`：输入文本；
- `key`：按键，比如 Enter、Escape；
- `scroll`：滚动；
- `visit_url`：打开 URL；
- `wait`：等待；
- `terminate`：任务结束。

坐标来自截图，格式类似：

```xml
<tool_call>
{"name":"computer_use","arguments":{"action":"left_click","coordinate":[420,300]}}
</tool_call>
```

### 6.4 如果这条路跑不通怎么办？

有两个备选：

1. **短期备选：不用 CUA，用 hybrid/DOM agent。**
   这仍然可以用你的本地 Qwen，但主要靠 DOM 工具和部分视觉工具，不要求模型完全按 CUA 协议操作屏幕。

2. **长期备选：新增一个 `QwenCUAClient`。**
   如果你的 Qwen 服务有自己的工具调用格式，而不适合伪装成 FARA/Microsoft，就在 `packages/core/lib/v3/agent/` 里新增 client，并在 `AgentProvider` 里注册 `qwen` provider。这是改代码方案，工作量更大，但最干净。

### 6.5 验证任务

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
