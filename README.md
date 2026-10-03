# monaiq

Add licensing, subscriptions, and feature gates to your .NET or React app - AI-guided SDK integration, pricing design, and catalog management.

> This repository is the **published distribution** of the Monaiq AI plugin, assembled and
> mirrored here by Sidub's release pipeline. Do not edit it by hand - changes are overwritten on the next release.

Version: `2.1.0` | License: MIT | Homepage: https://docs.monaiq.com

## Install

### Claude Code
```
/plugin marketplace add Sidub-Inc/Monaiq.AI
/plugin install monaiq@monaiq
```

### VS Code / GitHub Copilot
Add the plugin from this repository (`Sidub-Inc/Monaiq.AI`) - Copilot discovers `plugin.json` at the repo root.

### Codex CLI
Add the plugin from this repository (`Sidub-Inc/Monaiq.AI`) - Codex discovers `.codex-plugin/plugin.json` and `.agents/plugins/marketplace.json`.

## What's inside

- `skills/` - auto-invocable skills (12 total)
- `agents/` - the `monaiq` agent (Claude + Copilot)
- `.mcp.json` - MCP server config (all platforms); server endpoint `https://api.monaiq.com/mcp`
- platform manifests: `plugin.json` (Copilot), `.claude-plugin/` (Claude), `.codex-plugin/` + `.agents/` (Codex)

The MCP server is hosted at `https://api.monaiq.com/mcp` and is shared by every platform manifest.

## Sign-in

The server uses MCP-spec OAuth: the first time your assistant connects it will prompt you to
sign in with an email one-time passcode. There is no API key or session key to configure -
the URL above is the entire setup.