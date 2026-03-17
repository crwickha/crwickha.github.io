---
layout: post
title: "Building AI Network Agents with n8n, MCP, and Cisco Meraki"
date: 2026-03-16
description: "A step-by-step guide to building 4 progressively complex AI agents that analyze, verify, and remediate Meraki network alerts — with human-in-the-loop approval via Webex."
---

What if your network could diagnose its own problems and fix them — but still ask a human before making changes?

In this post, we will build exactly that. Using n8n as the workflow engine, a custom MCP server for Meraki API access, and LLMs for reasoning, we will create four AI agents that progressively evolve from simple alert analysis to a full multi-agent remediation pipeline with human approval.

The four agents we will build:

1. **Alert Analysis** — An LLM analyzes a raw Meraki alert and posts a diagnosis to Webex. No API calls, just reasoning over the alert payload.
2. **MCP-Verified Analysis** — Same idea, but now the agent can query the Meraki dashboard via MCP to confirm what is actually happening before it responds.
3. **Single-Agent Remediation** — The agent diagnoses the issue, presents a Webex Adaptive Card with Resolve/Dismiss buttons, and a human operator decides whether to let the agent fix it.
4. **Multi-Agent Pipeline** — Same approval flow, but the remediation side splits into three specialized agents: one to verify, one to execute, one to confirm. Better design, fewer rate limit issues.

All code, configs, and workflow exports are available in the project repo:
[github.com/crwickha/n8n-agent-demo](https://github.com/crwickha/n8n-agent-demo)

Here is what the architecture looks like at a high level:

```
+------------------+       +-------------------------+
|  Meraki Dashboard| ----> | Webhook                 |
|  (Alert Webhook) |       | https://YOURDOMAIN/     |
+------------------+       |   webhook/agent-demo    |
                           +------------+------------+
                                        |
                                        v
                           +-------------------------+
                           |         n8n             |
                           |  +-------------------+  |
                           |  |    AI Agent(s)    |  |
                           |  |    (LLM + MCP)    |  |
                           |  +--------+----------+  |
                           |           |             |
                           |           v             |
                           |  +-------------------+  |
                           |  |   Meraki MCP      |  |
                           |  |   Server          |  |
                           |  | (Docker container)|  |
                           |  +--------+----------+  |
                           +-----------|-------------+
                                       |
                           +-----------v-----------+
                           |    Webex Teams Room   |
                           |  (Notifications +     |
                           |   Approval Cards)     |
                           +-----------------------+
```

## Prerequisites

Before you start, you will need:

- **Docker and Docker Compose** installed on your host. This runs fine on a Raspberry Pi, a cloud VM, or any Linux box.
- **A Cloudflare account** (free tier works). Your domain's DNS must be managed by Cloudflare.
- **A domain name** pointed to Cloudflare DNS.
- **A Cisco Meraki dashboard account** with API access enabled, and an API key.
- **A Webex Bot** — create one at [developer.webex.com](https://developer.webex.com). You will need the bot's bearer token and a room ID.
- **An OpenAI API key** and/or **Anthropic API key**. The example agents use GPT-4.1 and Claude (Haiku 4.5 / Sonnet 4.5), but you can substitute other models.
- **A Meraki network with alerting enabled** so you have real webhooks to trigger the agents.

## Cloudflare Tunnel and Domain Setup

We need n8n reachable on a public domain so Meraki can send alert webhooks and Webex can send card action callbacks. Instead of running NGINX with Let's Encrypt certificates, we use a Cloudflare Tunnel. No ports to open on your firewall, no certificates to manage, and it works behind NAT or CGNAT — which is great if you are running this on a Raspberry Pi at home.

**Step 1: Create a tunnel.**
Go to the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com), navigate to Networks -> Tunnels, and create a new tunnel. Give it a name (e.g., `n8n-tunnel`). Cloudflare will give you a tunnel token — save it, you will need it for the `.env` file.

**Step 2: Add a public hostname.**
In the tunnel configuration, add a public hostname:

- **Subdomain**: whatever you want (e.g., `n8n`)
- **Domain**: your Cloudflare-managed domain
- **Service**: `http://n8n:5678`

This tells Cloudflare to route traffic for `n8n.yourdomain.com` through the tunnel to port 5678 on the `n8n` container. Cloudflare handles HTTPS termination automatically.

**Step 3: Verify DNS.**
Cloudflare should automatically create a CNAME record pointing your subdomain to the tunnel. Verify it shows up in your DNS settings.

## Docker Compose Setup

Clone the project repo and set up your environment:

```bash
git clone https://github.com/crwickha/n8n-agent-demo.git
cd n8n-agent-demo
cp .env.example .env
```

Edit the `.env` file with your values:

```
CLOUDFLARE_TUNNEL_TOKEN=your-tunnel-token-here
MERAKI_API_KEY=your-meraki-api-key-here
MERAKI_ORG_ID=your-meraki-org-id-here
```

The `docker-compose.yml` defines three services:

```yaml
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: always
    environment:
      - N8N_HOST=YOURDOMAIN
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://YOURDOMAIN/
      - N8N_SECURE_COOKIE=true
      - N8N_PROXY_HOPS=2
      - TZ=America/Vancouver
    volumes:
      - ./data:/home/node/.n8n
      - ./files:/files
    networks:
      - n8n_network

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: always
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    networks:
      - n8n_network

  meraki-mcp:
    build:
      context: ./meraki-mcp
      dockerfile: Dockerfile
    container_name: meraki-mcp
    restart: always
    environment:
      - MERAKI_API_KEY=${MERAKI_API_KEY}
      - MERAKI_ORG_ID=${MERAKI_ORG_ID:-}
    networks:
      - n8n_network

networks:
  n8n_network:
    driver: bridge
```

A few things to note:

- **`N8N_HOST` and `WEBHOOK_URL`** must match your public domain. Replace `YOURDOMAIN` with your actual domain (e.g., `n8n.yourdomain.com`). The `WEBHOOK_URL` is what n8n uses to generate webhook URLs — if this is wrong, your webhook paths will not work.
- **`N8N_PROXY_HOPS=2`** tells n8n it is behind two proxy layers (Cloudflare and the tunnel). This ensures n8n reads the correct client IP from forwarded headers.
- **`meraki-mcp`** has no `ports` section. It is only accessible on the internal Docker network. n8n reaches it at `http://meraki-mcp:3000/mcp`. Nothing is exposed to the internet.
- **`cloudflared`** reads the tunnel token from the `.env` file and connects outbound to Cloudflare's edge. No inbound ports needed.

Bring it up:

```bash
docker compose up -d
```

Verify n8n is accessible at `https://YOURDOMAIN`. You should see the n8n setup screen on your first visit.

## The Meraki MCP Container

MCP (Model Context Protocol) is an open standard introduced by Anthropic that lets AI models call external tools through a standardized interface. Instead of hardcoding API calls into your workflow, you give the agent access to an MCP server and let it decide which tools to use based on the task.

Our MCP server wraps the Meraki Dashboard API. It runs as a Docker container on the internal network and exposes a JSON-RPC endpoint that n8n's MCP Client node can talk to.

**The Dockerfile** is straightforward:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir fastmcp meraki
COPY meraki_mcp_server.py .
EXPOSE 3000
CMD ["python", "meraki_mcp_server.py"]
```

**The server** (`meraki_mcp_server.py`) does three things:

1. **Defines tools** — each tool maps to one or more Meraki API calls. Examples: `list_organizations`, `list_networks`, `list_devices`, `get_device_detail`, `configure_switch_port`, `get_firewall_rules`, `run_ping_test`, and about 25 more. Each tool has a name, description, and input schema that the LLM can read.

2. **Handles JSON-RPC** — the server listens on port 3000 and responds to standard MCP methods: `initialize`, `tools/list` (returns available tools), and `tools/call` (executes a tool and returns the result).

3. **Calls the Meraki API** — when the LLM decides to call a tool, the server translates the request into the corresponding Meraki Python SDK call and returns the JSON result.

The key design choice: no ports are exposed outside the Docker network. The MCP server is internal-only. n8n connects to it at `http://meraki-mcp:3000/mcp`, and that traffic never leaves the Docker bridge network. The LLM gets access to Meraki data, but the Meraki API key is never exposed to the internet.

## n8n Configuration and Importing Agents

Once n8n is running, open it at `https://YOURDOMAIN` and create your account.

### Setting up credentials

Go to **Settings -> Credentials** and create:

- **OpenAI** — add your API key. Used by Agents 1, 2, and 3.
- **Anthropic** — add your API key. Used by Agents 3 and 4.

You do not need both providers for every agent. If you want to use only one LLM provider, you can swap the model nodes in each workflow after importing.

### Importing a workflow

The `agents/` directory in the project repo contains the four workflow JSON files. To import one:

1. In n8n, click **Workflows** in the left sidebar.
2. Click the **three-dot menu** (top-right) and select **Import from File**.
3. Select the Agent JSON file. The workflow opens in the editor.
4. Review the workflow, then click **Save**.

### What to update after importing

Imported workflows do not carry credential secrets or instance-specific values. You need to update a few things in each workflow:

**In every agent:**
- **LLM Credentials** — Open the OpenAI Chat Model or Anthropic Chat Model node, click the credential dropdown, and select the credential you created. The imported workflow will show a warning until you re-link it.

**In every Webex HTTP Request node:**
- **Bearer Token** — Replace `YOUR BOTS BEARER TOKEN` with your Webex bot's actual bearer token.
- **Room ID** — Replace `YOUR ROOM ID` or `YOUR WEBEX ROOM ID` with your Webex room ID.

**In Agents 3 and 4 (the Prepare Response code node):**
- **Room ID in the card body** — The Code node that builds the Adaptive Card has a hardcoded `roomId` in the JSON body. Update it to match your Webex room.

### A note on LLM choice

The exported workflows use GPT-4.1 (OpenAI) and Claude Haiku 4.5 / Sonnet 4.5 (Anthropic), but n8n supports many LLM providers — Google Gemini, Ollama for local models, Azure OpenAI, and more. You can swap the model nodes to use whatever you prefer.

That said, not all models are equal. The MCP-enabled agents (2, 3, 4) rely heavily on tool calling — the model needs to decide which Meraki tools to use, interpret the results, and sometimes chain multiple calls together. Models with weaker tool-use capabilities may call the wrong tool, pass incorrect parameters, or fail to interpret results properly. This is especially true for Agent 4, where each agent in the pipeline has strict constraints on what it should and should not do.

If you swap models, test with a real alert and watch the execution logs. Adjust prompts if the model struggles with tool selection or output formatting.

## Agent 1: Alert Analysis

This is the simplest agent and a good starting point to verify your setup is working end-to-end.

**What it does:** When a Meraki alert webhook arrives, an LLM analyzes the raw JSON payload and posts an operator-ready diagnosis to a Webex room. No MCP, no API calls — just the LLM reasoning over whatever Meraki sent.

**The flow:**

```
Webhook --> Switch (resolved?) --+--> AI Agent --> Webex
                                 |
                                 +--> Alert Resolved Message
```

**Webhook** listens at `/webhook/agent-demo`. Meraki sends alert payloads here.

**Switch** checks whether `alertData.resolved_at` exists. If the alert has already been resolved by Meraki, there is no point running an AI analysis — we just send a simple "Alert Resolved" message with the device name, alert type, and timestamps. Only unresolved (active) alerts go to the AI Agent.

**AI Agent** receives the full webhook payload and uses this prompt:

> You are an IT troubleshooting assistant. Analyze the following Cisco Meraki alert and produce a concise, operator-ready response.

The prompt instructs the LLM to produce a structured response with:
- **Issue Overview** — plain-language description of what the alert means
- **Potential Issues** — most likely causes, ordered by probability
- **How to Confirm & Resolve** — specific checks and steps for each potential issue

**Webex** posts the AI-generated markdown to your Webex room.

This agent is useful on its own for alerting, but it has a limitation: the LLM is guessing based on the alert payload alone. It has no way to check what is actually happening on the network. That is what Agent 2 fixes.

## Agent 2: MCP-Verified Analysis

**What it does:** Same trigger and output as Agent 1, but now the AI Agent has access to the Meraki MCP server. Before producing its response, the agent can query device status, check port configurations, pull client lists, and verify the issue in real time.

**The flow:**

```
Webhook --> Switch (resolved?) --+--> AI Agent --+--> Webex
                                 |               |
                                 |          MCP Client
                                 |     (meraki-mcp:3000)
                                 |
                                 +--> Alert Resolved Message
```

The only structural difference from Agent 1 is the **MCP Client** node connected to the AI Agent as a tool. The MCP Client node is configured with one setting:

- **Endpoint URL**: `http://meraki-mcp:3000/mcp`

That is it. When the agent runs, n8n sends the MCP server's tool list to the LLM alongside the prompt. The LLM then decides which tools to call — it might check `get_device_detail` for the alerting device, `list_switch_ports` to see port status, or `get_network_clients` to see what is connected.

The prompt is updated to reflect this:

> Use the Meraki MCP server to actively confirm the issue before providing remediation guidance.

And the response structure changes to include an **Issue Confirmation** section where the agent explains what it checked and what it found, rather than just listing potential issues.

The result is a more concise, data-backed response. Instead of "this could be X, Y, or Z," the agent says "I checked the device and port 4 is showing link down with no connected client — here is how to fix it."

## A Note on Notifications and Human-in-the-Loop

Before we move to Agents 3 and 4, a quick note about alternatives.

**Notifications:** We use Webex in this guide because it is where our network operations team works. But n8n has native integrations for Slack, Microsoft Teams, Discord, Telegram, email (SMTP), PagerDuty, and many more. Any of these can replace the Webex HTTP Request nodes — just swap the notification step at the end of the workflow.

**Human-in-the-loop:** n8n also has built-in options for approval workflows. The "Wait for Approval" node and "Send and Wait" patterns can pause a workflow until a human responds via email or a built-in approval link. These are simpler to set up than what we are about to build.

**Why we built a custom Webex implementation:** Our human-in-the-loop uses Webex Adaptive Cards because the notification and the approval happen in the same place. The operator sees the diagnosis, reads the recommended remediation, and clicks "Resolve" or "Dismiss" without leaving the Webex room. The Adaptive Card gives rich, contextual information inline — alert type, device serial, network name, full diagnosis text, and action buttons. The trade-off is more setup (Webex bot, webhook registration, card construction in a Code node), but the result is a purpose-built operator experience embedded in the team's existing communication tool.

Choose the approach that fits your team. The agent logic — diagnosis, MCP verification, remediation pipeline — is the same regardless of how you handle notifications and approvals.

## Agent 3: Single-Agent Remediation with Human Approval

This is where things get interesting. Agent 3 does not just analyze the alert — it diagnoses the issue using MCP, presents its findings to a human operator, and if the operator approves, it uses MCP to fix the problem.

**The key idea:** One webhook endpoint handles two different types of incoming requests. Meraki sends alert payloads, and Webex sends card action callbacks. The workflow figures out which is which and routes accordingly.

### Path 1: A new alert arrives

```
Meraki Alert
    |
    v
Webhook (/webhook/agent-demo)
    |
    v
Switch1 (resolved_at exists?)
    |
    +--[yes]--> Alert Resolved Message --> Webex
    |
    +--[no]---> Switch (attachmentAction?)
                    |
                    +--[no]---> Remediation Agent
                    |               (GPT-4.1 + MCP + Memory)
                    |                   |
                    |                   v
                    |           Prepare Response
                    |           (build Adaptive Card)
                    |                   |
                    |                   v
                    |              Webex (post card)
                    |
                    +--[yes]--> (Path 2 below)
```

1. Meraki sends an alert to `https://YOURDOMAIN/webhook/agent-demo`.
2. **Switch1** checks `resolved_at`. Active alerts continue; resolved alerts get a simple notification.
3. **Second Switch** checks if `body.resource === "attachmentActions"`. A Meraki alert will not have this, so it routes to the Remediation Agent.
4. The **Remediation Agent** (GPT-4.1 with MCP tools and a Memory node) analyzes the alert and queries Meraki to confirm the issue. It produces a diagnosis with confirmation details and guided remediation steps.
5. The **Prepare Response** Code node takes the diagnosis and builds a Webex Adaptive Card. The card includes the alert details, the full diagnosis, and two buttons: "Resolve Issue" and "Dismiss". Crucially, the card embeds the alert data and diagnosis as JSON strings in each button's submit payload — this is how the data comes back when the operator clicks a button.
6. The card is posted to your Webex room. The workflow stops here. The agent has diagnosed the issue but will not act without human approval.

### Path 2: The operator clicks "Resolve"

```
Webex attachmentAction
    |
    v
Webhook (/webhook/agent-demo)
    |
    v
Switch1 --> Switch (attachmentAction?)
                |
                +--[yes]--> HTTP Request
                            (fetch action data from Webex)
                                |
                                v
                            Format Response
                            (extract caseId, action, alert, diagnosis)
                                |
                                v
                            If (action === "resolve"?)
                                |
                                +--[yes]--> Resolution Agent
                                |           (Claude Haiku 4.5 + MCP + Memory)
                                |               |
                                |               v
                                |           Webex (post result)
                                |
                                +--[no]---> (end, dismissed)
```

1. When the operator clicks "Resolve Issue" on the Adaptive Card, Webex sends an `attachmentActions` webhook to the same `https://YOURDOMAIN/webhook/agent-demo` endpoint.
2. The switches route it to the approval path.
3. **HTTP Request** calls the Webex API to fetch the full action data. The webhook only contains an action ID — the actual button data (which button was clicked, the embedded JSON) requires a separate API call.
4. **Format Response** extracts the caseId, action (resolve or cancel), original alert data, and the diagnosis text from the card's submit payload.
5. **If** checks whether the operator clicked "Resolve" or "Dismiss".
6. The **Resolution Agent** (Claude Haiku 4.5 with MCP and shared Memory) receives the alert data and diagnosis. It uses MCP to execute the recommended fix and posts the result back to Webex.

### How the pieces connect

**Shared memory:** Both the Remediation Agent and Resolution Agent use a Buffer Window Memory node with the `caseId` as the session key. This means when the Resolution Agent runs, it has the full conversation context from the diagnosis phase — it knows what was checked and what was found.

**Data round-trip through the card:** The Adaptive Card is not just a notification — it is a data transport. The Prepare Response code node serializes the alert data and diagnosis into the button's submit payload. When the operator clicks a button, that data comes back through the Webex webhook. This means the Resolution Agent has everything it needs without a separate database or state store.

**One webhook, two purposes:** Both Meraki and Webex send their payloads to the same `/webhook/agent-demo` endpoint. The Switch nodes inspect the payload to determine the source and route accordingly. This keeps the setup simple — one webhook URL to configure everywhere.

## Agent 4: Multi-Agent Pipeline

Agent 3 works, but the Resolution Agent has a lot on its plate. It needs to verify the current state, execute the fix, and confirm the result — all in one prompt, all in one sequence of API calls. On complex remediations, this can lead to Meraki API rate limits or confused tool use when the model tries to do too much at once.

Agent 4 splits the resolution into three focused agents, each with a single job and strict constraints on what it can do.

### Same diagnosis and approval flow

The front half of Agent 4 is identical to Agent 3: the Remediation Agent diagnoses the issue using MCP, a Webex Adaptive Card is sent to the operator, and the operator clicks "Resolve" or "Dismiss". The difference is what happens after approval.

### The three-stage resolution pipeline

```
Operator clicks "Resolve"
    |
    v
Pre-Remediation Agent ---Wait---> Execution Agent ---Wait---> Confirmation Agent
 (Haiku 4.5 + MCP)                (Haiku 4.5 + MCP)          (Sonnet 4.5 + MCP)
 "verify + plan"                   "execute the plan"         "verify the result"
    |                                  |                           |
    v                                  v                           v
 Structured JSON plan              1 MCP write call            Final report
 (tool, params, reason)            (no deviations)             posted to Webex
```

**Stage 1 — Pre-Remediation Agent** (Claude Haiku 4.5):
- Reads the diagnosis and alert data from the approval path.
- Makes 1-2 MCP read calls to check the current state of the affected device, port, or VLAN.
- Compares the current state to the desired state from the diagnosis.
- Outputs a structured JSON remediation plan with the exact MCP tool name, exact parameters, and the reason for the change.
- If the issue has self-resolved (current state already matches desired state), it sets `needs_remediation` to false and the pipeline stops.
- This agent does NOT execute any changes.

**Stage 2 — Execution Agent** (Claude Haiku 4.5):
- Receives the structured plan from Stage 1.
- If `needs_remediation` is false, it reports "No action needed — issue self-resolved."
- Otherwise, it makes exactly ONE MCP write call using the exact tool and parameters specified in the plan.
- No re-diagnosis. No deviation from the plan. No extra API calls.
- Reports the raw result.

**Stage 3 — Confirmation Agent** (Claude Sonnet 4.5):
- Receives the execution result from Stage 2.
- If execution failed, it reports the failure. It does NOT retry.
- If execution succeeded, it makes ONE MCP read call to verify the change actually took effect.
- Produces a final human-readable report that gets posted to Webex.

### Why Wait nodes matter

You will notice Wait nodes between each stage. These serve two purposes:

1. **Meraki API rate limits.** The Meraki Dashboard API has rate limits. Back-to-back calls from multiple agents in rapid succession can trigger 429 responses. The Wait nodes introduce a deliberate pause between stages.
2. **Resource management.** Each agent execution consumes LLM tokens and n8n resources. The Wait nodes give n8n a chance to complete the current agent's execution cleanly before starting the next one.

### Model selection

Not every stage needs the same model. The strategy here is to match model capability to task complexity:

- **Haiku 4.5** for the Pre-Remediation and Execution agents. These are tightly constrained tasks — verify one thing, execute one thing. A smaller, faster, cheaper model handles them well.
- **Sonnet 4.5** for the Remediation Agent (diagnosis) and Confirmation Agent. These require more reasoning: the diagnosis agent needs to interpret an alert, decide which tools to call, and synthesize findings. The confirmation agent needs to compare before/after state and produce a clear report.

### Why this is a better design

Compared to Agent 3's single Resolution Agent:

- **Focused prompts.** Each agent has one job with explicit constraints ("make exactly ONE MCP call," "do NOT re-diagnose"). This reduces the chance of the model going off-script.
- **Isolated failures.** If the execution fails, the confirmation agent reports it cleanly. You do not lose the diagnosis or the plan.
- **Rate limit friendly.** Spreading API calls across stages with pauses between them avoids hitting Meraki's rate limits.
- **Cost efficient.** The simple stages use Haiku (cheaper and faster). Only the stages that need real reasoning use Sonnet.

## Webex Human-in-the-Loop Setup

Agents 3 and 4 require a Webex webhook so that card button clicks are sent back to n8n. Here is how to set that up.

### Create a Webex Bot

Go to [developer.webex.com](https://developer.webex.com), sign in, and create a new Bot. Save the bot's bearer token — this is what goes in the HTTP Request nodes and the `.env` file.

Add the bot to the Webex room where you want alerts to appear. Note the room ID (you can get it via the Webex API or the developer portal).

### Register the webhook

The Webex webhook tells Webex to send card action events to your n8n instance. Run this once:

```bash
curl -X POST https://webexapis.com/v1/webhooks \
  -H "Authorization: Bearer YOUR_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "n8n Card Response Handler",
    "targetUrl": "https://YOURDOMAIN/webhook/agent-demo",
    "resource": "attachmentActions",
    "event": "created"
  }'
```

Replace `YOUR_BOT_TOKEN` with your bot's bearer token and `YOURDOMAIN` with your public domain.

This registers a webhook that fires whenever someone interacts with an Adaptive Card posted by your bot. The `targetUrl` points to the same `/webhook/agent-demo` endpoint that receives Meraki alerts — the Switch nodes in Agents 3 and 4 know how to tell them apart.

### Configure Meraki alert webhooks

In the Meraki Dashboard, go to **Network-wide -> Alerts** and add an HTTP server with:

- **URL**: `https://YOURDOMAIN/webhook/agent-demo`

Then configure which alert types should trigger the webhook. Start with something easy to test, like a port status change — unplug a cable and watch the workflow fire.

## Wrapping Up

We built four agents that progressively add capability:

1. **Agent 1** — LLM analyzes a raw alert. Fast to set up, good for notification enrichment.
2. **Agent 2** — LLM verifies the alert via MCP before responding. More accurate, fewer guesses.
3. **Agent 3** — Full remediation loop with human approval. The agent diagnoses, the operator decides, the agent fixes.
4. **Agent 4** — Multi-agent pipeline splits remediation into verify, execute, confirm. Better resource utilization, cleaner failure handling, rate-limit friendly.

Each agent builds on the last. You can start with Agent 1 to get value immediately and work your way up to Agent 4 as you get comfortable with the system.

The project repo has everything you need to get started:
[github.com/crwickha/n8n-agent-demo](https://github.com/crwickha/n8n-agent-demo)
