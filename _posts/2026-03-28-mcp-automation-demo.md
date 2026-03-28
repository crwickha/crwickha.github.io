---
layout: post
title: "Automating Branch Network Provisioning with MCP and Cisco Meraki"
date: 2026-03-28
description: "A template-driven MCP server that provisions complete Meraki branch networks — VLANs, SSIDs, firewall rules, VPN — through natural language, with the flexibility to customize any site without breaking automation."
---

What if deploying a new branch office meant describing what you want in plain English and letting an AI agent handle the rest — VLANs, SSIDs, firewall rules, VPN tunnels, all of it?

In this post, we build an MCP server that provisions complete Meraki branch networks from YAML templates. An LLM reads the templates, understands your network design, and executes the provisioning through the Meraki API. No scripts to maintain, no rigid pipelines, no clicking through dashboards. Just a conversation.

This is a working example built around a specific branch network design, but the approach is fully customizable. Your templates, your tiers, your addressing scheme. The MCP server is the framework — what you put in the templates is up to you.

All code, configs, and templates are available in the project repo:
[github.com/crwickha/mcp_automation_demo](https://github.com/crwickha/mcp_automation_demo)

## The Problem with Traditional Network Automation

Most network automation follows one of two patterns: either you write scripts that execute a fixed sequence of API calls, or you build out a full infrastructure-as-code pipeline with Terraform or Ansible. Both work. Both have the same fundamental limitation — they are rigid.

A Python script that provisions a branch network does exactly what you coded it to do. Need to add a VLAN to one specific site? You either modify the script, write a new one, or do it manually. Want to change the firewall rules for a single location because they have a unique compliance requirement? The automation does not handle exceptions well. You end up with a growing collection of scripts, each handling a slightly different variation.

Infrastructure-as-code is better at managing state, but the complexity scales with the number of exceptions. Every site that deviates from the standard template needs its own configuration block. The "automation" becomes a configuration management problem.

The real issue is that traditional automation assumes uniformity. Real networks are not uniform. Branch 114 might need an IoT VLAN. Branch 258 might not. Branch 300 might need a custom firewall rule because they have a legacy device that needs to reach a specific external IP. Traditional automation makes the common case easy and the exceptions painful.

## A Different Approach: Templates + LLM + MCP

Instead of writing imperative automation that handles every case, we define templates that cover the common patterns and let an LLM handle the variations.

The architecture has three components:

**YAML Templates** define your standard branch network designs. Each template has tiers (small, medium, large) with progressively more VLANs, SSIDs, and firewall rules. The templates use variables that get computed from the network name — site number, location, IP addressing — so each deployment is unique but consistent.

**A Network Registry** tracks every network the system has provisioned. It records what was deployed, when, and what tier was used. This serves as both a configuration backup and a lookup table. The LLM can query the registry to understand what exists before making changes.

**An MCP Server** wraps the Meraki Dashboard API and exposes five tools that an LLM can call: list templates, create a network, list managed networks, get a network's configuration, and update a network. The LLM decides which tools to call and in what order based on what you ask it to do.

Here is what happens when you ask the agent to create a new branch:

```
You: "Create a medium branch network for site 114 in Langley"

Agent thinks:
  1. I should check the available templates -> calls auto_list_templates
  2. The "branch" template has a "medium" tier with Corp, Guest, and VoIP
  3. Site 114 -> IP addressing: Corp 10.0.114.0/24, VoIP 10.0.115.0/24
  4. I'll provision this -> calls auto_create_branch_network

Agent executes:
  - Creates network "114 - Langley" in Meraki
  - Enables VLANs on the appliance
  - Creates VLAN 10 (Corp), VLAN 20 (Guest), VLAN 30 (VoIP)
  - Configures SSIDs: Corp-Langley, Guest-Langley, VoIP-Langley
  - Applies firewall rules (guest isolation, VoIP-to-corp allow)
  - Configures site-to-site VPN as spoke
  - Adds the network to the registry

Agent: "Done. Network 114 - Langley is provisioned as a medium branch.
        Here's what was configured: [detailed provisioning log]"
```

The entire provisioning — network creation, VLANs, SSIDs, firewall rules, VPN — happens in a single conversation turn. The agent reads the template, computes the addressing, and makes the API calls.

## The Template System

Templates are YAML files that define what a branch network looks like at each size tier. Here is the structure:

```yaml
product_types:
  - appliance
  - switch
  - wireless

tiers:
  small:
    description: "Small branch - Corp + Guest"
    vlans:
      - id: 10
        name: Corp
        subnet: "10.{site_high}.{site_low}.0/24"
        appliance_ip: "10.{site_high}.{site_low}.1"
        dhcp_handling: "Run a DHCP server"
      - id: 20
        name: Guest
        subnet: "192.168.100.0/24"
        appliance_ip: "192.168.100.1"
        dhcp_handling: "Run a DHCP server"

    ssids:
      - number: 0
        name: "Corp-{location}"
        enabled: true
        auth_mode: psk
        encryption_mode: wpa
        psk: "YourCorpPassword"
        default_vlan_id: 10
      - number: 1
        name: "Guest-{location}"
        enabled: true
        auth_mode: psk
        encryption_mode: wpa
        psk: "YourGuestPassword"
        default_vlan_id: 20

    firewall_rules:
      - comment: "Block guest to corp"
        policy: deny
        protocol: any
        src_cidr: "192.168.100.0/24"
        dest_cidr: "10.{site_high}.{site_low}.0/24"
        dest_port: any

    vpn:
      mode: spoke
      hubs:
        - hub_id: "YOUR_HUB_NETWORK_ID"
          name: "HQ"
          use_default_route: false
      subnets:
        - local_subnet: "10.{site_high}.{site_low}.0/24"
          use_vpn: true
        - local_subnet: "192.168.100.0/24"
          use_vpn: false
```

### Template Variables

Variables are computed automatically from the network name. When you create "114 - Langley":

| Variable | Value | How it is computed |
|----------|-------|--------------------|
| `{name}` | 114 - Langley | Full network name |
| `{location}` | Langley | Everything after the dash |
| `{site_number}` | 114 | Everything before the dash |
| `{site_high}` | 0 | `114 // 256` |
| `{site_low}` | 114 | `114 % 256` |
| `{site_low_voip}` | 115 | `114 % 256 + 1` |
| `{site_low_iot}` | 116 | `114 % 256 + 2` |

The two-octet scheme supports up to 65,279 unique sites. Site 114 gets `10.0.114.0/24`. Site 257 gets `10.1.1.0/24`. Site 1500 gets `10.5.220.0/24`. Each site gets a unique, predictable address space.

### Tiers

The included template has three tiers that build on each other:

**Small** — Corp and Guest. Two VLANs, two SSIDs, one firewall rule (guest isolation), VPN spoke with Corp routed.

**Medium** — Adds VoIP. Three VLANs, three SSIDs, additional firewall rules (guest-to-VoIP block, VoIP-to-Corp allow), VoIP subnet routed over VPN.

**Large** — Adds IoT. Four VLANs, four SSIDs, full inter-VLAN firewall policy (IoT isolated from Corp and VoIP, guest isolated from everything), all production subnets routed over VPN.

This is just the example. You could define tiers for retail stores, warehouses, executive offices, or anything else. You could have a completely different template for data centers with different product types and no wireless. The template system does not care — it renders whatever YAML you give it.

## The Network Registry

Every network the MCP server provisions gets recorded in `network_registry.yaml`:

```yaml
- network_id: L_764486036746147404
  name: 115 - Surrey
  site_number: 115
  location: Surrey
  tier: medium
  tags:
  - medium_branch
  - automated
  template: branch
  product_types:
  - appliance
  - switch
  - wireless
  vlans:
  - id: 10
    name: Corp
    subnet: 10.0.115.0/24
  - id: 20
    name: Guest
    subnet: 192.168.100.0/24
  - id: 30
    name: VoIP
    subnet: 10.0.116.0/24
  ssids:
  - number: 0
    name: Corp-Surrey
  - number: 1
    name: Guest-Surrey
  - number: 2
    name: VoIP-Surrey
  vpn_mode: spoke
  provisioned_at: '2026-03-28T15:41:17.158661+00:00'
  provisioned_status: success
```

The registry serves multiple purposes:

**Configuration backup.** Every managed network's baseline configuration is recorded. You can see exactly what was deployed and when.

**LLM context.** When the agent needs to modify a network, it looks up the registry entry first. It knows the current VLANs, SSIDs, and tier without making extra API calls.

**Fleet visibility.** Ask the agent "list all managed networks" and it returns the full registry — site numbers, tiers, locations, provisioning status.

**Change tracking.** When the agent modifies a network (adds a VLAN, changes a firewall rule), the registry entry is updated with the live state from Meraki, including a `last_modified` timestamp.

## Where This Gets Interesting: Customization Without Rigidity

The template handles the standard deployments. But the real power is in what happens after provisioning.

Say you deployed 50 branches from the medium template. They all have Corp, Guest, and VoIP. Now branch 42 needs a security camera VLAN. With traditional automation, you are modifying scripts or creating a one-off configuration. With the MCP approach, you just ask:

```
You: "Add a VLAN 50 called 'Cameras' with subnet 10.0.43.0/24 to
      site 42. Add a firewall rule to block cameras from reaching
      the corp network. Don't route it over VPN."
```

The agent calls `auto_update_branch_network` with the right parameters. The VLAN is created, the firewall rule is added, and the registry is updated to reflect the new state. Site 42 now has a configuration that deviates from the standard template, and that is fine. The registry tracks what it actually has, not what the template says it should have.

This is the key difference from traditional automation: **the templates define the starting point, not the ceiling.** Any site can be customized after provisioning without breaking the automation for every other site. The registry maintains an accurate record of each site's actual configuration, and the LLM can reason about individual sites when asked.

You could also ask the agent to make bulk changes:

```
You: "For all large branches, add a firewall rule that blocks IoT
      devices from reaching 10.100.0.0/16."
```

The agent lists the managed networks, filters for large tier sites, and applies the change to each one. The registry updates for every affected site.

## Dry Run: Preview Before You Commit

Both the create and update tools support a `dry_run` mode. When enabled, the agent walks through the entire provisioning or modification plan and returns exactly what it would do — without making any API calls.

```
You: "Do a dry run of creating a large branch for site 300 in Victoria"
```

The agent returns the full plan: network name, VLANs with subnets, SSIDs with names, firewall rules, VPN configuration. You review it, and if it looks right, you run it for real.

This is especially useful when customizing existing sites. Before adding a VLAN or changing a firewall rule, you can preview the change and confirm it matches your intent.

## Integration Points: ServiceNow, IPAM, and Beyond

The MCP server handles the Meraki provisioning, but it does not exist in isolation. In a production environment, network provisioning is part of a larger workflow.

**ServiceNow or ticketing integration.** A new branch deployment starts with a service request. An n8n workflow could watch for approved tickets, extract the site details (number, location, tier), and trigger the MCP server to provision the network. The provisioning log goes back to the ticket as a work note. No human touches the Meraki dashboard.

**IPAM integration.** Instead of computing IP addressing from a static formula, the MCP server could query an IPAM system (NetBox, Infoblox, phpIPAM) via another MCP tool to allocate the next available subnet. The template variables would be populated from IPAM rather than from arithmetic. This ensures your IP addressing stays in sync with your source of truth.

**CMDB updates.** After provisioning, the agent could update your CMDB with the new network's details — device serials, subnet assignments, VLAN IDs, VPN spoke configuration. MCP makes this straightforward because each integration is just another tool the LLM can call in sequence.

**Approval workflows.** For environments that require change control, you can add a human-in-the-loop step between the dry run and the actual provisioning. The agent generates the plan, posts it to Slack or Webex for approval, and only executes after an operator confirms. This is the same pattern we use for remediation agents in our [previous post](https://blog.craigsandbacon.com/blog/n8n-meraki-ai-agents/).

The point is that MCP tools are composable. The automation MCP server handles Meraki. An IPAM MCP server handles address allocation. A ServiceNow MCP server handles ticket updates. Wire them together in an n8n workflow, and the LLM orchestrates the entire process — from ticket to provisioned network to updated CMDB — in a single conversation.

## The MCP Server

The server is a Python application that implements the Model Context Protocol (MCP) specification. It runs as a Docker container and exposes five tools over JSON-RPC.

### Tools

| Tool | Description |
|------|-------------|
| `auto_list_templates` | List all available templates with tier descriptions, VLAN counts, and SSID counts |
| `auto_create_branch_network` | Provision a complete branch network from a template (supports dry run) |
| `auto_list_networks` | List all networks in the master registry |
| `auto_get_network_config` | Retrieve full configuration of a managed network (registry + live Meraki state) |
| `auto_update_branch_network` | Modify or delete a managed network — add/remove VLANs, SSIDs, firewall rules, tags (supports dry run) |

### How It Works

The server loads YAML templates at startup and initializes the Meraki Python SDK. When an LLM calls a tool:

1. The MCP client (n8n, Claude Desktop, or any MCP-compatible client) sends a JSON-RPC request to the server.
2. The server validates the request and routes it to the appropriate tool handler.
3. The tool handler reads the template, computes variables, and makes the necessary Meraki API calls.
4. Results are returned as structured JSON — provisioning logs, configuration snapshots, or error details.
5. The network registry is updated to reflect the current state.

The server also exposes a `/health` endpoint for monitoring:

```json
{
  "status": "healthy",
  "service": "automation-mcp",
  "version": "1.0.0",
  "meraki_connected": true,
  "templates_loaded": ["branch"],
  "registry_count": 12
}
```

## Docker Compose Setup

Clone the project and set up your environment:

```bash
git clone https://github.com/crwickha/mcp_automation_demo.git
cd mcp_automation_demo
cp .env.example .env
```

Edit the `.env` file with your values:

```
MERAKI_API_KEY=your-meraki-api-key-here
MERAKI_ORG_ID=your-org-id-here
CLOUDFLARE_TUNNEL_TOKEN=your-cloudflare-tunnel-token-here
```

The `docker-compose.yml` defines two services:

```yaml
services:
  automation-mcp:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: automation-mcp
    restart: always
    environment:
      - MERAKI_API_KEY=${MERAKI_API_KEY}
      - MERAKI_ORG_ID=${MERAKI_ORG_ID:-}
    volumes:
      - ./templates:/app/templates:ro
      - ./network_registry.yaml:/app/data/network_registry.yaml
    networks:
      - mcp_network

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: always
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    networks:
      - mcp_network

networks:
  mcp_network:
    driver: bridge
```

A few things to note:

* **Templates are mounted read-only.** The container reads templates from `./templates/` but cannot modify them. To change a template, edit the YAML file on the host and restart the container.
* **The network registry is mounted read-write.** This file is the persistent state of all managed networks. Back it up.
* **No ports are exposed.** The Cloudflare Tunnel handles external access. The MCP server is reachable through the tunnel but has no ports open on the host.
* **No n8n in this compose file.** This is just the MCP server and tunnel. You can connect to it from n8n, Claude Desktop, or any MCP-compatible client by pointing to the tunnel URL.

Configure your Cloudflare Tunnel to route your chosen hostname to `http://automation-mcp:3003`. Then bring it up:

```bash
docker compose up -d
```

Verify the server is running:

```bash
curl https://your-tunnel-hostname/health
```

## Connecting an MCP Client

The MCP server speaks standard JSON-RPC over HTTP. Any MCP-compatible client can connect to it.

**From n8n:** Add an MCP Client tool node and set the endpoint to `http://automation-mcp:3003/mcp` (if n8n is on the same Docker network) or `https://your-tunnel-hostname/mcp` (if connecting externally).

**From Claude Desktop or Claude Code:** Add the server to your MCP configuration with the tunnel URL as the endpoint.

Once connected, the client automatically discovers the five available tools. The LLM can see the tool descriptions and input schemas, and it decides when and how to use them based on your instructions.

## Customizing for Your Environment

This example is built around a specific branch network design — three tiers, a particular VLAN scheme, PSK wireless. Your environment will be different. Here is what to change:

**Templates.** Edit `templates/branch.yaml` or create new template files. Change the VLANs, SSIDs, firewall rules, and VPN configuration to match your standards. Add new tiers. Remove tiers you do not need. The server loads all YAML files from the templates directory at startup.

**Addressing.** The two-octet scheme (`10.{site_high}.{site_low}.0/24`) is computed in the server code. If your addressing scheme is different, modify the `compute_site_octets()` function or replace it entirely. If you use an IPAM system, you could replace the static computation with an API call.

**VPN hubs.** The template has a hardcoded hub network ID. Replace it with your actual hub network ID, or make it a variable if you have multiple hubs.

**Wireless authentication.** The example uses PSK. If you use 802.1X, RADIUS, or open authentication, change the SSID configuration in the template. The MCP server passes whatever you define in the template through to the Meraki API.

**Product types.** The example deploys appliance, switch, and wireless. If some branches only have an appliance, create a template with just that product type.

**Additional tools.** The server has five tools. If you need more — say, a tool that configures switch port profiles, or one that sets up RADIUS servers — you can add them by following the same pattern in `server.py`. Define the tool schema and add the execution logic.

## Wrapping Up

Traditional network automation makes you choose between standardization and flexibility. Templates give you consistent deployments. Scripts give you repeatable execution. But the moment a single site needs something different, the rigidity becomes a problem.

The MCP approach gives you both. Templates handle the 90% case — standard branch deployments that follow your design. The LLM handles the 10% — site-specific customizations, one-off changes, bulk modifications across a subset of sites. The network registry keeps track of what each site actually has, not just what it was supposed to have.

This is an example implementation. The template, the addressing scheme, the tier definitions — all of it is meant to be replaced with your own design. The value is in the pattern: define your standards as templates, expose your network API as MCP tools, and let an LLM bridge the gap between what you want and the API calls required to get there.

And because MCP tools are composable, this does not have to stop at Meraki. Add an IPAM tool for address allocation. Add a ServiceNow tool for ticket management. Add a CMDB tool for asset tracking. The LLM orchestrates across all of them. What used to be a multi-team, multi-tool provisioning workflow becomes a conversation.

The code is on GitHub: [github.com/crwickha/mcp_automation_demo](https://github.com/crwickha/mcp_automation_demo). Clone it, swap in your templates, and start provisioning.
