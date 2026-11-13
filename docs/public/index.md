# NovaOS Agent SDK

The NovaOS Agent SDK is a Python library for two roles:

- **Building agents.** [`AgentBuilder`](reference/api.md#novaos_client_sdk.agent_builder) registers skills and serves them over the Agent2Agent Protocol (A2A): mount one ASGI endpoint and the builder handles JSON-RPC dispatch, task lifecycle, and streaming for you.
- **Calling agents.** [`Client`](reference/api.md#novaos_client_sdk.client) lets any Python application send a message to an agent, stream its progress, and query, cancel, or list its tasks — whether the caller is another agent or an end-user application.

Both roles ship in the same package, `novaos-agent-sdk`, imported as `novaos_client_sdk`.

## Current SDK

This documentation describes SDK `2.x`, which uses A2A JSON-RPC 2.0. For new projects, install `novaos-agent-sdk>=2,<3` and use the current [getting-started guide](getting-started.md). See [Versioning and support](versioning-and-support.md) for compatibility and documentation scope.

## Where to go next

- New to the SDK? Start with [Getting started](getting-started.md) — install, verify the import, make your first call, and build your first agent.
- New to A2A itself? [The A2A primer](concepts/a2a-primer.md) covers the vocabulary the rest of these docs assume.
- Calling an existing agent? See [Invoking agents](guides/invoking-agents.md) and [Authentication](guides/authentication.md).
- Building an agent? See [Building an agent](guides/building-an-agent.md) and [Platform tools](guides/platform-tools.md).
- Looking for a specific class or function? See the [API reference](reference/api.md).

## Reporting issues

Found a bug or have a question about the SDK itself? Open an issue on this repository's [Issues](https://github.com/OrbitronAI-Repo/novaos-agent-sdk/issues) page. For a security vulnerability, see [Security](security.md) instead of filing a public issue.
