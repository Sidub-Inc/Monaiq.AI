# Monaiq

**Add licensing, subscriptions, and feature gates to your .NET or React app - guided step by step, right inside your AI coding assistant.**

Monaiq turns "I need to charge for this" into working code. Ask your assistant to set up licensing and the Monaiq plugin walks it through pricing design, catalog setup, SDK integration, checkout, and feature gating - using your real product catalog and credentials instead of boilerplate guesses.

Works with **Claude Code**, **GitHub Copilot** (VS Code), and **Codex CLI**.

## What you can do

Once it's installed, just ask in plain language. For example:

- "Use Monaiq to add licensing to my .NET app."
- "Design a pricing tier for my SaaS product."
- "Add a buy button and Stripe checkout to my React app."
- "Gate this feature behind the Pro plan."
- "Set up a free trial with usage limits."

Monaiq handles the licensing domain so you can stay focused on your product:

- **Pricing and packaging** - design tiers, trials, and usage-based plans
- **Catalog management** - products, features, and offerings
- **SDK integration** - install and wire up the licensing SDK for .NET or React
- **Purchase flows** - embedded in-app checkout, or a no-code public storefront
- **Feature gating** - entitlement checks, rate limits, and paywalls

## Getting started

You'll need a Monaiq account to issue and manage licenses - create one at [monaiq.com](https://monaiq.com). The plugin guides you through connecting it the first time you use it.

### Claude Code
```
/plugin marketplace add Sidub-Inc/Monaiq.AI
/plugin install monaiq@monaiq
```

### GitHub Copilot (VS Code)
Add this repository (`Sidub-Inc/Monaiq.AI`) as an agent plugin source. See the [Copilot agent plugins guide](https://code.visualstudio.com/docs/copilot/customization/agent-plugins).

### Codex CLI
Add this repository (`Sidub-Inc/Monaiq.AI`) as a plugin. See the [Codex documentation](https://developers.openai.com/codex).

## How it works

Monaiq runs as a plugin in your AI coding assistant and connects to the hosted Monaiq service that manages your products, offerings, licenses, and checkout. Your assistant receives step-by-step, framework-aware guidance, so the code it writes matches the current SDK and your live catalog.

## Documentation and support

- Documentation: https://docs.monaiq.com
- Website: https://monaiq.com
- Questions or feedback: [open an issue](https://github.com/Sidub-Inc/Monaiq.AI/issues)

## License

MIT - Copyright (c) Sidub Inc.

---

_This is the official distribution of the Monaiq plugin and is updated automatically with each release. For help or to request a change, please open an issue rather than editing files here._