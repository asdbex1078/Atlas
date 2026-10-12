# OpenClaw：Claude 的 prompt cache 标记（OpenRouter 与直连 Anthropic 对比）

> 源码版本：openclaw-2026.7.1
> 涉及函数：`createOpenRouterSystemCacheWrapper`（`src/llm/providers/stream-wrappers/proxy.ts:134`）、`packages/ai` openai-completions 传输层、`applyAnthropicPayloadPolicyToParams`（`src/agents/anthropic-payload-policy.ts:256`）

## TL;DR

- Anthropic 的 prompt cache 必须在请求里显式带 `cache_control` 断点才会生效。OpenClaw 有**两条路径**会打这个标记：
  - **直连 Anthropic**：Anthropic Messages 原生格式，传输层组装请求体时一次打好，最多用 4 个断点。
  - **OpenRouter**：OpenAI chat 格式，OpenRouter 会把 content parts 上的 `cache_control` 透传给 Anthropic。这条路径由**两层**配合完成：
    1. `packages/ai` 的 openai-completions 传输层打一套完整标记：system 稳定前缀、最后一个 tool、最后一条稳定的对话消息；
    2. `createOpenRouterSystemCacheWrapper` 在请求发出前再补一遍：给 system 最后一块打标记，并删掉 thinking 块上的标记。
- OpenRouter 默认走哪个传输层，结果差别很大：
  - 默认走 `packages/ai`：两层都生效，结果和直连差不多。
  - 配了 `request.proxy`、`request.tls` 或 `localService`：会换成 OpenClaw 自己的传输层，**只剩 wrapper 在 system 上打的一个断点**，缓存分界也会被删掉。
- TTL：默认 5 分钟。显式配置 `cacheRetention: "long"` 时是 1h，`"none"` 时不打标记。

---

## 一、OpenRouter 路径

### 1.1 第一层：`packages/ai` 传输层（默认）

判断条件（`packages/ai/src/providers/openai-completions.ts:1373`）：

```ts
const cacheControlFormat =
  provider === "openrouter" && model.id.startsWith("anthropic/") ? "anthropic" : undefined;
```

model 配置里也可以用 `compat.cacheControlFormat` 显式覆盖（`:1439`）。

命中后，在组装请求体时打标记（`openai-completions.ts:756`、`:867`）：

```ts
function applyAnthropicCacheControl(messages, tools, cacheControl, cacheOptOutIndexes) {
  addCacheControlToSystemPrompt(messages, cacheControl);   // 第一条 system/developer
  addCacheControlToLastTool(tools, cacheControl);          // 最后一个 tool 定义
  addCacheControlToLastConversationMessage(messages, cacheControl, cacheOptOutIndexes);
}
```

- **system**：`buildCacheControlledTextParts`（`:966`）按缓存分界 `<!-- OPENCLAW_CACHE_BOUNDARY -->` 切开，**只给稳定前缀**打标记，动态后缀不打。分界在 `src/agents/system-prompt.ts:1291` 插入，前面是每轮不变的内容，后面是动态项目上下文。
- **对话**：从后往前找最后一条 user 或 assistant 消息，跳过 `cacheOptOutIndexes` 里的运行时上下文消息（`runtimeContextCarrier`，也就是每轮都变的元数据）。
- **TTL**（`getCompatCacheControl`，`:856`）：`"none"` 时不打标记；`"long"` 且 `compat.supportsLongCacheRetention` 为真时给 1h（这一项只有 Together、Cloudflare 为 false）；其他情况给 5 分钟。
- 测试：`openai-completions.test.ts:789`（按分界切开）、`:831`（跳过运行时上下文消息）。

### 1.2 第二层：`createOpenRouterSystemCacheWrapper`

挂载位置在 `src/agents/embedded-agent-runner/extra-params.ts:834`（`applyPostPluginStreamWrappers`），每次请求都会经过它。

```ts
// src/llm/providers/stream-wrappers/proxy.ts
if (
  !modelId ||
  !isAnthropicModelRef(modelId) ||                               // A：id 以 anthropic/ 开头
  !(
    endpointClass === "openrouter" ||                            // B1：host 命中 openrouter.ai
    (endpointClass === "default" && provider === "openrouter")   // B2：没配 baseUrl 且 provider=openrouter
  )
) {
  return underlying(model, context, options);                    // 原样透传
}
return streamWithPayloadPatch(underlying, model, context,
  stripCacheRetentionOption(options),
  (payloadObj) => applyAnthropicEphemeralCacheControlMarkers(
    payloadObj,
    resolveAnthropicEphemeralCacheControl(model.baseUrl, cacheRetention) ?? null,
  ));
```

- **条件 A**：`isAnthropicModelRef`（`anthropic-family-cache-semantics.ts:12`）只检查 `startsWith("anthropic/")`。
- **条件 B**：`endpointClass` 由 `resolveProviderEndpoint(baseUrl)` 算出（`src/agents/provider-attribution.ts:432`）：

  | baseUrl | endpointClass |
  |---|---|
  | 未配置 | `default` |
  | host 命中插件 manifest 的 `hostSuffixes: ["openrouter.ai"]`（`extensions/openrouter/openclaw.plugin.json:29`） | `openrouter` |
  | localhost / `*.local` / `*.internal` | `local` |
  | 其他 host（自建网关、任意 OpenAI 兼容代理） | `custom` |

- **注入方式**：`streamWithPayloadPatch`（`stream-payload-utils.ts:5`）挂一个 `onPayload` 钩子。下层 `buildParams` 组装完请求体后会调用它，钩子直接修改请求体对象，修改完才发出去（`openai-completions.ts:173-174`）。
- **`stripCacheRetentionOption`**：把 `cacheRetention` 从 options 里删掉。但更内层的 extraParams wrapper（`extra-params.ts:552-568`）会根据配置重新写回，所以下层还是能读到配置的值。
- **打标记**（`applyAnthropicEphemeralCacheControlMarkers`，`anthropic-payload-policy.ts:291`）：
  - system/developer 的 content 是字符串时，改写成 `[{ type: "text", text, cache_control }]`；
  - 是数组时，**无条件**给最后一个块（thinking 除外）打标记；
  - assistant 消息里 thinking/redacted_thinking 块上的 `cache_control` **一律删除**。OpenRouter 会拒绝挂在 reasoning 内容上的这个标记，所以 `"none"` 时也照样删（见测试 `extra-params.openrouter-cache-control.test.ts:96`）。

### 1.3 两种底层传输的实际效果

| 底层传输层 | 什么时候走 | 效果 |
|---|---|---|
| `packages/ai` openai-completions（`streamSimple`） | 默认 | 第一层打完整标记，第二层再叠加 |
| OpenClaw 传输层 `src/agents/openai-transport-stream.ts:4287` | 配了 `request.proxy`、`request.tls` 或 `localService`（`provider-transport-stream.ts:103`、`:121`） | 会先 `stripSystemPromptCacheBoundary` 删掉分界，自己不打标记，**只剩第二层在 system 上的一个断点**，对话历史不走缓存 |

### 1.4 两层的判断条件不一致

| 场景 | 第一层（看 provider 名） | 第二层（看 baseUrl host） |
|---|---|---|
| provider=`openrouter`，baseUrl=openrouter.ai 或未配置 | ✅ | ✅ |
| provider=`openrouter`，baseUrl 是自建网关 | ✅ | ❌（`custom`） |
| provider 是别的名字，baseUrl=openrouter.ai | ❌ | ✅ |
| 模型 id 没有 `anthropic/` 前缀 | ❌ | ❌ |

---

## 二、直连 Anthropic 路径

调用链：`src/agents/anthropic-transport-stream.ts:1012` 调 `resolveAnthropicPayloadPolicy({ enableCacheControl: true, ... })`，然后在 `:1124` 调 `applyAnthropicPayloadPolicyToParams`（`src/agents/anthropic-payload-policy.ts:256`）。

请求体是 Anthropic Messages 原生格式：顶层 `system` 块数组 + `tools` + `messages`。

1. **system**（`applyAnthropicCacheControlToSystem`）：
   - 有缓存分界：切成稳定前缀和动态后缀，**只给前缀打标记**；
   - 没有分界：给每个还**没有** `cache_control` 的 text 块打标记，已有的不覆盖；
   - 用 OAuth 登录（Claude 订阅）时，system 前面还有两个固定块（计费块、"You are Claude Code…"，`anthropic-transport-stream.ts:1037`），它们也各占一个断点。
2. **messages**（`applyAnthropicCacheControlToMessages`）：
   - 预算是 `ANTHROPIC_CACHE_CONTROL_LIMIT (4)` 减去 system 和 tools 已经用掉的断点；
   - 从后往前找**最后一条 user 消息**，跳过 `cacheBreakpointOptOutMessageIndexes` 里的运行时上下文消息（在 `convertAnthropicMessages` 转换时记录，`:422`、`:463`）；
   - 给最后一个 text 或 image 块打标记；有 `tool_result` 且还有剩余断点时，把 `tool_result` 也标上；只剩 1 个断点时优先标 `tool_result`。
3. **tools**：不主动打标记，只计入断点数量。
4. **`"none"`**：不打任何标记，只把 system 里的分界标记文本删掉（`stripAnthropicSystemPromptBoundary`）。

## 三、TTL 规则

wrapper 和直连路径用的是同一个 `resolveAnthropicEphemeralCacheControl`（`anthropic-payload-policy.ts:63`）：

| cacheRetention | 结果 |
|---|---|
| `"none"` | 不打标记 |
| 没配，也没有 `OPENCLAW_CACHE_RETENTION=long` | `{ type: "ephemeral" }`，5 分钟 |
| 显式配置 `"long"` | `{ type: "ephemeral", ttl: "1h" }` |
| 只通过环境变量得到 long | 只有 host 是 `api.anthropic.com` 或 Vertex 时才给 1h |

`packages/ai` 传输层走自己的 `getCompatCacheControl`，规则见 1.1。

---

## 四、对比

| 维度 | 直连 Anthropic | OpenRouter（默认传输层 + wrapper） |
|---|---|---|
| 请求格式 | Anthropic Messages：顶层 `system` 数组，内容块 | OpenAI chat：system 是一条消息，content 是 parts 数组 |
| 在哪打 | 组装请求体时打，由传输层负责 | 传输层组装时打一遍，wrapper 在 `onPayload` 里再补一遍 |
| system | 按分界切，只标稳定前缀；已有标记不覆盖 | 传输层标稳定前缀，wrapper **无条件**再标最后一块 |
| tools | 不打，只计数 | 标最后一个 tool |
| 对话历史 | 最后一条稳定的 user 消息，可带 tool_result | 最后一条稳定的 user 或 assistant 消息 |
| 断点上限 | 显式按 4 个预算分配 | 没有显式计数 |
| thinking 块 | 不需要特殊处理 | wrapper 删掉上面的 `cache_control` |
| 路由校验 | 不需要 | 只在 OpenRouter + `anthropic/*` 上打 |

## 五、为什么不一样

1. **请求格式不同。** Anthropic 原生格式的 system 在顶层，是块数组。OpenAI 格式的 system 是一条消息。OpenRouter 是把 content parts 上的 `cache_control` 原样透传，所以必须在 OpenAI 格式里的对应位置放标记，没法复用同一个函数。
2. **谁掌握上下文，谁才能标对话历史。** 直连路径和 `packages/ai` 都是在把内部消息转成请求体的**过程中**打标记，知道哪几条是运行时上下文消息，所以能把断点放在最后一条稳定消息上。wrapper 只看得到拼好的最终 JSON，分辨不出这些消息，所以只敢标位置固定的 system。它的定位是 OpenRouter 专用补丁，对话历史的缓存本来就应该由传输层负责。
3. **任意 OpenAI 兼容代理不一定接受 `cache_control`。** 发给不认识这个字段的后端可能直接报错，所以两层都只在确认是 OpenRouter + `anthropic/*` 时才打。Anthropic 官方端点天然支持，不需要这层判断。
4. **OpenRouter 会拒绝挂在 reasoning 内容上的 `cache_control`**，所以 wrapper 必须删掉 thinking 块上的标记。直连路径没有这个限制。

## 六、可能的问题（只读了代码，没跑验证）

走默认传输层时两层会叠加：
- 传输层把 system 切成 `[稳定前缀(cc), 动态后缀]`；
- wrapper **无条件**给最后一块打标记，最后一块正是**动态后缀**。

直连路径的 `applyAnthropicCacheControlToSystem` 会先检查块上有没有 `cache_control`，所以不会这样。

结果是 system 上多出一个断点，每轮动态内容一变就要写一次缓存，多一份缓存写入费用。数量上 system 2 个 + tool 1 个 + 对话 1 个，正好 4 个，没超过上限，不会报错，只是断点放得不太对。

验证方法：写一个测试，用默认 openai-completions 传输层跑 OpenRouter + `anthropic/*`，system 里带缓存分界，经过 `createOpenRouterSystemCacheWrapper` 之后，断言动态后缀上有没有 `cache_control`。

## 七、排查清单：为什么没命中缓存？

| 现象 | 原因 |
|---|---|
| provider 不叫 `openrouter`，baseUrl 也不是 openrouter.ai（比如自建网关） | 两层都不打标记 |
| 模型 id 写成 `claude-xxx`，没有 `anthropic/` 前缀 | 两层都不打标记 |
| 配了 `request.proxy`、`request.tls` 或 `localService` | 换成 OpenClaw 传输层，只剩 system 一个断点，对话历史不缓存 |
| 配置了 `cacheRetention: "none"` | 不打新标记 |
| 期望 1h，实际是 5 分钟 | 只设了环境变量，没有显式配置 `cacheRetention: "long"` |
| system 能命中，对话历史不能 | 看看是不是走了 OpenClaw 传输层，或者最后一条消息被判定成了运行时上下文 |

确认方法：检查实际的 provider 名、`baseUrl`、模型 id 写法、有没有 proxy/tls 配置，再抓一次请求体，看 `messages`、`tools` 里的 `cache_control` 都在哪。

## 测试参考

- `packages/ai/src/providers/openai-completions.test.ts:789`、`:831`：OpenRouter 传输层的分界切分和运行时上下文跳过
- `src/agents/embedded-agent-runner/extra-params.openrouter-cache-control.test.ts`：wrapper 的命中和不命中用例
- `src/llm/providers/stream-wrappers/proxy.test.ts`
- `src/agents/embedded-agent-runner-extraparams-openrouter.test.ts`
