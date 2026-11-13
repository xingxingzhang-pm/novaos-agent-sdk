# NovaOS Agent SDK

[![Python Version](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](https://github.com/OrbitronAI-Repo/novaos-agent-sdk/blob/main/LICENSE)
[![PyPI](https://img.shields.io/pypi/v/novaos-agent-sdk.svg)](https://pypi.org/project/novaos-agent-sdk/)
[![Development Status](https://img.shields.io/badge/status-development%20(2.x%20A2A%20line)-orange.svg)](https://orbitronai-repo.github.io/novaos-agent-sdk/versioning-and-support/)

A Python client library for connecting to and communicating with AI Agents on the NovaOS platform. Speaks the **Agent2Agent Protocol (A2A), spec v1.0.1** for agent-to-agent communication, and the NovaOS Execution Gateway for end-user applications.

**Current SDK:** `2.x` uses Agent2Agent (A2A). For a new project, install `novaos-agent-sdk>=2,<3`. See [Versioning and support](https://orbitronai-repo.github.io/novaos-agent-sdk/versioning-and-support/) for the documentation scope and compatibility policy.

Published on PyPI as `novaos-agent-sdk`; imported as `novaos_client_sdk`.

## Installation

```bash
pip install "novaos-agent-sdk>=2,<3"
```

See [Getting started](https://orbitronai-repo.github.io/novaos-agent-sdk/getting-started/) for the full extras table (`crypto`, `llm`, `schema`, `otel`, `evaluation`, `grpc`).

## Quick start: call an agent

Under A2A the **base URL identifies the agent** — a `Client` addresses one agent, and `skill_id` chooses which of its skills runs.

```python
import asyncio
from novaos_client_sdk import A2AMessage, Client, Task

async def main() -> None:
    async with Client(base_url="http://my-agent:8000") as client:
        result = await client.send_message("Hello, agent!", skill_id="chat")

        if isinstance(result, Task):
            print(result.artifacts[0].text if result.artifacts else result.status.state)
        elif isinstance(result, A2AMessage):
            print(result.text)

asyncio.run(main())
```

## Quick start: build an agent

```python
from novaos_client_sdk import AgentBuilder, AgentExecutionContext, SkillResult

agent = AgentBuilder(name="greeter", version="1.0.0")

@agent.skill("greet", description="Greet the caller by name")
async def greet(ctx: AgentExecutionContext, input_text: str) -> SkillResult:
    return SkillResult(output=f"Hello, {input_text or 'there'}!")
```

Mount `agent.asgi_endpoint()` on your ASGI app at `A2A_JSONRPC_PATH` and run it with uvicorn. See [Getting started](https://orbitronai-repo.github.io/novaos-agent-sdk/getting-started/) for the full snippet.

## Documentation

Full documentation, including streaming, discovery, authentication, platform tools, and the complete API reference, is at:

**https://orbitronai-repo.github.io/novaos-agent-sdk/**

- [Getting started](https://orbitronai-repo.github.io/novaos-agent-sdk/getting-started/)
- [The A2A primer](https://orbitronai-repo.github.io/novaos-agent-sdk/concepts/a2a-primer/)
- [Invoking agents](https://orbitronai-repo.github.io/novaos-agent-sdk/guides/invoking-agents/)
- [Building an agent](https://orbitronai-repo.github.io/novaos-agent-sdk/guides/building-an-agent/)
- [Authentication](https://orbitronai-repo.github.io/novaos-agent-sdk/guides/authentication/)
- [Platform tools](https://orbitronai-repo.github.io/novaos-agent-sdk/guides/platform-tools/)
- [Error handling](https://orbitronai-repo.github.io/novaos-agent-sdk/guides/error-handling/)
- [API reference](https://orbitronai-repo.github.io/novaos-agent-sdk/reference/api/)
- [Versioning and support](https://orbitronai-repo.github.io/novaos-agent-sdk/versioning-and-support/)
- [Security](https://orbitronai-repo.github.io/novaos-agent-sdk/security/)

## Requirements

- Python 3.12+
- `httpx>=0.24.0` and `idna>=3.0` — required runtime dependencies: HTTP for the A2A JSON-RPC transport, DNS and the gateway services. The A2A wire format is NovaOS's own code (no vendor A2A library), verified against the official `a2a-sdk` in the test suite.

## License

MIT License (see [LICENSE](https://github.com/OrbitronAI-Repo/novaos-agent-sdk/blob/main/LICENSE))
