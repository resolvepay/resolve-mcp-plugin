# Resolve MCP Plugins & Skills

Plugins and skills for managing your [Resolve Pay](https://resolvepay.com) account from AI assistants, powered by the [Resolve MCP](https://docs.resolvepay.com/guides/mcp) server (`https://mcp.resolvepay.com/mcp`).

Everything lives on `main`. Each path is its own marketplace:

| Marketplace root | For | Manifest | Contents |
|---|---|---|---|
| `/` (repo root) | Claude Code | `.claude-plugin/marketplace.json` | Plugins in [`plugins/`](plugins/) bundling the MCP connector and skills |
| [`openai/`](openai/) | Codex (also source for ChatGPT skills) | `openai/.agents/plugins/marketplace.json` | Plugins in [`openai/plugins/`](openai/plugins/) using the file-attachment flow |

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

The base plugin includes the MCP connector — on first use, Claude Code opens the Resolve sign-in (your dashboard login). See the [Claude Code plugins documentation](https://code.claude.com/docs/en/plugins).

### Claude (web / desktop)

Download the skill zips from the [docs site](https://docs.resolvepay.com/guides/mcp-skill) and upload them as skills — see [Use skills in Claude](https://support.claude.com/en/articles/12512180-use-skills-in-claude). Then add the Resolve connector (server URL `https://mcp.resolvepay.com/mcp`).

### ChatGPT

Add the Resolve connector (server URL `https://mcp.resolvepay.com/mcp`), then upload the skill(s) from [`openai/plugins/`](openai/plugins/) — zip a skill folder, or download the prebuilt zips from the [docs site](https://docs.resolvepay.com/guides/mcp-skill). See [Skills in ChatGPT](https://help.openai.com/en/articles/20001066-skills-in-chatgpt).

### Codex

Add this repo as a plugin marketplace with the sparse path `openai/`:

```bash
codex plugin marketplace add https://github.com/resolvepay/resolve-mcp-plugin.git --sparse openai
```

(Or in the UI: source `https://github.com/resolvepay/resolve-mcp-plugin`, git ref `main`, sparse path `openai/`.) Then install `resolve-mcp` and the optional domain plugins. See the [Codex plugins documentation](https://developers.openai.com/codex/plugins).

### Custom MCP clients

Point your client at `https://mcp.resolvepay.com/mcp` with an M2M bearer token — see [MCP Authentication](https://docs.resolvepay.com/guides/mcp-authentication).

## Documentation

- [Resolve MCP overview](https://docs.resolvepay.com/guides/mcp)
- [Getting connected](https://docs.resolvepay.com/guides/mcp-connect)
- [Authentication (web app & M2M)](https://docs.resolvepay.com/guides/mcp-authentication)
- [Tool reference](https://docs.resolvepay.com/guides/mcp-tools)
