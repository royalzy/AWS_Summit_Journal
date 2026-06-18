# AWS Summit HK 2026 — Overall Day

**Date:** June 17, 2026  
**Where:** AWS Summit Hong Kong  
**Company:** Plaza Premium Group  

## Why I was even there

So I’m a summer intern at Plaza Premium Group right now, still in uni. My degree is CS + AI but I’m working in HR this summer (don’t ask). Keith — our Global Director for IT Infrastructure and InfoSec — brought me along because the rest of the team had to do the AI League competition thing. I basically got free rein to wander around and go to whatever talks looked interesting, then camped out in the two workshops for the rest of the day. Pretty good deal honestly.

## Keynote — the stuff that stuck

The opening keynote had people from Octopus Holdings, Vista Group, AWS, HKSTP, and the government. A couple lines actually hit:

> *“Innovation doesn't come from avoiding failures, it comes from lowering the cost of failure”*

> *“Innovation is not about being right the first time but being fast enough to adjust”*

It’s basic startup logic but it clicked more when they tied it to AI agents — because agents *will* mess up, so you need systems that fail cheaply and recover fast, not systems that try to be perfect.

Also heard this in one of the later talks:

> *“Prompt engineering is old, loop engineering is the new”*

Basically everyone can write a decent prompt now. The real work is designing the loops around it — tool calling, error handling, multi-agent coordination. That’s where the actual engineering happens.

## Security talk (Crypto.com)

Went to a session with Jason Lau (CISO at Crypto.com) and Boris So from AWS. Key points:

- AI is both a weapon and a shield for security
- AI-powered attacks are way faster than human response times
- So you need AI on defense too
- Observability and guardrails aren’t optional if you care about security

I don’t work in security, but hearing it from someone who actually handles crypto threats made it feel more real.

## Human on the loop vs human in the loop

Another thing I noted down:

- **Human in the loop** = every action needs human approval. Slows everything down.
- **Human on the loop** = human sets boundaries, monitors, steps in when needed. Agent runs autonomously within those limits.

The goal is “on the loop” for production. You want supervised autonomy, not a human bottleneck.

---

## Workshops — what I actually did

I did two hands-on workshops. Both were pretty intense but also the most useful part of the day.

# Workshop 1: AI Customer Service Agents on EKS — What I Built

By the end of this workshop, I had built a customer service chatbot system that could talk to customers, look up orders, answer product questions, and remember what people said earlier in the conversation. It all ran on Kubernetes.

## The main pieces

**A model switch (LiteLLM)** — Instead of hardcoding one AI model, I put a proxy in front. The agents talked to the proxy, and the proxy decided whether to send requests to a smaller self‑hosted model (Qwen on Inferentia) or to Amazon Bedrock (Nova). Flipping between them took one line of code. The agents themselves didn't care which one was answering.

**The agents** — I used the Strands SDK to build them. Started with one agent that could do everything, but by the end I split it into three specialists:
- *Order Agent* — checked order status, handled returns
- *Product Agent* — answered "do you have this?" style questions by searching a product database
- *Orchestrator* — listened to the user, figured out which specialist should handle it, and routed the question over

**Tool discovery** — The agents didn't have tools hardcoded into them. Instead, they asked an MCP server what tools were available at startup. So I could add or update tools (like `lookup_order` or `check_inventory`) without redeploying the agents themselves.

**Memory** — For the self‑hosted version, I used a vector database (Milvus) to store product info so the agent could do RAG. For the AWS version, I swapped that for AgentCore Memory, which remembered the actual conversation history per customer (e.g., what they asked two turns ago).

**Seeing what's happening** — Wired everything into Langfuse, which gave me a tree view of every single request. I could see: the user asked X, the agent called tool Y, it got back Z, then it called the model again, etc. Really useful for debugging.

**Running it** — Everything deployed as HTTP services on EKS. The orchestrator exposed a chat endpoint, and the UI talked to that.

So the final build was basically a fully working customer support agent system, with model switching, tool discovery, memory, multi‑agent routing, and full tracing — all on Kubernetes.

---

# Workshop 2: Quant Backtesting System — What I Built

This one was about building an automated system that helps quants test trading strategies. It took a strategy idea in plain English, ran a simulation against real market data, and gave back a performance report — and later, you could chat with it about past results.

## The main pieces

**Strategy Generator** — An agent that takes simple English descriptions (like "buy when the 10‑day average crosses above the 30‑day average") and turns it into actual Python trading code (Backtrader). Deployed as an AgentCore Runtime.

**Quant Agent (the boss)** — The main orchestrator. It had four steps it ran automatically:
1. Call the Strategy Generator to get Python code
2. Fetch real stock price data from an external source
3. Run the simulation using the generated code + real data
4. Call another agent (Result Summarizer) to write a plain‑English report on the performance

**Data access (Gateway + Identity)** — I set up an AgentCore MCP Gateway that had one tool: `get_market_data`. The Gateway used Cognito for authentication, so the agent could securely fetch data from S3 Tables without me having to manage credentials in the agent code.

**Seeing what happened** — Turned on AgentCore Observability, which sent traces to CloudWatch. I could open a trace and see: "Strategy Generator took 2s, Market Data call took 1.5s, Backtest ran in 5s, Summarizer took 1s". Also saw token usage per model.

**Memory + Chat mode** — Added AgentCore Memory so every backtest run (strategy code, trades, performance metrics) got stored. Then I uncommented a "chat mode" in the agent. In this mode, instead of running a new backtest, I could ask things like:
- "Which strategy had the best Sharpe ratio?"
- "Show me my recent runs"
- "How can I improve my EMA strategy?"
and the agent would pull from past results and answer conversationally.

**Running it** — All agents deployed with the `agentcore` CLI. The frontend (a Next.js app) called the agents via API. Full workflow took about 30‑60 seconds from "enter strategy" to "see results".

So the final build was a complete quant research assistant — describe a strategy, get it coded and backtested with real data, get a written analysis, and later, just chat with it to compare past strategies or ask for improvement ideas.

--- 
## End of day

Long day, lots of terminal commands, lots of waiting for deployments, but honestly worth it. Got to see how the patterns I studied in uni actually look in production — and the infrastructure layer is way thicker than I expected.

Glad Keith brought me along.

*— June 17, 2026*
