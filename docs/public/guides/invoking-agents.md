# Invoking agents

There are two distinct paths for driving agent execution, and they're not interchangeable:

- **`Client`** speaks A2A directly to an agent's own HTTP endpoint. Use this for agent-to-agent calls.
- **`GatewayClient`** drives execution through the NovaOS Execution Gateway's REST/SSE API. Use this from end-user applications (a frontend, a bot) that don't talk A2A directly.

## `Client`: send, stream, or fire-and-poll

```python
import asyncio
from novaos_client_sdk import A2AMessage, Client, Task

async def main() -> None:
    async with Client(base_url="https://my-agent.example.test") as client:
        result = await client.send_message("Hello, agent!", skill_id="chat")
        if isinstance(result, Task):
            print(result.artifacts[0].text if result.artifacts else result.status.state)
        elif isinstance(result, A2AMessage):
            print(result.text)

asyncio.run(main())
```

`send_message` calls A2A `SendMessage`. `skill_id` picks which of the agent's skills should run — A2A has no dedicated field for it, so it travels as `message.metadata["skillId"]`; a single-skill agent doesn't need it. It returns either a `Task` (the common case) or a bare `A2AMessage` when the agent answers without creating one — `isinstance` tells them apart.

By default `send_message` blocks until the task reaches a terminal or interrupt state. Pass `return_immediately=True` to get the task back as soon as it exists instead, and follow up with `subscribe_to_task` or by polling `get_task`.

For output produced incrementally, use `send_streaming_message` (A2A `SendStreamingMessage`):

```python
import asyncio
from novaos_client_sdk import A2AMessage, Client, Task, TaskState, is_last_frame, TaskArtifactUpdateEvent

async def main() -> None:
    async with Client(base_url="https://my-agent.example.test") as client:
        ended = False
        saw_artifact_update = False
        async for frame in client.send_streaming_message("Tell me a story", skill_id="storyteller"):
            if isinstance(frame, A2AMessage):
                print(frame.text)
                ended = True
                break
            if isinstance(frame, TaskArtifactUpdateEvent):
                print(frame.artifact.text, end="", flush=True)
                saw_artifact_update = True
            if is_last_frame(frame):
                if isinstance(frame, Task) and not saw_artifact_update:
                    for artifact in frame.artifacts or []:
                        print(artifact.text)
                if frame.status.state != TaskState.COMPLETED:
                    print(f"Task stopped in state: {frame.status.state}")
                ended = True
                break
        if not ended:
            raise RuntimeError("Stream ended without a final task state or direct message")

asyncio.run(main())
```

The example prints text updates as they arrive. A fallback or polling response can instead contain a complete `Task`; the final-task branch prints its artifacts when no artifact updates were displayed. For non-text artifacts or a complete persisted result, retain the task ID and inspect `get_task()` separately. An interrupted or failed state is reported explicitly rather than treated as successful completion.

Each frame is one of A2A's `StreamResponse` members: a `TaskStatusUpdateEvent`, a `TaskArtifactUpdateEvent`, a whole `Task`, or a bare `A2AMessage` — dispatch with `isinstance`, not a `kind` tag, since A2A v1.0 carries none.

Use `is_last_frame(frame)` to detect a terminal or interrupt task state. Terminal states (`COMPLETED`, `FAILED`, `CANCELED`, `REJECTED`) finish the task; interrupt states (`INPUT_REQUIRED`, `AUTH_REQUIRED`) pause it. Inspect the state to distinguish those outcomes. A direct `A2AMessage` is a response without a task and `is_last_frame` returns `False` for it. If a task stream closes without either a final state or a direct message, treat it as a transport failure. When streaming is disabled in the client configuration or the Agent Card declares no streaming support, the default fallback calls `send_message` and yields its result. With polling enabled, it can also yield later task-state frames. A client without an Agent Card still attempts streaming; a missing card alone does not trigger fallback.

## Tasks: fetch, reconnect, list, cancel

```python
# Fetch a task snapshot.
task = await client.get_task(task_id)

# Reconnect a dropped stream — the first frame is always the task's current
# state, so there's no cursor to keep. A2A does not promise to replay
# messages or artifacts missed while disconnected, only the status.
async for frame in client.subscribe_to_task(task_id):
    if is_last_frame(frame):
        break

# Enumerate tasks.
page = await client.list_tasks(status="TASK_STATE_WORKING", page_size=20)
for t in page.tasks:
    print(t.id, t.status.state)

# Ask the agent to stop a task. Cancellation is a request, not a guarantee —
# read task.status.state on the result rather than assuming it canceled.
canceled = await client.cancel_task(task_id)
```

`list_tasks` accepts a `TaskState` or its wire string for `status`, plus `context_id`, `page_token`, `history_length`, `status_timestamp_after`, and `include_artifacts` — every filter is a request the agent may honor partially. `cancel_task` raises `TaskNotCancelableError` (A2A `-32002`) if the task is already terminal.

## Discovery

Two different discovery scopes exist, matching two different questions:

**"What can this agent do, and how do I reach it?"** — [`fetch_agent_card`](../reference/api.md#novaos_client_sdk.discovery) fetches one Agent Card, the A2A document describing an agent's skills, capabilities, and `securitySchemes`:

```python
from novaos_client_sdk import fetch_agent_card

card = await fetch_agent_card("https://my-agent.example.test")
print(card.name, [s.id for s in card.skills])
```

For repeated lookups, [`AgentDiscoveryService`](../reference/api.md#novaos_client_sdk.discovery) keeps a client around and caches the card for `cache_ttl` seconds (default 300); `svc.to_client(allowed_host_suffixes=[...])` builds a `Client` from the card it already fetched, matching the protocol binding and tenant the card declares. The card's URL decides where the client connects and where its credentials are sent, so you must name the domains it may point at (`allowed_host_suffixes`).

**"Where is this agent, anywhere on the network?"** — the Agent DNS service answers that, and `discover_and_connect` combines the lookup with client construction in one call:

```python
import asyncio
from novaos_client_sdk import A2AMessage, Task, discover_and_connect, get_dns_url_from_env

async def main() -> None:
    dns_url = get_dns_url_from_env()  # reads AGENT_DNS_URL — see Getting started

    async with discover_and_connect(agent="my-agent", dns_base_url=dns_url) as client:
        result = await client.send_message("Hello", skill_id="chat")
        if isinstance(result, Task):
            print(result.artifacts[0].text if result.artifacts else result.status.state)
        elif isinstance(result, A2AMessage):
            print(result.text)

asyncio.run(main())
```

`get_dns_url_from_env()` checks, in order: an explicit `custom_url`, the variable named by `env_var` if provided, `AGENT_DNS_URL` (the platform convention), `ACP_DNS_URL`, `DNS_SERVICE_HOST`/`DNS_SERVICE_PORT`, then a per-environment variable (`ACP_DNS_URL_<ENV>`). There is no default host for any environment — it raises `ValueError` with configuration guidance if none is set.

`discover_and_connect` defaults to `trust_dns_urls=False`: a discovered URL is validated against the same private/loopback/link-local guard as any other, since Agent DNS is not itself the trusted golden path to an agent — the Execution Gateway is. Pass `trust_dns_urls=True` only when your caller knowingly needs a direct dial. For lower-level control — filtering by capability, domain, or tag before connecting — use [`AgentDNSClient`](../reference/api.md#novaos_client_sdk.dns) directly; `discover_and_connect` is built on top of it.

## `GatewayClient`: for end-user applications

The Execution Gateway is the platform's REST/SSE front door for applications that aren't themselves A2A agents — a marketplace frontend, a bot. `GatewayClient` addresses an installed Agent instance through the platform's REST/SSE contract. Authentication uses a NovaOS IAM (OIDC) bearer token — see [Authentication](authentication.md).

```python
import asyncio
from novaos_client_sdk import GatewayClient, BearerTokenAuth

async def main() -> None:
    auth = BearerTokenAuth(token="<novaos-iam-token>")

    async with GatewayClient(base_url="https://execution-gateway.example.test", auth=auth) as gw:
        result = await gw.execute(instance_id="<installed-agent-instance-uuid>", input_text="Inspire me!")

        async for event in gw.events(result.run_id):
            if event.type == "chunk":
                print(event.content, end="", flush=True)
            if event.is_terminal:
                if event.is_failure:
                    print(f"\nRun failed: {event.content}")
                break

asyncio.run(main())
```

`execute()` starts a run and returns immediately with a `run_id`; `events()` opens a Server-Sent Events subscription for it. Branch on `event.is_terminal` / `event.is_failure` rather than on `event.type == "done"` or `"error"`: the gateway can also end a run by replaying a `status` event carrying a terminal run status — `is_terminal` is the one check that covers every shape an ending arrives in. `instance_id` identifies an *installed* agent instance (not an agent name) — the UUID your application already has from installing the agent on the platform.

`GatewayClient` exposes no generic cancellation method for a run. Closing an event subscription ends the client connection; it does not establish that the run stopped. `gw.streaming_health()` checks whether the real-time SSE subsystem is available before you subscribe, so you can fail over to polling if it isn't.
