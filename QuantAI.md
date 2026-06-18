Workshop 2: Agentic Backtesting for Quants — Technical Breakdown

1. Overall Architecture

The system was a multi-agent backtesting orchestrator built with Bedrock AgentCore and Strands Agents SDK.

                 ┌─────────────────────────────────────────────────────────────────┐
                 │                    Quant Agent (Orchestrator)                   │
                 │  (Strands Agent on AgentCore Runtime, Claude Sonnet 4.6)       │
                 │  System prompt: execute 4-step workflow automatically          │
                 └──────────────────────────┬──────────────────────────────────────┘
                                            │
            ┌───────────────────────────────┼───────────────────────────────┐
            │                               │                               │
            ▼                               ▼                               ▼
┌───────────────────────┐   ┌─────────────────────────┐   ┌─────────────────────────────────┐
│ Strategy Generator    │   │ AgentCore MCP Gateway   │   │ Result Summarizer               │
│ (AgentCore Runtime,   │   │ + Cognito Identity      │   │ (AgentCore Runtime,             │
│ Claude Opus 4.6)      │   │ → Lambda → S3 Tables    │   │ Nova Lite)                      │
│ prompt: generate      │   │ tool: get_market_data   │   │ prompt: analyse backtest        │
│ Backtrader code       │   │ (JSON‑RPC 2.0)          │   │ metrics -> JSON report          │
└───────────────────────┘   └─────────────────────────┘   └─────────────────────────────────┘

- All agents deployed via agentcore CLI
- Observability: CloudWatch traces (X‑Ray) + CloudWatch GenAI dashboard
- Memory: AgentCore Memory (event‑based persistence of backtest runs)
- Authentication: Cognito OAuth2 client_credentials for Gateway outbound calls


2. AgentCore Runtime — Strategy Generator (Lab 1)

Agent definition
- Entrypoint: strategy_generator.py
- Model: us.anthropic.claude-opus-4-6-v1:0 (Bedrock)
- Framework: Strands Agents SDK (BedrockModel)
- AgentCore wrapper: BedrockAgentCoreApp() handles HTTP lifecycle

Code structure
from strands import Agent
from strands.models.bedrock import BedrockModel
from bedrock_agentcore import BedrockAgentCoreApp

class StrategyGeneratorAgent:
    def __init__(self):
        model = BedrockModel(
            model_id=os.environ["STRATEGY_GENERATOR_MODEL_ID"],
            region_name=os.environ["AWS_REGION"]
        )
        self.agent = Agent(
            model=model,
            system_prompt=SYSTEM_PROMPT  # instructions for generating Backtrader code
        )

    def invoke(self, payload: dict) -> str:
        # payload = {"name": "...", "stock_symbol": "...", "buy_conditions": "...", ...}
        result = self.agent(json.dumps(payload))
        return result.message

SYSTEM_PROMPT (what I wrote)
"You are a trading strategy code generator. Convert JSON strategy configurations into executable Backtrader Python code. Generate clean, efficient Backtrader strategy code that: 1. Implements all buy and sell conditions from the JSON 2. Uses proper Backtrader indicators (EMA, SMA, RSI, ROC) 3. Handles stop loss and take profit if specified 4. Includes proper error handling and parameter validation. Always return complete, runnable Python code with proper imports and class structure."

Deployment commands
cd /project/workshop/backend-agents/templates/strategy-generator-agent
agentcore configure \
  --entrypoint strategy_generator.py \
  --name strategy_generator \
  --requirements-file requirements.txt \
  --non-interactive \
  --region $WORKSHOP_REGION

agentcore launch --auto-update-on-conflict \
  --env AWS_REGION=$WORKSHOP_REGION \
  --env STRATEGY_GENERATOR_MODEL_ID=us.anthropic.claude-opus-4-6-v1:0

Verify
agentcore status --agent strategy_generator

Save ARN to SSM
STRATEGY_ARN=$(grep "agent_arn" .bedrock_agentcore.yaml | awk '{print $2}')
aws ssm put-parameter --name "/workshop/backtesting/strategy-generator-arn" \
  --value "$STRATEGY_ARN" --type String --overwrite --region $WORKSHOP_REGION

Attach IAM policy (for inter-agent invocation)
ROLE_NAME=$(grep AmazonBedrockAgentCoreSDKRuntime .bedrock_agentcore.yaml | awk -F'/' '{print $NF}')
aws iam attach-role-policy --role-name "$ROLE_NAME" \
  --policy-arn "$RUNTIME_POLICY"

Test invoke
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "$STRATEGY_ARN" \
  --runtime-session-id "test-session-$(date +%s)" \
  --qualifier DEFAULT \
  --payload file:///tmp/test-strategy.json \
  --region $WORKSHOP_REGION


3. AgentCore MCP Gateway + Identity (Lab 2)

Gateway creation
The infrastructure pipeline had already deployed:
- S3 Tables bucket with historical OHLCV data (AMZN, etc.)
- Lambda function that queries S3 Tables and returns data in a given time range

My task: create an AgentCore MCP Gateway that exposes this Lambda as a tool.

Step 1: Prepare .env with S3 bucket, Lambda ARN, region
cd /project/workshop/backend-agents/templates/quant-agent/tools/market_data_mcp/deploy
cp .env.example .env
# populate with values from SSM parameters

Step 2: create_mcp_gateway.sh — does:
- Create IAM role for the Gateway
- Create Gateway endpoint (with Cognito user pool for inbound auth)
- Store gateway URL, ARN, Cognito details in SSM

Step 3: setup_gateway_target.sh — does:
- Register the Lambda as the Gateway's target (tool)
- Associate it with the Gateway

Gateway details (output stored in SSM):
- Gateway URL: https://<gateway-id>.gateway.agentcore.<region>.amazonaws.com
- Gateway ARN: arn:aws:bedrock-agentcore:<region>:<account>:gateway/<id>
- Cognito User Pool ID, Client ID, Client Secret, Domain

Step 4: Test Gateway directly
./test_gateway.sh
What it does:
- Gets OAuth2 token from Cognito (client_credentials flow)
- Calls the Gateway's /mcp endpoint with JSON‑RPC 2.0:
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_market_data",
    "arguments": {
      "symbol": "AMZN",
      "start_date": "2024-01-01",
      "end_date": "2024-12-31"
    }
  },
  "id": 1
}
- Returns OHLCV array (date, open, high, low, close, volume)

Cognito Identity flow
- Gateway requires Bearer token from Cognito
- Client credentials flow: client_id + client_secret -> access_token
- Agent's market_data tool uses the same flow internally

Key IAM roles:
- Gateway execution role: assumed by the Gateway to invoke Lambda
- Lambda execution role: read from S3 Tables
- Cognito: manages client credentials for outbound auth


4. Multi-Agent Orchestration (Lab 3)

Quant Agent (orchestrator) definition
Built with Strands Agents SDK, deployed as AgentCore Runtime.

Agent-as-Tool pattern
The orchestrator has tools that invoke other AgentCore runtimes via the Runtime API.

Tools defined in tools/strategy_generator.py and tools/result_summarizer.py:

# tools/strategy_generator.py (Challenge 1)
@tool
def generate_strategy(config: dict) -> str:
    """
    Call the Strategy Generator agent to produce Backtrader code.
    """
    # invoke_agent_runtime() using boto3 client
    response = bedrock_agentcore_runtime.invoke_agent_runtime(
        agentRuntimeArn=STRATEGY_GENERATOR_RUNTIME_ARN,
        runtimeSessionId=f"session-{uuid4()}",
        payload=json.dumps(config),
        qualifier="DEFAULT"
    )
    return response['body'].read().decode()

# tools/market_data.py
@tool
def fetch_market_data_via_gateway(symbol: str, start_date: str, end_date: str) -> dict:
    """
    Authenticate with Cognito, call AgentCore MCP Gateway for get_market_data.
    """
    # OAuth2 client_credentials flow
    token = get_cognito_token(COGNITO_DOMAIN, CLIENT_ID, CLIENT_SECRET)
    # JSON-RPC call to Gateway
    response = requests.post(
        GATEWAY_URL + "/mcp",
        headers={"Authorization": f"Bearer {token}"},
        json={
            "jsonrpc": "2.0",
            "method": "tools/call",
            "params": {
                "name": "get_market_data",
                "arguments": {"symbol": symbol, "start_date": start_date, "end_date": end_date}
            },
            "id": 1
        }
    )
    return response.json()['result']['content'][0]['text']

# tools/backtest_engine.py
@tool
def run_backtest(code: str, data: list) -> dict:
    """
    Dynamically execute Backtrader strategy code with market data.
    """
    # Use exec() to define the strategy class, then run bt.Cerebro()
    # Return performance metrics (total_return, sharpe, max_drawdown, win_rate, trades)

# tools/result_summarizer.py (invoke another agent)
@tool
def summarize_results(metrics: dict) -> str:
    # invoke AgentCore Runtime for Result Summarizer (Nova Lite)
    return invoke_summarizer(metrics)

System prompt for Quant Agent (Challenge 2)
I wrote:
"You are a quantitative backtesting orchestrator. When the user provides a strategy configuration, you MUST execute the following 4 steps in order:
1. Call generate_strategy with the user's config to get Backtrader Python code.
2. Call fetch_market_data_via_gateway to get historical data for the specified symbol.
3. Call run_backtest with the generated code and market data to get performance metrics.
4. Call summarize_results with the metrics to get a human-readable analysis.

Return the final summary to the user. If any step fails, explain what went wrong and suggest fixes. Do not ask the user for confirmation between steps — execute the full workflow automatically."

Tool registration (Challenge 3)
agent = Agent(
    model=model,
    system_prompt=SYSTEM_PROMPT,
    tools=[
        generate_strategy,
        fetch_market_data_via_gateway,
        run_backtest,
        summarize_results
    ]
)

Deploy Quant Agent
cd /project/workshop/backend-agents/templates/quant-agent
agentcore launch --auto-update-on-conflict \
  --env AWS_REGION=$WORKSHOP_REGION \
  --env QUANT_AGENT_MODEL_ID=us.anthropic.claude-sonnet-4-6 \
  --env STRATEGY_GENERATOR_RUNTIME_ARN=$STRATEGY_ARN \
  --env BACKTEST_SUMMARY_RUNTIME_ARN=$SUMMARIZER_ARN \
  --env AGENTCORE_GATEWAY_URL=$GATEWAY_URL \
  --env COGNITO_DOMAIN=$COGNITO_DOMAIN \
  --env COGNITO_CLIENT_ID=$COGNITO_CLIENT_ID \
  --env COGNITO_CLIENT_SECRET=$COGNITO_CLIENT_SECRET

Test full workflow
aws bedrock-agentcore invoke-agent-runtime \
  --agent-runtime-arn "$QUANT_ARN" \
  --runtime-session-id "full-test" \
  --payload '{"prompt": "how is the strategy performance: {\"name\":\"EMACrossover\",\"stock_symbol\":\"AMZN\",...}"}' \
  --region $WORKSHOP_REGION


5. AgentCore Observability (Lab 4)

Account-level configuration (one-time)
Step 1: Create CloudWatch resource policy to allow X-Ray and Logs Delivery to write spans.
aws logs put-resource-policy \
  --policy-name TransactionSearchXRayAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Sid": "TransactionSearchXRayAccess",
      "Effect": "Allow",
      "Principal": {"Service": ["xray.amazonaws.com", "delivery.logs.amazonaws.com"]},
      "Action": ["logs:PutLogEvents", "logs:CreateLogGroup", "logs:CreateLogStream"],
      "Resource": [
        "arn:aws:logs:'$WORKSHOP_REGION':'$ACCOUNT_ID':log-group:aws/spans:*",
        "arn:aws:logs:'$WORKSHOP_REGION':'$ACCOUNT_ID':log-group:aws/application-signals/data:*"
      ]
    }]
  }' \
  --region $WORKSHOP_REGION

Step 2: Set X-Ray trace destination to CloudWatch Logs.
aws xray update-trace-segment-destination --destination CloudWatchLogs --region $WORKSHOP_REGION

After that, redeploy agents (agentcore launch again) — they automatically emit OTel traces.

Trace view in CloudWatch
- Open CloudWatch -> GenAI Observability -> Bedrock AgentCore
- Select quant_agent, see Sessions/Traces
- Each backtest run has a trace tree:
  quant_agent (Claude Sonnet)
  ├── generate_strategy (tool call)
  │   └── strategy_generator (sub-agent span, Claude Opus)
  │       └── llm invocation (token usage, latency)
  ├── fetch_market_data (tool call)
  │   └── gateway call (HTTP span)
  ├── run_backtest (tool call)
  │   └── Python execution (custom span)
  └── summarize_results (tool call)
      └── result_summarizer (sub-agent span, Nova Lite)

Model Invocations dashboard
- Shows token usage, latency (TTFT, end-to-end), estimated TPM quota per model
- Can filter by model_id (Claude Sonnet, Claude Opus, Nova Lite)


6. AgentCore Memory + Chat Mode (Lab 5)

Memory resource
AgentCore Memory was pre-provisioned by Terraform.
- Memory ID and ARN injected via environment variables (AGENTCORE_MEMORY_ID, AGENTCORE_MEMORY_ARN)
- IAM role attached to agent service account with permissions to create_event, list_events

Memory storage schema per backtest run
Each backtest run stored as a structured event with payload:
{
  "backtest_result": {
    "symbol": "AMZN",
    "strategy_code": "class EMAStrategy(bt.Strategy): ...",
    "trades": [{"entry_date": "...", "exit_date": "...", "pnl": 123.45}, ...],
    "trade_summary": {"win_rate": 0.67, "total_trades": 3},
    "performance": {"total_return": 11.44, "sharpe": 1.01, "max_drawdown": -5.2},
    "timestamp": "2025-05-29T10:30:00Z"
  }
}

Memory integration in quant_agent (after Lab 5)
In quant_agent.py, I uncommented the chat mode code.

New tool: get_backtest_history
@tool
def get_backtest_history(limit: int = 10) -> str:
    """
    Retrieve recent backtest runs from AgentCore Memory.
    """
    events = client.list_events(
        memoryId=MEMORY_ID,
        actorId="workshop-user",
        sessionId=CURRENT_SESSION,
        maxResults=limit,
        includePayloads=True
    )
    # Format as readable text (strategy name, performance, win rate, etc.)

Chat mode system prompt (replaces the 4-step workflow prompt)
"You are a quant analyst assistant. You can help users review past backtest results, compare strategies, and suggest improvements. Use get_backtest_history to retrieve past runs. Answer questions about performance metrics (Sharpe, drawdown, total return) and provide actionable recommendations."

Mode switching
The agent's invoke() function now checks payload.get("mode"):
- mode == "backtest" (default): execute 4-step pipeline (Strategy Generator → Market Data → Backtest → Summarizer)
- mode == "chat": use the chat agent (with get_backtest_history tool) to answer questions conversationally

Deployment after uncommenting
agentcore launch --auto-update-on-conflict with same env vars (including MEMORY_ID and MEMORY_ARN).

Test chat mode
From frontend /chat tab, ask:
"Show me my recent backtest results"
"Which strategy had the best Sharpe ratio?"
"How can I improve my EMA crossover strategy?"


7. Frontend & End-to-End Flow

Frontend: Next.js app running on Code Editor EC2 instance.
- Environment: .env.local with AGENTCORE_ARN (Quant Agent ARN)
- API routes call AgentCore Runtime with the Quant Agent ARN
- Two pages: main (backtest form) and chat (chat interface)

Backtest form submission flow:
1. User fills in: strategy name, symbol, window, stop_loss, take_profit, buy/sell conditions
2. Frontend calls its own API route -> invokes Quant Agent with payload:
   { "prompt": "how is the strategy performance: {config}", "mode": "backtest" }
3. Quant Agent executes 4 steps (30-60 seconds)
4. Returns final analysis (markdown) to frontend
5. Frontend renders performance metrics and AI summary

Chat flow:
1. User types a question in /chat
2. Frontend invokes Quant Agent with mode="chat"
3. Quant Agent uses get_backtest_history, answers conversationally
4. Response displayed in chat UI


Summary: What the final system actually did

By the end of the workshop, I had built:

1. A Strategy Generator agent (Claude Opus) that converts natural‑language strategy descriptions into executable Backtrader Python code.
2. A Quant Agent orchestrator (Claude Sonnet) that executes a 4‑step backtesting pipeline: generate code → fetch market data (via Gateway + Cognito) → run backtest → summarise results.
3. An AgentCore MCP Gateway with Cognito Identity, providing secure outbound access to market data (Lambda → S3 Tables) via JSON‑RPC.
4. Full observability via CloudWatch traces, showing every agent invocation, tool call, and model latency/token usage.
5. AgentCore Memory that persists each backtest run (strategy code, trades, metrics) as structured events.
6. A chat mode that uses the same memory to answer questions about historical backtest runs (Sharpe, drawdown, improvement suggestions) — all from the same agent, just mode‑switched.
7. A Next.js frontend that calls the Quant Agent via AgentCore Runtime API, with both backtest and chat interfaces.

All agents deployed serverlessly with the agentcore CLI, with IAM roles, environment variables, and auto‑update capabilities.
