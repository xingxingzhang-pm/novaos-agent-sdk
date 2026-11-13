# Getting started

## Install

```bash
pip install "novaos-agent-sdk>=2,<3"
```

Requires Python 3.12+. A few features need an optional extra:

| Extra | Adds | Needed for |
|---|---|---|
| `crypto` | `PyJWT[crypto]`, `cryptography`, `rfc8785` | `JWKSTokenVerifier`, sealed-envelope secrets, Agent Card signature verification |
| `llm` | `openai` | `LLMClient` (calling the LLM Gateway directly) |
| `schema` | `jsonschema` | `validate_against_schema` (structured-output validation) |
| `otel` | OpenTelemetry SDK + OTLP exporter | Exporting traces, not just creating them |
| `evaluation` | `langfuse`, `openai` | The offline/CI agent-evaluation harness |
| `grpc` | `a2a-sdk` (schema only), `grpcio`, `grpcio-status` | The gRPC A2A protocol binding |
| `all` | all optional extras, including compatibility tooling | Install only when the full dependency set is needed |

```bash
pip install "novaos-agent-sdk[llm]"
```

## Verify the install

```python
import novaos_client_sdk

print(novaos_client_sdk.__version__)
```

## Your first call: run an agent

Every agent-to-agent call goes through [`Client`](reference/api.md#novaos_client_sdk.client), which speaks A2A JSON-RPC 2.0 to an agent's HTTP endpoint. Under A2A the URL *is* the agent — there is no separate agent name to pass — so a `Client` addresses one agent, and `skill_id` picks which of its skills runs.

```python
import asyncio
from novaos_client_sdk import A2AMessage, Client, Task

async def main() -> None:
    async with Client(base_url="http://localhost:8000") as client:
        result = await client.send_message("Nova", skill_id="greet")

        if isinstance(result, Task):
            print(result.artifacts[0].text if result.artifacts else result.status.state)
        elif isinstance(result, A2AMessage):
            print(result.text)

asyncio.run(main())
```

`send_message` returns either a `Task` or an `A2AMessage`; check its type before reading task state or artifacts. This example assumes the greeting agent below is running. For another agent, use its advertised skill ID and input format. See [The A2A primer](concepts/a2a-primer.md) and [Invoking agents](guides/invoking-agents.md) for streaming, task access and the Execution Gateway.

## Your first agent: build a skill

[`AgentBuilder`](reference/api.md#novaos_client_sdk.agent_builder) registers one or more **skills** — async functions that receive a governed `AgentExecutionContext` (`ctx`) and the caller's input text, and return a `SkillResult`.

```python
from novaos_client_sdk import AgentBuilder, AgentExecutionContext, SkillResult

agent = AgentBuilder(name="greeter", version="1.0.0")

@agent.skill("greet", description="Greet the caller by name")
async def greet(ctx: AgentExecutionContext, input_text: str) -> SkillResult:
    ctx.logger.info("Handling greet skill")
    return SkillResult(output=f"Hello, {input_text or 'there'}!")
```

`agent.asgi_endpoint()` turns that into a ready-made ASGI route handler — mount it at `A2A_JSONRPC_PATH` and the builder handles JSON-RPC decoding, task dispatch, and streaming for you:

```python
from fastapi import FastAPI
from novaos_client_sdk import A2A_JSONRPC_PATH

app = FastAPI()
app.post(A2A_JSONRPC_PATH)(agent.asgi_endpoint())
```

Save both server snippets in `my_agent.py`. Install its web-server dependencies explicitly:

```bash
pip install "fastapi>=0.100" "uvicorn>=0.30"
```

Run it locally (`uvicorn my_agent:app --port 8000`) and call it with the `Client` snippet above, pointed at `http://localhost:8000`. See [Building an agent](guides/building-an-agent.md) for platform tools, multi-skill routing, and streaming responses.

## Environment variables you'll actually need

Many SDK calls accept explicit constructor arguments (`base_url`, `auth`, ...). Some helpers also read environment variables; configure only the services your application uses:

- `AGENT_DNS_URL` — the recommended discovery variable, for example `http://agent-dns.example.test:8000`. You can instead pass an explicit URL or use another supported configuration source. See the [full discovery precedence](guides/invoking-agents.md#discovery); there is no built-in default discovery host.
- Authentication helpers can read a selected variable, with `ACP_API_KEY` retained as a compatibility default. See [Authentication](guides/authentication.md).

Model, memory and execution-context helpers can also use environment configuration. Follow the relevant [API reference](reference/api.md) for the helper you choose. Hosted caller identity and execution credentials depend on the target environment; installing the SDK does not provide them. Keep secrets out of source code and logs.
