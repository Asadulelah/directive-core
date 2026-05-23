# Directive Core

> **A framework that upgrades any AI coding assistant into a seasoned systems architect — specialized in building robust, deterministic background workflows.**

You write plain-English protocols. The AI reads its instructions, follows the rules, and builds, tests, and deploys real background workflows. No vibes-coded prompts. No hallucinated SDK syntax. Just a clean three-layer architecture: **Protocol → Brain → Engine**.

---

## Pick your engine

This repo ships two flavors of the framework — same Brain, different execution engines:

### [`trigger.dev/`](./trigger.dev) — for **Trigger.dev**
Cloud-native background tasks with built-in retries, idempotency, and concurrency control. Best when you want managed infrastructure and easy observability. Includes [`mcp.json`](./trigger.dev/mcp.json) for direct AI-to-Trigger.dev integration via MCP.

### [`modal/`](./modal) — for **Modal**
Serverless Python compute with GPU access, fast cold-starts, and powerful scheduling primitives. Best when you need heavy compute, ML inference, or Python-first pipelines.

Both flavors use the same Protocol contract. You can switch engines without changing your protocols.

---

## How it works

1. **You write a protocol** — a plain-English Markdown file in `protocols/` describing what you want automated. Goal, inputs, outputs, schedule. No code.

2. **You point your AI at the protocol** — Cursor, Windsurf, Claude Code, Antigravity, whatever you use. The AI reads the Directive Core instructions and the engine reference, then takes over.

3. **The AI does everything else** — picks the right primitives, writes the code with current SDK syntax (no hallucinations because the reference file is loaded), validates, deploys, and reports back.

---

## Quick start

Inside whichever engine folder you pick, the [`HOW_TO_USE.md`](./trigger.dev/HOW_TO_USE.md) walks through:

1. Install the framework files in your project
2. Configure your AI IDE (Cursor, Windsurf, Claude Code, Antigravity, etc.)
3. Write your first protocol
4. Unleash the agent

The whole setup takes under five minutes. From there, every new automation is a Markdown file plus a single AI prompt.

---

## The architecture

**Three layers. Strict separation.**

| Layer | What it owns | Where it lives |
|---|---|---|
| **Protocol** | The plain-English description of what to automate | `protocols/*.md` files in your project |
| **Brain** | The rules, mindset, and constraints the AI must follow | `DIRECTIVE_CORE.md` (loaded into the AI's system instructions) |
| **Engine** | The actual runtime — Trigger.dev or Modal | The deployed background task |

The Brain doesn't write protocols. The Engine doesn't make architectural decisions. The Protocol doesn't contain code. Each layer has one job. That separation is the whole point — it's what keeps the AI from hallucinating and what keeps the workflows deterministic.

---

## Why this exists

Most AI-built automation looks great in a demo and falls apart in production. The failure modes are predictable: stale SDK syntax, half-built error handling, no retry logic, no idempotency, secrets leaked into the wrong layer.

Directive Core engineers against each of those failure modes by separating concerns — the AI's freedom lives in the Protocol layer; the Brain enforces the rules; the Engine handles infrastructure. The result is workflows that ship the first time and survive the third week.

---

## License

[MIT](LICENSE). Use it, fork it, build on it.

---

## Built by

[**Asadulelah**](https://github.com/Asadulelah) — frameworks for AI-native operations. More repos + the full portfolio at [asad-portfolio-lake.vercel.app](https://asad-portfolio-lake.vercel.app).
