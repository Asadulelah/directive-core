# How to Use Directive Core

Welcome to **Directive Core**, the ultimate agentic automation framework. 

This framework upgrades any AI coding assistant into a seasoned systems architect and unyielding automation builder, specialized in **Trigger.dev**. 

By feeding your AI this framework, you are giving it the mindset, knowledge, and tools to autonomously build, test, and deploy robust, deterministic background workflows.

Here is how to set it up in your favorite AI IDE or Terminal Agent (like Cursor, Windsurf, Claude Code, or Antigravity):

## Step 1: Install the Framework

You have three files that make up the framework. You need to provide them to your AI:

1. **`DIRECTIVE_CORE.md` (The Brain's Instructions)**
   - This defines the Protocol → Brain → Engine architecture and sets the strict rules your AI must follow.
   - **How to install:** Copy this file to the root of your project. Rename it to match the standard system instruction file for your tool. 
     - *Cursor:* Rename to `.cursorrules`
     - *Windsurf:* Rename to `.windsurfrules`
     - *Claude Desktop/Projects:* Upload it or paste the contents into "Custom Instructions".
     - *Other Agents (Antigravity, etc):* Keep it as `DIRECTIVE_CORE.md` or `AGENTS.md` and explicitly tell the AI: *"Before we start, read the system instructions in DIRECTIVE_CORE.md"*

2. **`trigger-ref.md` (The Engineering Textbook)**
   - Trigger.dev is powerful and its SDK updates often. This file prevents the AI from hallucinating old syntax by giving it perfect, modern code examples.
   - **How to install:** Place this file in your project root. The AI will read it automatically when it needs to build a task.

3. **`mcp.json` (The Hands)**
   - This connects your AI directly to the Trigger.dev Cloud using the Model Context Protocol (MCP). It lets the AI deploy code, check logs, and run local dev servers autonomously.
   - **How to install:** Follow your specific AI editor's instructions for adding an MCP server, using the configuration provided in this file.

## Step 2: Write Your First Protocol

Create a folder named `protocols/` in your project. Inside, create a simple Markdown file (e.g., `protocols/scrape_leads.md`) written in plain English. 

Pretend you are writing instructions for a human employee. Tell it:
- What the goal of the automation is.
- Where the data comes from.
- Where the data should go.
- How often it should run (e.g., "Run every morning at 9 AM").

## Step 3: Unleash the Agent

Now, open your AI chat window and simply type:

> *"Please build the automation described in protocols/scrape_leads.md"*

### What Happens Next?

Once you hit enter, your AI will completely take over, following the Directive Core rules perfectly:

1. **Understanding:** It will read the protocol and immediately ask you targeted questions (e.g., *"Do you have the API key for this service? What should I do if the data is missing?"*).
2. **Setup:** It will securely scaffold the `src/trigger` directory and remind you to update your `.env` variables—refusing to hardcode secrets.
3. **Execution Code:** It will write perfect TypeScript logic using the `@trigger.dev/sdk`. It will utilize idempotency, retries, and durable wait states automatically because you taught it exactly how with `trigger-ref.md`.
4. **Testing:** It uses the MCP tools to run the dev server and test the automation locally. If an error occurs, the AI won't give up—it reads the stack trace, fixes the code, and tests again (Self-Annealing).
5. **Deployment:** Once it is working locally, it will ask for your explicit permission. Once granted, the AI will use the MCP tool to automatically deploy your automation to Trigger.dev production.

You don't need to know how to code. You just write the Protocol, and the Brain handles the Engine!
