# Resolve MCP Plugins & Skills

Plugins and skills for managing your [Resolve Pay](https://resolvepay.com) account from AI assistants, powered by the [Resolve MCP](https://docs.resolvepay.com/guides/mcp) server (`https://mcp.resolvepay.com/mcp`).

Everything lives on `main`:

| Directory | For | Contents |
|---|---|---|
| [`plugins/`](plugins/) | Claude Code (plugin marketplace) | Plugins bundling the MCP connector and skills |
| [`openai/`](openai/) | ChatGPT & Codex | Skill folders using ChatGPT's file-attachment flow |

## What's included

| Plugin / skill | What it does |
|---|---|
| **resolve-mcp** | Base: the Resolve MCP connector plus the core skill — customers, invoices, shipments, payments, credit notes. Install this first. |
| **resolve-invoice-management** | Deep-dive invoice workflows: lifecycle rules, PDF handling, advances, credit notes, shipments. |
| **resolve-customer-management** | Customer onboarding and credit: create customers, credit checks, enrollment, credit lines. |

The two file-handling variants exist because ChatGPT attaches files directly to the conversation, while Claude and other MCP clients use Resolve's temp-file upload flow. Details: [MCP Tool Reference](https://docs.resolvepay.com/guides/mcp-tools).

## Install

### Claude Code

```
/plugin marketplace add resolvepay/resolve-mcp-plugin
/plugin install resolve-mcp@resolve
```

Optionally add the domain plugins:

```
/plugin install resolve-invoice-management@resolve
/plugin install resolve-customer-management@resolve
```

The base plugin includes the MCP connector — on first use, Claude Code opens the Resolve sign-in (your dashboard login).

### Claude (web / desktop)

Download the skill zips from the [docs site](https://docs.resolvepay.com/guides/mcp-skill) and upload under **Settings > Capabilities > Skills**, then add the Resolve connector (**Settings > Connectors**, server URL `https://mcp.resolvepay.com/mcp`).

### ChatGPT

1. Add the Resolve connector (Settings > Apps & Connectors, server URL `https://mcp.resolvepay.com/mcp`).
2. Go to **Skills > New skill > Upload from your computer** and upload the skill(s) from [`openai/`](openai/) (zip a folder, or download the prebuilt zips from the [docs site](https://docs.resolvepay.com/guides/mcp-skill)).

> Skills in ChatGPT are in beta on Business, Enterprise, and Edu plans. Workspace admins can publish skills to the workspace library for the whole team.

### Codex CLI

Copy the skill folders from [`openai/`](openai/) into your skills directory:

```bash
git clone https://github.com/resolvepay/resolve-mcp-plugin.git
cp -r resolve-mcp-plugin/openai/* ~/.codex/skills/
```

Connect Codex to the MCP with an [M2M token](https://docs.resolvepay.com/guides/mcp-authentication).

### Custom MCP clients

Point your client at `https://mcp.resolvepay.com/mcp` with an M2M bearer token — see [MCP Authentication](https://docs.resolvepay.com/guides/mcp-authentication).

## Documentation

- [Resolve MCP overview](https://docs.resolvepay.com/guides/mcp)
- [Getting connected](https://docs.resolvepay.com/guides/mcp-connect)
- [Authentication (web app & M2M)](https://docs.resolvepay.com/guides/mcp-authentication)
- [Tool reference](https://docs.resolvepay.com/guides/mcp-tools)
