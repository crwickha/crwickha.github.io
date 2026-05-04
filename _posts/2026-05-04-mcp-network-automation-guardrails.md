---
layout: post
title: "Guardrails for AI and MCP in Network Automation: A Crawl, Walk, Run Guide"
date: 2026-05-04
description: "Practical lessons on running AI agents and MCP servers against real networks -- avoiding hallucinations, rate limits, tool sprawl, and other failure modes that bite once you move past the demo."
---

Letting an LLM touch your network is a different conversation than letting it touch your filesystem. A bad shell command deletes a file you can restore from backup. A bad API call against a production firewall locks 400 people out of a building.

I have spent the last several months building MCP servers for Meraki, Catalyst, NetFlow, and a handful of internal tools. Some of it has gone well. Some of it has not. This post is the running list of guardrails I wish I had set up earlier -- the things that make the difference between a useful agent and a confidently wrong one.

It is organized roughly as a progression. Start in read-only mode. Earn the right to make changes. Add scope only when the previous scope is boring. Crawl, walk, run.

## Why Guardrails Matter More Here

Two things make network automation a sharper edge than most other AI use cases.

First, the actions are not local. When Claude edits a file in your repo you can review the diff, undo it, push or not push. When an agent calls `update_firewall_rules` the change is live the moment the API returns 200. There is no staging step. The blast radius is whatever you scoped the API key to.

Second, the data is structured but ambiguous. A Meraki organization has many networks. A network has many devices. A device has many interfaces. The model has to keep all of that straight while juggling IDs that look almost identical. `L_123456789012345678` and `N_123456789012345678` are not the same thing, and the model will absolutely confuse them at 2am if you let it.

Guardrails are not about distrusting the model. They are about acknowledging that you are operating inside a system where small mistakes compound quickly.

## Crawl: Read-Only Everything

The first guardrail is the easiest one to enforce, and the one most people skip past on their way to building something cool.

**Start with an MCP server that only exposes read operations.** No `update_*`, no `create_*`, no `delete_*`, no `reboot_*`. Just `get_*` and `list_*`. You want the agent to be able to look at the network, not change it.

There are two reasons this matters more than it sounds:

1. You learn what the agent is good at. Most of the value in an AI assistant for network ops is the diagnostic loop -- "client X cannot reach server Y, what is wrong?" That entire workflow is read-only. If you can answer that question reliably with read tools, you have already captured 70% of the value before you take on any of the risk.

2. You learn how the model handles your data shape. Watch the agent paginate through `list_networks` for the first time and you will immediately see whether your API key is scoped too broadly. Watch it confuse a network ID with an organization ID and you will know to tighten your tool descriptions before adding write operations.

Run in read-only for at least a few weeks. Use it for real work. If you find yourself reaching for a write tool, write it down -- but do not build it yet.

## Walk: Write Operations With a Net

When you do add write tools, do it deliberately. A few patterns that have saved me:

**Confirm before commit.** The MCP tool can take the inputs and return a "this is what I am about to do, do you want me to proceed?" response on the first call. The agent has to call the tool a second time with a confirmation flag to actually execute. This sounds annoying. It is. It is also the difference between "the agent rebooted the wrong switch" and "the agent told me it was about to reboot the wrong switch and I caught it."

**Dry-run mode by default.** Every write tool takes a `dry_run: bool` parameter that defaults to `True`. The agent has to explicitly pass `False` to make a change. This puts the burden of the destructive action on the prompt, where you can review it.

**Scope the API key, not the tool.** If your write tools only operate on a single test network, you cannot accidentally hose production no matter what the agent does. Cisco Meraki API keys can be scoped per organization. Use that. Run your experimental agent against an org that contains exactly one network with exactly one device.

**Action batches over individual calls.** When the platform supports it (Meraki does), prefer creating an action batch in synchronous mode over making individual calls. The whole batch either applies or it does not. Half-applied changes are the worst kind of failure.

A pattern I have settled on for write tools:

```
+----------------------+    +----------------------+    +----------------------+
| Read tools always    |    | Write tools require  |    | Destructive tools    |
| available            |    | dry_run=False        |    | require human        |
|                      |    |                      |    | approval webhook     |
| get_*, list_*        |    | update_*, create_*   |    | delete_*, reboot_*   |
+----------------------+    +----------------------+    +----------------------+
```

Three tiers, three different levels of friction. Read is free. Modify is gated by a parameter the agent has to set on purpose. Destructive is gated by a human pressing a button somewhere -- in my case, an Adaptive Card in Webex.

## Run: Trust, But Audit

The last stage is when the agent operates with less direct supervision. Maybe it is responding to alerts. Maybe it is running a scheduled drift-check. Whatever the workflow, by the time you get here you should have:

- A log of every tool call the agent made, with arguments, in a place you can grep.
- A diff of every write operation, captured before and after.
- A way to roll back -- either via `save_config` snapshots, infrastructure-as-code state, or the platform's own version history.

If you cannot answer "what did the agent do at 3:47am last Tuesday" in under two minutes, you are not ready for this stage. Go back to walk.

## Hallucinations: Where They Actually Happen

People worry about hallucinations in the wrong places. The model is not going to invent a brand new firewall rule and apply it. What it will do is much subtler.

**It will confuse identifiers.** Meraki has organization IDs, network IDs, device serials, MAC addresses, and client IDs. They all look like opaque strings. The model has no inherent way to know which is which. If you have a tool that takes `network_id` and the model has been working with `organization_id` for the last five turns, it will sometimes pass the org ID into the network slot. The API call will fail, but you do not want to rely on the API to be your validation layer.

The fix is in the tool description and parameter naming. Be explicit:

```python
@mcp.tool()
def get_network_clients(network_id: str) -> list:
    """List clients on a Meraki network.

    network_id: The Meraki network ID. Starts with 'L_' or 'N_'.
    Do not pass an organization ID (which starts with a number)
    or a device serial (format: XXXX-XXXX-XXXX).
    """
```

That extra paragraph is not boilerplate. It is the model's only signal about what shape the argument should take.

**It will fabricate plausible-sounding data when the tool returns nothing.** If `list_devices` returns an empty array because you queried the wrong network, a less careful model will sometimes "remember" devices from earlier in the conversation and present them as the current result. Always have your tools return errors loudly. An empty list and a missing network should not look the same to the model.

**It will summarize, and the summary will lose detail.** When a tool returns 50 events and the model decides to summarize them for you, important outliers can get dropped. If the data matters, ask for the raw output. Or build a tool that already filters server-side rather than asking the model to filter mentally.

**It will bridge gaps with assumptions.** If you ask "is the Langley site healthy?" and the model has tools for `get_device_status` but not `get_site_status`, it will sometimes invent a coherent-sounding health summary by reasoning over the device statuses. That is often what you want -- but make sure you know when it is reasoning versus reporting.

## Rate Limits: The Quiet Killer

Rate limits are the failure mode that hits you in production and not in development. In development, you have one tab open and you are running one query. In production, the agent can decide to call `list_devices` across forty networks in parallel and burn through your API quota in under a minute.

A few things that have helped:

**Cache aggressively in the MCP server.** Most read endpoints return data that does not change minute to minute. List of organizations? Cache for an hour. List of networks in an org? Cache for ten minutes. Device inventory? Cache for five. The agent does not know it is hitting a cache. It just sees a fast response.

**Use bulk endpoints where they exist.** Meraki has both `getNetworkDevices` (one network at a time) and `getOrganizationDevices` (all networks, paginated). If the agent wants to look across the org, surface the bulk version as a tool. Otherwise it will loop, and looping is what eats your quota.

**Limit fan-out at the tool level, not the prompt level.** Telling the model "do not check more than five networks at once" works about half the time. Returning an error from your tool when it is called more than N times in a window works every time.

**Watch the per-key burst limit, not just the per-second.** Meraki's burst protection will start returning 429s well before you hit the steady-state limit if the agent fires ten requests in two seconds. Add jittered backoff inside the MCP server so the model does not have to think about it.

## Choosing the Right Model

Not every task needs the biggest model. Picking right saves money and, more importantly, latency -- which makes the agent feel responsive instead of laggy.

The rough framing I use:

| Task | Model |
|---|---|
| Long, multi-step reasoning across many tools | Claude Opus |
| Day-to-day diagnostic queries, summaries, well-defined workflows | Claude Sonnet |
| Single-shot classification, alert triage, "is this a real problem yes/no" | Claude Haiku |

The biggest mistake is using Opus for everything because it is the smartest. It is also the slowest and the most expensive, and on simple tasks the speed difference is what your users will notice. For a "categorize this Meraki alert as critical/warning/info" workflow, Haiku is fine and finishes in a fraction of a second. Save Opus for the cases where you genuinely need the agent to think across ten tool calls.

A useful test: if you can describe the task in two sentences and the answer fits in one paragraph, try the smaller model first. If it gets it wrong, move up. Do not start at the top.

## Tool Count: Less Is More

Every tool you give the model is something it has to consider every time it decides what to do. Past a certain count, this stops being useful and starts being noise.

I have run an MCP server with 60 tools. The model can technically handle it -- the context window is fine. But the *quality* of tool selection drops. The model spends more time deliberating, makes more wrong-tool errors, and sometimes invents tool names that do not exist because it is confusing two similar ones.

The number that has worked for me is somewhere between 8 and 20 tools per server, with a hard ceiling around 30. If you have more than that, you probably need to:

**Split into multiple MCP servers by domain.** Instead of one giant `meraki-mcp` server, have `meraki-read-mcp`, `meraki-config-mcp`, `meraki-troubleshoot-mcp`. The user enables only the ones they need for the current task. Even better, the model only sees the tools that are loaded.

**Combine narrow tools into broader ones.** Five tools that each return one piece of device info can usually become one tool that takes a `fields: list[str]` argument. The model's job gets easier; the surface area shrinks.

**Remove tools that are never called.** Watch the logs. If a tool has not been called in a month, it is probably not earning its slot in the context window. Cut it.

## Naming: Boring Is Good

Tool names and server names are part of the prompt. The model reads them. Bad names are bad prompts.

**Server names should describe the system, not the implementation.** `meraki-mcp` is good. `craig-internal-server-v3` is bad. The model has to reason about which server to use for what -- give it words it can connect to the user's question.

**Tool names should be `verb_noun`, lowercase, with the verb that the user would say.** `get_arp_table` is good. `arpTableGetter` is bad. `network_arp` is bad. Match the language people actually use.

**Be consistent across tools in the same server.** If one tool uses `network_id` and another uses `networkId` and a third uses `net_id`, the model will get them wrong eventually. Pick a convention and stick to it.

**Prefix related tools with a common stem.** All my Meraki read tools start with `get_` or `list_`. All my write tools start with `update_`, `create_`, `delete_`, or `configure_`. This gives the model a strong signal about what kind of action it is taking. It also makes log analysis trivial -- I can grep for `update_*` to find every write operation an agent made.

A naming pattern that has worked well:

```
+--------------------+--------------------------------------------+
| Prefix             | Meaning                                    |
+--------------------+--------------------------------------------+
| get_               | Returns one record                         |
| list_              | Returns multiple records                   |
| create_            | Adds a new resource                        |
| update_            | Modifies an existing resource              |
| delete_            | Removes a resource                         |
| configure_         | Replaces a resource's full config          |
| run_               | Executes an active operation (test, ping)  |
+--------------------+--------------------------------------------+
```

The model picks up the pattern fast. After three or four conversations using the server it stops mis-selecting tools entirely.

## Descriptions: The Prompt You Did Not Write

Most people write tool descriptions like docstrings -- a sentence about what the tool does, maybe a parameter list. That is leaving capability on the table.

The description is the place to put:

- What the tool returns (shape, not just type).
- When the model should call this tool versus a similar one.
- What kinds of inputs are valid and which are common mistakes.
- How long the call typically takes (the model can use this to decide whether to fan out).
- Side effects, if any.

Compare these two:

```python
@mcp.tool()
def get_interfaces(device_serial: str) -> list:
    """Get device interfaces."""
```

```python
@mcp.tool()
def get_interfaces(device_serial: str) -> list:
    """Return all physical and logical interfaces on a Catalyst device.

    Use this when you need interface state, speed, duplex, or counters.
    For just the up/down status of access ports, prefer
    list_switch_ports which is faster.

    device_serial: The device's serial number, format XXXX-XXXX-XXXX.
    Not the MAC address. Not the hostname.

    Returns: list of dicts with keys: name, status, admin_status,
    speed, duplex, vlan, description.

    Typical latency: 200-500ms per call.
    """
```

The second one will be selected correctly far more often. It also tells you something useful: if I add `list_switch_ports` later, I should update this description so the model knows to prefer it.

## A Few Final Things

Some loose lessons that did not fit cleanly anywhere else:

**Log every tool call, including failures.** The arguments, the timestamp, the response. If the agent does something surprising you want to be able to reconstruct what it was looking at when it decided. I push these to a SQLite file that I can query later.

**Have a kill switch.** A way to immediately disable the MCP server without restarting Claude Desktop or the agent runtime. For local servers this is a flag file the server checks at the top of every tool call. For remote servers it is the ability to shut down the container.

**Test with realistic data.** A network with three devices does not exercise the failure modes of a network with three hundred. Pagination bugs, rate-limit interactions, and ID confusion all only show up at scale.

**Do not give the agent credentials it does not need.** The MCP server should hold the credentials. The agent should not see them. If the agent ever has access to your raw API key, you are one prompt injection away from a bad day.

**Write the runbook for what happens when the agent is wrong.** Not if. When. What is the procedure to undo a misapplied firewall rule? Who gets paged? How do you find out which agent action caused the problem? Build that path before you need it.

## Where to Go From Here

The honest summary is that AI in network operations is mostly a normal engineering problem with one extra wrinkle: the system on the other end of your code is non-deterministic. You handle that the same way you handle any other non-deterministic system -- by adding boundaries, retries, validation, and observability around it.

If you are starting out, build the read-only server first. Use it for a month. The discipline of not having write tools forces you to think harder about what the agent is actually adding, and that thinking is what tells you which write tools are worth building.

If you are already running write operations, audit your logs. Find one surprising tool call from the last week and trace through why the agent picked that tool with those arguments. Whatever you find -- a vague description, an ambiguous parameter name, a missing dry-run flag -- fix that, then go look for the next one.

The agents get better as the guardrails get better. The work is in the guardrails.
