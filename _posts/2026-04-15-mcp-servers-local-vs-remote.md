---
layout: post
title: "MCP Servers in Claude Desktop: Local vs. Remote and Building Your First One"
date: 2026-04-15
description: "A practical guide to local and remote MCP servers in Claude Desktop -- what they are, how they differ, and how to build and connect your first local MCP server from scratch."
---

Claude Desktop can connect to MCP servers in two fundamentally different ways: remotely via a URL that Anthropic's infrastructure connects to on your behalf, or locally as a process running on your machine. The distinction matters more than you might think -- it affects what the server can access, where it works, and how you set it up.

This post breaks down both approaches, walks through setting up each type, and ends with building a local MCP server from scratch -- first a minimal hello-world example, then a practical one that talks to the Meraki Dashboard API.

## What Is MCP?

The [Model Context Protocol](https://modelcontextprotocol.io) is a standard that lets AI applications like Claude discover and use external tools. Instead of hardcoding integrations, Claude can connect to any MCP server and dynamically learn what tools it offers -- what they do, what parameters they accept, and how to call them.

Think of it like a USB port for AI. Plug in an MCP server, and Claude gains new capabilities -- reading files, querying APIs, controlling devices -- whatever the server exposes.

## The Two Types: Connectors (Remote) vs. Local MCP Servers

### Connectors (Remote MCP Servers)

Connectors connect Claude to remote MCP servers via a URL. The connection is brokered through Anthropic's infrastructure -- even in Claude Desktop, it is Anthropic's servers reaching out to the remote MCP server, not your local machine. But the MCP server itself can run anywhere: a cloud provider, a container on a server in your data center, a Raspberry Pi in your closet -- as long as it is reachable via URL. You could run one locally and expose it with a Cloudflare Tunnel, and it would work as a connector.

Key traits:

- **Works everywhere.** Connectors sync across all your Claude apps. Install one on Claude Desktop and it is automatically available on claude.ai and other clients too.
- **Authentication via OAuth.** When you add a connector, you go through a standard OAuth flow to sign in and grant permissions.
- **Two flavors.** Built-in connectors come from Anthropic's catalog (Gmail, Google Calendar, Google Drive). Custom connectors let you supply your own remote MCP server URL pointing to a server you host yourself.
- **No direct local access.** Because the connection is established through Anthropic's infrastructure, the server cannot reach your local filesystem or network directly. If you need a connector to access local resources, you would need to expose those resources to the network where the server runs.

### Local MCP Servers

Local MCP servers are programs that run directly on your computer. Claude Desktop launches them as subprocesses and communicates with them over stdio.

Key traits:

- **Desktop only.** Local MCP servers are not available on claude.ai -- only in Claude Desktop.
- **Full local access.** They can read local files, run local processes, query local APIs, and interact with anything your machine can reach -- including devices on your LAN.
- **Not synced.** They stay on the machine where they are configured. Moving to a different computer means reconfiguring.
- **You control the code.** You write the server, you decide what tools it exposes, you manage its dependencies.

### Desktop Extensions -- The Middle Ground

Desktop extensions are a newer addition. They package local MCP servers into installable `.mcpb` files that you can add with a single click instead of editing JSON config files. Under the hood, they are still local MCP servers -- they run on your machine and have the same capabilities and limitations.

All desktop extensions are local MCP servers, but not all local MCP servers are extensions.

## Quick Comparison

| | Connectors (Remote) | Local MCP Servers |
|---|---|---|
| Where it runs | Anywhere reachable via URL (connection brokered by Anthropic) | Your machine |
| Available on claude.ai | Yes | No |
| Available in Claude Desktop | Yes | Yes |
| Synced across devices | Yes | No |
| Setup method | UI / OAuth flow | Edit JSON config or install .mcpb |
| Local file/resource access | No | Yes |
| Advanced Research support | Yes | No |

The short version: connectors are remote servers reachable via URL, with the connection brokered through Anthropic's infrastructure, and they work everywhere. Local MCP servers run on your machine and are more powerful for local resource access but only work in Claude Desktop.

## Adding a Remote Connector

Adding a remote connector is straightforward and happens entirely in the UI.

1. Open Claude Desktop (or claude.ai).
2. Go to **Settings** and find the **Connectors** section.
3. Browse the built-in catalog or click **Add Custom Connector** to supply your own remote MCP server URL.
4. Complete the OAuth sign-in flow when prompted.
5. The connector appears in your tool list and syncs to all your Claude apps automatically.

That is it. No config files, no code, no process management. The trade-off is that the connection goes through Anthropic's infrastructure, so the remote server cannot directly access your local machine or network.

## Setting Up a Local MCP Server

Local MCP servers are configured in a JSON file that Claude Desktop reads on startup:

- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Linux:** `~/.config/Claude/claude_desktop_config.json`

The file contains a `mcpServers` object where each key is a server name and each value tells Claude Desktop how to launch it.

Here is the general structure:

```json
{
  "mcpServers": {
    "server-name": {
      "command": "/path/to/interpreter",
      "args": ["/path/to/your/script.py"],
      "env": {
        "SOME_API_KEY": "your_key_here"
      }
    }
  }
}
```

Three fields:

- `command` -- the executable to run (Python interpreter, Node.js, etc.)
- `args` -- arguments passed to the command (typically the path to your server script)
- `env` -- environment variables the server needs (API keys, config values)

**Important:** Claude Desktop does not inherit your shell's PATH or virtual environment. Use full absolute paths for both the interpreter and the script. Instead of `python`, use something like `/usr/bin/python3` or `/Users/you/myenv/bin/python`.

## Build Your First Local MCP Server: Hello World

Let's start with the simplest possible MCP server -- one tool that returns a greeting.

### Prerequisites

You need Python 3.10+ and the `mcp` package:

```bash
pip install mcp
```

If you are using a virtual environment (recommended), note the full path to the Python interpreter inside it. You will need it for the config file.

### The Server Script

Create a file called `hello_mcp.py` anywhere on your machine:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("HelloWorld")

@mcp.tool()
def hello(name: str) -> str:
    """Say hello to someone."""
    return f"Hello, {name}! This response came from your local MCP server."

@mcp.tool()
def add_numbers(a: float, b: float) -> float:
    """Add two numbers together."""
    return a + b

mcp.run()
```

That is the entire server. `FastMCP` handles the protocol negotiation, tool discovery, and communication with Claude Desktop. You just define functions and decorate them with `@mcp.tool()`.

A few things to notice:

- The docstring becomes the tool description that Claude sees. Write clear descriptions -- they help Claude understand when and how to use each tool.
- Type hints on the parameters become the tool's input schema. Claude uses these to know what arguments to pass.
- The return value gets sent back to Claude as the tool result.

### Connect It to Claude Desktop

Open your `claude_desktop_config.json` and add:

```json
{
  "mcpServers": {
    "hello": {
      "command": "/usr/bin/python3",
      "args": ["/full/path/to/hello_mcp.py"]
    }
  }
}
```

Replace `/usr/bin/python3` with the actual path to your Python interpreter. If you installed the `mcp` package in a virtual environment, use that environment's Python path (e.g., `/home/you/mcp-env/bin/python`).

Replace `/full/path/to/hello_mcp.py` with the actual path to your script.

### Test It

1. Restart Claude Desktop (or quit and reopen it).
2. Go to **Settings > Developer**. You should see "hello" listed with a green status indicator.
3. Start a new conversation and ask: *"Say hello to Craig"*
4. Claude will call your `hello` tool and return the greeting from your local server.

If the status indicator is red or the server does not appear, check the logs in Settings > Developer for error messages. Common issues:

- Wrong Python path -- the interpreter cannot be found.
- Missing `mcp` package -- it is not installed in the Python environment you pointed to.
- JSON syntax error in the config file -- use a JSON validator if unsure.

## A Practical Example: Meraki Dashboard MCP Server

The hello world server proves the pattern works. Now let's build something useful -- a local MCP server that queries the Cisco Meraki Dashboard API.

### The Server Script

Create a file called `meraki_mcp.py`:

```python
from mcp.server.fastmcp import FastMCP
import meraki

mcp = FastMCP("Meraki")

dashboard = meraki.DashboardAPI(suppress_logging=True)

@mcp.tool()
def list_organizations() -> list:
    """List all Meraki organizations the API key has access to."""
    return dashboard.organizations.getOrganizations()

@mcp.tool()
def list_networks(organization_id: str) -> list:
    """List all networks in a Meraki organization."""
    return dashboard.organizations.getOrganizationNetworks(organization_id)

@mcp.tool()
def get_network_health(network_id: str) -> list:
    """Get health alerts for a Meraki network."""
    return dashboard.networks.getNetworkHealthAlerts(network_id)

@mcp.tool()
def list_devices(network_id: str) -> list:
    """List all devices in a Meraki network."""
    return dashboard.networks.getNetworkDevices(network_id)

mcp.run()
```

### Install Dependencies

```bash
pip install mcp meraki
```

### Configure Claude Desktop

Add this to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "meraki": {
      "command": "/usr/bin/python3",
      "args": ["/full/path/to/meraki_mcp.py"],
      "env": {
        "MERAKI_DASHBOARD_API_KEY": "your_api_key_here"
      }
    }
  }
}
```

The `env` block is how you pass secrets to your MCP server. The Meraki Python SDK reads `MERAKI_DASHBOARD_API_KEY` from the environment automatically, so you do not need to hardcode it in your script. This keeps your API key out of your source code.

**Where to get a Meraki API key:** Log in to the [Meraki Dashboard](https://dashboard.meraki.com), go to **Organization > Settings**, scroll to the **Dashboard API access** section, and generate an API key. Treat it like a password.

### How It Works

When Claude Desktop starts:

1. It reads `claude_desktop_config.json` and finds the "meraki" server entry.
2. It spawns `python /full/path/to/meraki_mcp.py` as a subprocess, passing the environment variables from the `env` block.
3. The server starts up, initializes the Meraki SDK (which picks up the API key from the environment), and announces its available tools over stdio.
4. Claude Desktop discovers the tools and makes them available in conversations.

You do not need to manually run the script. Claude Desktop handles the full lifecycle -- starting, monitoring, and restarting the server if it crashes.

### Using It

Restart Claude Desktop, verify the server shows green in Settings > Developer, then try:

- *"List my Meraki organizations"*
- *"Show me all networks in org X"*
- *"Are there any health alerts on network Y?"*
- *"What devices are in the Seattle office network?"*

Claude calls the appropriate tool, gets the response from the Meraki API via your local server, and presents the results conversationally. Since the server runs on your machine, it can reach your Meraki dashboard using your API key -- something a remote connector could not do with a locally-scoped API key.

## Tips for Building Local MCP Servers

**Use full paths everywhere.** Claude Desktop does not source your shell profile. If your script imports packages from a virtual environment, point `command` at that environment's Python binary.

**Keep servers focused.** A server that does one thing well is easier to debug than one that does everything. You can configure multiple MCP servers in the same config file.

**Use the `env` block for secrets.** Never hardcode API keys, tokens, or passwords in your server scripts. The `env` block in the config file injects them as environment variables at runtime.

**Check the Developer panel.** Settings > Developer in Claude Desktop shows connection status and logs for each configured server. It is your first stop when something is not working.

**Write good docstrings.** Claude reads the docstrings and type hints to understand what your tools do and how to call them. Vague descriptions lead to vague tool usage.

## When to Use Which

Use **remote connectors** when:

- You want the integration available on claude.ai and Claude Desktop.
- The service is cloud-hosted and supports OAuth (Google Workspace, SaaS tools).
- You do not need access to local resources.

Use **local MCP servers** when:

- You need to access local files, local APIs, or devices on your network.
- You want full control over what the server does and how it works.
- The API requires credentials that should stay on your machine.
- You are prototyping or building custom tooling.

Both can coexist. You might use a Google Calendar connector for scheduling while running a local MCP server for querying your Meraki network. Claude sees all the tools from all connected servers and can use them together in the same conversation.

## What's Next

This post covered the fundamentals -- understanding the two types of MCP servers and building simple local ones. From here, you can:

- Add more tools to your server. Any Python function can become an MCP tool.
- Build servers in other languages. The MCP SDK has implementations in TypeScript, Go, and others.
- Package your server as a desktop extension (`.mcpb`) for easier distribution.
- Deploy a remote MCP server with OAuth for team-wide access, and add it as a custom connector.

For a more advanced example of what local MCP servers can do, check out the [branch network provisioning post](https://blog.craigsandbacon.com/blog/mcp-automation-demo/) where we built an MCP server that provisions entire Meraki branch networks from YAML templates through natural language.

The MCP ecosystem is growing fast. The protocol is open, the tooling is maturing, and the barrier to entry is a Python script and a JSON config file. If Claude cannot do something you need today, you can build the tool yourself and hand it over.
