Workshop 1: Building Production-Ready AI Agents on EKS — Technical Breakdown

1. Model Plane (vLLM + LiteLLM)

The entire workshop ran on a single model-plane abstraction. Every agent talked to one endpoint, regardless of whether the underlying model was self-hosted or managed.

vLLM (self-hosted track)
- Model: Qwen2.5-3B-Instruct, pre-compiled for AWS Neuron using optimum-neuron
- Instance: inf2.xlarge (1 Neuron device, 2 NeuronCores)
- Key vLLM config:
  - tensor-parallel-size: 2 (distributed across both cores)
  - max-model-len: 8192
  - max-num-seqs: 2
- Deployment: Karpenter provisioned the Inferentia node when the pod requested aws.amazon.com/neuroncore
- Service: ClusterIP at qwen2-5-3b-neuron.vllm.svc.cluster.local:8000

LiteLLM (the abstraction proxy)
LiteLLM sat in front of both vLLM and Bedrock, exposing a single OpenAI-compatible /v1/chat/completions endpoint.

- Routing table (loaded at startup):
  | model_id              | upstream                                               |
  |-----------------------|--------------------------------------------------------|
  | qwen2-5-3b-neuron     | http://qwen2-5-3b-neuron.vllm.svc.cluster.local:8000/v1 |
  | nova-lite             | Amazon Bedrock (via bedrock:Converse* IAM actions)     |

- Authentication: Agents sent a master key (injected via ConfigMap). LiteLLM used Pod Identity to assume an IAM role for Bedrock — the agent pods themselves held no credentials.
- Service: ClusterIP at litellm.litellm.svc.cluster.local:4000
- Admin UI: Available at /ui on the same ALB. Showed model routes, playground, request logs, and virtual keys.

Verification commands
kubectl get pods -n vllm -l app=qwen2-5-3b-neuron

kubectl logs deploy/litellm -n litellm | grep -iE "qwen|nova-lite"

curl -s $LITELLM_URL/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "qwen2-5-3b-neuron", "messages": [{"role": "user", "content": "2+2"}]}'

curl -s $LITELLM_URL/v1/chat/completions \
  -H "Authorization: Bearer $LITELLM_KEY" \
  -d '{"model": "nova-lite", "messages": [{"role": "user", "content": "2+2"}]}'


2. Agent Framework (Strands SDK)

All agents were built with the Strands Agents SDK. The core pattern was:

from strands.models.openai import OpenAIModel
from strands import Agent

model = OpenAIModel(
    client_args={
        "base_url": os.environ["LITELLM_BASE_URL"],
        "api_key": os.environ.get("LITELLM_API_KEY", "not-needed")
    },
    model_id=os.environ.get("MODEL_ID", "nova-lite"),   # one-line backend swap
    params={"max_tokens": 1024, "temperature": 0.3}
)

agent = Agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=[lookup_order, search_products]  # or discovered via MCP
)

- Tool definition: Plain Python functions with the @tool decorator. The SDK generated OpenAPI-compatible schemas from the function signature and docstring.
- Agent loop: The Agent() class handled the full ReAct-style loop — sending messages to the LLM, deciding which tools to call, executing them, and feeding results back to the model until a final answer was produced.
- Deployment: Each agent ran as a long-lived HTTP service (FastAPI/Uvicorn) inside a Kubernetes Deployment. The agent-config ConfigMap injected environment variables (LITELLM_BASE_URL, LITELLM_API_KEY, MODEL_ID).


3. Observability (Langfuse)

Langfuse was deployed on the cluster (web, worker, PostgreSQL, ClickHouse, Redis) and ingested OpenTelemetry spans from both the Strands SDK and LiteLLM.

Integration points
- Strands SDK: With the [otel] extra, it automatically emitted spans for:
  - Agent loop (root span)
  - Each LLM call (input/output, token count)
  - Each tool invocation (input args, output)
- LiteLLM: Forwarded proxy-level spans via its Langfuse callback. This gave separate visibility into proxy routing latency vs. agent-side logic.
- Flush: Called langfuse.flush() at the end of each request to guarantee export before the HTTP response.

Trace structure (single customer query)
chat_turn (root)
├── agentcore_memory_read (if enabled)
├── agent_loop (Strands)
│   ├── llm_call_1 (prompt -> tool_choice)
│   ├── tool_call (lookup_order)
│   ├── llm_call_2 (tool_result -> final_answer)
├── litellm_proxy_span (latency from proxy perspective)
└── agentcore_memory_write (if enabled)

Key metrics visible in Langfuse UI
- Token usage per model (input / output)
- Latency breakdown (LLM vs. tool vs. network)
- Exact prompt and completion pairs
- Tool call arguments and return values

Trace comparison (self-managed vs integrated)
| Track         | model_id label          | backend              |
|---------------|-------------------------|----------------------|
| Self-managed  | qwen2-5-3b-neuron       | vLLM on Inferentia   |
| Integrated    | nova-lite               | Bedrock              |


4. Memory Systems

The workshop demonstrated two distinct memory paradigms.

4a. Self-managed: Milvus (vector DB for RAG)
- Deployment: Milvus standalone + etcd + MinIO in the milvus namespace
- Use case: Product catalog + FAQ retrieval (knowledge memory)
- Embedding model: all-MiniLM-L6-v2 via fastembed (ONNX runtime, ~20MB — PyTorch-free)
- Collection: product_catalog with ~13 product/FAQ entries
- Tool: search_products(query: str) -> list[dict]
  - Embeds the query
  - Runs milvus.search() with cosine similarity
  - Returns top-k matches (product name, description, price)
- Seeding: seed_products.py ran as a one-off job from the agent image

# rag_tools.py
@tool
def search_products(query: str) -> str:
    embedding = embed_model.encode(query)
    results = milvus_client.search(
        collection_name="product_catalog",
        data=[embedding],
        anns_field="embedding",
        limit=5
    )
    return format_results(results)

4b. Integrated: AgentCore Memory (conversational memory)
- Resource: AgentCore Memory pre-provisioned by Terraform; ID injected via agent-config ConfigMap
- Use case: Session-based conversation history (episodic memory)
- Key operations:
  - list_events(memoryId, actorId, sessionId) -> returns recent turns (newest first)
  - create_event(memoryId, actorId, sessionId, payload) -> stores user+assistant turn
- Persistence: Events expired after 30 days (set in Terraform)
- Decorators: @observe wrapped memory calls so they appeared as child spans in Langfuse

@observe(name="agentcore_memory_recent_turns")
def recent_turns(actor_id, session_id, max_results=20):
    resp = client.list_events(
        memoryId=MEMORY_ID,
        actorId=actor_id,
        sessionId=session_id,
        maxResults=max_results,
        includePayloads=True
    )
    # reverse to chronological order
    return parse_events(resp.get("events", []))

Comparison table
| Aspect               | Milvus (self-managed)          | AgentCore Memory (integrated) |
|----------------------|--------------------------------|-------------------------------|
| What it stores       | Product embeddings             | Conversation events           |
| Access pattern       | Vector similarity search       | Chronological list by session |
| Use case             | RAG / knowledge retrieval      | Session memory & personalisation |
| Tool                 | search_products                | Implicit (hydrated into system prompt) |


5. Tool Access (MCP — Model Context Protocol)

MCP decoupled tool logic from agent code. The agent discovered tools at startup from an external server.

MCP Server
- Framework: mcp.server.fastmcp.FastMCP
- Tools exposed: lookup_order(order_id), check_inventory(product_name), initiate_return(order_id, reason)
- Transport: Streamable HTTP (ASGI app served by Uvicorn)
- Deployment: Kubernetes Deployment + ClusterIP Service on port 8080

# server.py
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("AnyCompany Tools")

@mcp.tool()
def lookup_order(order_id: str) -> dict:
    """Look up order status, tracking, and details."""
    # mock data
    return mock_orders.get(order_id, {"status": "not found"})

# returns ASGI app
app = mcp.streamable_http_app()

Agent-side MCP client
# agent.py startup
mcp_client = MCPClient(lambda: streamablehttp_client(mcp_server_url))
mcp_client.__enter__()

mcp_tools = mcp_client.list_tools_sync()
print(f"Discovered {len(mcp_tools)} MCP tools")

agent = Agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=[search_products] + mcp_tools   # mix of local + MCP tools
)

- list_tools_sync() pulled the OpenAPI schemas from the MCP server's /.well-known/agent.json
- The connection stayed open for the entire pod lifetime (avoided per-request handshake overhead)
- The local mock tools.py file was completely removed — all tool logic lived on the MCP server


6. Multi-Agent Orchestration (A2A — Agent-to-Agent Protocol)

The final step split the monolithic agent into three specialists communicating via A2A JSON-RPC.

Agent roles
| Agent             | Responsibility                          | Tools available                                         |
|-------------------|-----------------------------------------|---------------------------------------------------------|
| Order Agent       | Order status, returns, inventory        | MCP tools (lookup_order, initiate_return, check_inventory) |
| Product Agent     | Product / FAQ search                    | search_products (Milvus)                                |
| Orchestrator      | Route queries to specialists            | A2A client (calls Order/Product agents as tools)       |

A2A implementation (Strands SDK + a2a-sdk)
Each specialist was an A2AStarletteApplication with:
- AgentCard: Published metadata at /.well-known/agent.json (name, capabilities, skills, URL)
- AgentExecutor: The core execution loop (execute(context, event_queue))
- InMemoryTaskStore: Managed task state for the JSON-RPC requests

# order_agent.py
agent_card = AgentCard(
    name="Order Agent",
    url="http://order-agent.default.svc.cluster.local:8081",
    capabilities=AgentCapabilities(streaming=False),
    skills=[AgentSkill(id="orders", name="Order Management", tags=["orders"])]
)

app = A2AStarletteApplication(
    agent_card=agent_card,
    http_handler=DefaultRequestHandler(OrderAgentExecutor(), InMemoryTaskStore())
)

Orchestrator routing
The orchestrator used Strands Agent with two A2A tools:

@tool
def ask_order_agent(query: str) -> str:
    """Ask the Order Agent about order status, returns, or inventory."""
    return a2a_client.send_request("order-agent", query)

@tool
def ask_product_agent(query: str) -> str:
    """Ask the Product Agent about product catalog or FAQs."""
    return a2a_client.send_request("product-agent", query)

The system prompt told the orchestrator:
"You are a routing agent. For order-related queries, call ask_order_agent. For product questions, call ask_product_agent. For ambiguous queries, ask clarifying questions."

Trace visibility (Langfuse)
A2A hops created separate top-level traces (because the JSON-RPC call crossed a process boundary), but the orchestrator trace showed ask_order_agent and ask_product_agent as tool spans. Each specialist's internal work (MCP calls, Milvus searches) appeared in its own trace.


7. Infrastructure & Networking (EKS)

Compute & autoscaling
- Karpenter: Provisioned inf2.xlarge nodes when vLLM pods requested Neuron cores
- NodePool: Configured with aws.amazon.com/neuroncore as a resource request

Networking (internal)
| Service        | Namespace  | ClusterIP                                         | Port |
|----------------|------------|---------------------------------------------------|------|
| vLLM           | vllm       | qwen2-5-3b-neuron.vllm.svc.cluster.local          | 8000 |
| LiteLLM        | litellm    | litellm.litellm.svc.cluster.local                 | 4000 |
| MCP Server     | default    | mcp-server.default.svc.cluster.local              | 8080 |
| Order Agent    | default/agents | service-specific                              | 8081 |
| Orchestrator   | default/agents | service-specific                              | 8080 |

External access
- ALB Ingress: Exposed the chat UI and LiteLLM admin UI
- Ingress URL: Retrieved via kubectl get ingress -n litellm litellm -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

Configuration injection
- ConfigMap agent-config: Contained LITELLM_BASE_URL, LITELLM_API_KEY, MODEL_ID, LANGFUSE_* variables
- Pod Identity: LiteLLM pod assumed an IAM role for Bedrock; agent pods had no credentials
- Namespace separation: Self-managed track used default; integrated track used agents (pre-wired with AgentCore IAM)

Deployment commands (self-managed track example)
docker build --push -t $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/customer-agent:strands .

envsubst < k8s.yaml | kubectl apply -f -
kubectl rollout status deployment/customer-agent --timeout=120s


Summary: What the final system actually did

By the end of the workshop, I had:

1. A model-serving layer with vLLM on Inferentia and LiteLLM as an abstraction proxy
2. A Strands-based agent that could call tools, reason in a loop, and serve HTTP requests
3. Full observability via Langfuse — traces for every LLM call, tool invocation, and proxy hop
4. Two memory systems: Milvus for product RAG, AgentCore Memory for session history
5. MCP tool discovery — agents discovered tools at startup from an external server
6. Multi-agent A2A orchestration — an orchestrator routed queries to Order and Product specialists
7. All of it running on EKS with Karpenter, ClusterIP services, and ALB ingress

The integrated track swapped vLLM for Bedrock (just model_id="nova-lite"), and Milvus for AgentCore Memory — but the agent code, SDK, and HTTP endpoints stayed identical.
