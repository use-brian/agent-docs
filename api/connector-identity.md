---
title: Connector identity
description: When an assistant calls a tool on your custom MCP server, sidanclaw attaches server-resolved X-Sidanclaw-Actor-* headers so your server can authorize per end user.
tags: [api, connector-identity]
canonical: https://sidan.ai/docs/api/connector-identity
---

> Human-readable version: https://sidan.ai/docs/api/connector-identity

When the assistant calls a tool on your custom MCP server, sidanclaw can tell your server who the end user is. The identity is resolved by sidanclaw's backend from the authenticated session and sent as request headers, so your server can authorize per user. The model never sees or sets it, so it cannot be fabricated.

## Headers your server receives

Sent on the discovery calls and on every tool call, once the connector owner has enabled identity for your connector.

| Header | Value |
|---|---|
| `X-Sidanclaw-Actor-Channel` | The channel the message came from: `web`, `whatsapp`, `telegram`, or `slack`. |
| `X-Sidanclaw-Actor-Id` | The channel-native id of the sender: email on web, phone on WhatsApp, `@handle` on Telegram, user id on Slack. |
| `X-Sidanclaw-Actor-Email` | The sender's email when known, even on a channel turn. May be absent. |
| `X-Sidanclaw-User-Id` | The stable sidanclaw user id. Use this as the durable key; the channel-native id above can change. |

```
X-Sidanclaw-Actor-Channel: telegram
X-Sidanclaw-Actor-Id: @alice
X-Sidanclaw-Actor-Email: alice@example.com
X-Sidanclaw-User-Id: 7c2a9f04-...-e91
```

## When the headers are sent

Identity is off by default and opt-in per connector. The workspace owner enables it under Connectors, Settings, Send signed-in user identity. It applies to custom MCP servers only. Once enabled, every request to your server carries the headers.

## The contract that keeps it un-forgeable

These headers are only as trustworthy as how your server uses them. Follow all four rules.

### 1. Authorize on the header, never on a tool argument

A tool argument is filled by the model and can be hallucinated or steered by the user's text. The header is set by sidanclaw's backend after the model produced its call, so the model cannot touch it. Identify the user from the header only.

### 2. Do not expose an identity parameter in your tool schema

If your tool has a `user` or `email` parameter, the model will fill it, which reintroduces exactly the value you cannot trust. Leave identity out of the schema and let the header be the sole source.

### 3. Authenticate the connection

Give your connector a bearer token or a custom auth header. The identity header is trustworthy because only sidanclaw can reach your authenticated endpoint, the same model as a reverse proxy adding `X-Forwarded-User`. On an open endpoint, anyone could send the header, so it would mean nothing.

### 4. Key on the user id

Store and match users by `X-Sidanclaw-User-Id`. The channel-native id and email can change; the user id does not.

## Why the assistant can never fabricate it

The identity is resolved from the authenticated session, which the model never reads, and it is attached to the request after the model has produced its tool call. The model has no path to read, set, or alter it. User-configured headers also cannot use the `X-Sidanclaw-` prefix, so the namespace cannot be spoofed from configuration either.

## Other headers

Beyond identity, a connector owner can attach fixed preflight headers (for tenant, routing, or tracing) and the connector's own auth header. Identity uses the reserved `X-Sidanclaw-` namespace and always takes precedence.

## Notes for agents

- This applies to custom MCP servers you host, not to the `/messages` chat path. It is how your own tool endpoint learns the end user's identity.
- Never add a `user`, `email`, or similar identity field to a tool's input schema. Read identity from the headers only, or the model can supply a value you cannot trust.
- Authenticate your endpoint (bearer token or custom auth header). The identity headers mean nothing on an open endpoint, because anyone could set them.
- Use `X-Sidanclaw-User-Id` as the primary key for per-user storage; treat `X-Sidanclaw-Actor-Id` and `X-Sidanclaw-Actor-Email` as changeable.
- The headers arrive on discovery calls too, not only tool invocations, and only after the workspace owner opts the connector in.

## Related

- [Overview](overview.md)
- [Authentication](authentication.md)
- [Send messages](messages.md)
- [Identity & memory](identity.md)
