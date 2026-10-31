# Agent Note: pi-ai 请求超时拥有自己的传输期限

Status: implemented

[English](2026-10-31-pi-ai-request-timeout-transport.md) | 中文

## Problem

提供方 profile 的 `timeoutMs` 会传入 pi-ai 的 SDK 选项，却不会传入 SDK 默认 fetch 所使用的 Undici dispatcher。于是长时间的本地模型提示词预填充可能在提供方返回任何响应头之前被进程级 Undici 响应头期限终止，即使 profile 允许更长的请求。为单个请求替换全局 dispatcher 会影响无关流量，而裸的请求级 Agent 会绕过已安装的代理策略。

## Decision

`dsh-http-proxy` 提供 `createRequestFetch()`，供接受 fetch override 且需要独立 Undici 响应头或响应体期限的 SDK 使用。它快照当前代理策略，经由与全局 fetch 相同的按 origin 策略工厂创建 dispatcher，并返回 fetch 与 close 操作。调用方在请求结束后关闭该 dispatcher；进程级 dispatcher 不会被修改。

当 pi-ai profile 设置 `timeoutMs` 时，`dsh-llm-pi-ai` 以 `headersTimeout: timeoutMs` 和 `bodyTimeout: streamIdleTimeoutMs` 创建此请求 fetch，并把 fetch 传给 pi-ai。既有 idle watchdog 仍是流级 owner，适配器在正常完成、中止或失败时关闭请求 dispatcher。不设 `timeoutMs` 的 profile 继续使用既有全局 fetch 路径。

## Alternatives considered

**提高全局 Undici dispatcher 期限。** 否决：一条 profile 的模型预填充会改变 Host 中的每个出站请求，并且可能跨越一次配置变更。

**在 pi-ai 适配器中构造裸 Agent。** 否决：这会复制代理路由；进程策略要求代理时，它还可能静默直连。

**用超时包装全局 fetch。** 否决：它无法改变 Undici 的响应头和响应体计时器；更晚的 abort 不会移除更早的传输期限。

**只记录更大的 SDK 超时。** 否决：这已经是受影响的配置，无法覆盖先终止请求的 transport。

## Consequences

- 设置 `timeoutMs` 的长预填充会在 SDK 边界与等待响应头期间使用同一期限。
- 流响应体空闲时间仍由 `streamIdleTimeoutMs` 约束，既由既有 watchdog 约束，也由请求级 Undici dispatcher 约束。
- 请求级 transport 保留当前的直连、代理与绕过策略，但消费者必须关闭它。
- 修复不添加 profile 字段，也不改变省略 `timeoutMs` 的路由。

## Testing

- `packages/util/http-proxy/tests/install.spec.ts` 通过已安装代理策略执行请求级 fetch，并验证其响应头期限替代 Undici 进程默认值。
- `packages/llm/llm-pi-ai/tests/sdk-options.spec.ts` 验证已配置的 SDK 超时成为请求 fetch 的响应头期限、流空闲超时成为响应体期限，并验证适配器关闭 transport。
