# Agent Note: pi-ai request timeouts own their transport deadline

Status: implemented

English | [中文](2026-10-31-pi-ai-request-timeout-transport.zh.md)

## Problem

A provider profile's `timeoutMs` reached pi-ai's SDK options but not the Undici dispatcher used by the SDK's default fetch. Undici kept its process-wide header deadline, so a long local-model prompt prefill could fail before the provider returned any response headers even when the profile allowed a longer request. Replacing the global dispatcher for one request would change unrelated traffic and a bare request-local Agent would bypass the installed proxy policy.

## Decision

`dsh-http-proxy` exposes `createRequestFetch()` for consumers whose SDK accepts a fetch override and whose request needs its own Undici header or body deadline. It snapshots the active proxy policy, creates a dispatcher through the same per-origin policy factory as global fetch, and returns the fetch plus its close operation. The caller closes that dispatcher after its request settles; the process-wide dispatcher is never changed.

When a pi-ai profile sets `timeoutMs`, `dsh-llm-pi-ai` creates this request fetch with `headersTimeout: timeoutMs` and `bodyTimeout: streamIdleTimeoutMs`, then passes the fetch to pi-ai. Its existing idle watchdog remains the stream-level owner, and the adapter closes the request dispatcher on normal completion, abort, or failure. A profile without `timeoutMs` keeps using the existing global fetch path.

## Alternatives considered

**Raise the global Undici dispatcher deadline.** Rejected because one profile's model prefill would alter every outbound request in the Host and could outlive a configuration change.

**Construct a bare Agent in the pi-ai adapter.** Rejected because it would duplicate proxy routing and could silently send a request direct when the process policy requires a proxy.

**Wrap global fetch with a timeout.** Rejected because it cannot change Undici's header and body timers; aborting later leaves the earlier transport deadline intact.

**Document a larger SDK timeout only.** Rejected because that was already the affected configuration and cannot override the transport that terminates the request first.

## Consequences

- Long prefills with a configured `timeoutMs` use the same deadline at the SDK boundary and while waiting for response headers.
- Stream body inactivity remains bounded by `streamIdleTimeoutMs`, both in the existing watchdog and the request-local Undici dispatcher.
- Request-local transports preserve the active direct, proxy, and bypass policy but must be closed by their consumer.
- The fix adds no profile fields and does not alter routes that omit `timeoutMs`.

## Testing

- `packages/util/http-proxy/tests/install.spec.ts` exercises the request-local fetch through the installed proxy policy and verifies that its header deadline replaces Undici's process default.
- `packages/llm/llm-pi-ai/tests/sdk-options.spec.ts` verifies that a configured SDK timeout becomes the request fetch's header deadline, the stream idle timeout becomes its body deadline, and the adapter closes the transport.
