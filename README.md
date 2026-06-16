# Zapf Protocol Agent Skills ⚡️

AI Agent skills for building, debugging, and integrating the [Zapf Protocol](https://docs.zapf.app).

This repository contains machine-readable markdown instructions (`SKILL.md`) designed to be consumed by agentic coding tools like Claude Code, Cursor, and Copilot Workspace.

## Installation

You can install these skills directly into your project using the [Agent Skills CLI](https://skills.sh/).

To install the core Zapf Protocol skill:

```bash
npx skills add ohstr/zap-protocol
```

This will download the skill into your local `.agents/skills/zap-protocol` directory, giving your AI assistant instant expertise on:
- Kind 5520 (Zap Requests) and 5521 (Zap Receipts)
- Fallback Lightning Addresses and LNURL Webhooks
- Identity Resolution workflows

## Usage

Once installed, simply ask your AI agent to implement a feature related to the Zapf Protocol, and it will automatically reference the installed skill for the canonical rules, code examples, and best practices.

*Example prompt:*
> "Create a new function to generate a Kind 5520 Zap Request event according to the zap-protocol skill."
