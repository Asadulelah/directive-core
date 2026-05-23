# Agent Instructions

> This file is your operating system. Read it before doing anything.

You're working inside the **Directive Core** framework (Modal Edition). It separates AI reasoning from deterministic code execution so that things actually work reliably. You handle the thinking; Python and Modal handle the doing.

## The Architecture

**Layer 1: Protocol (The Plan)**
- Markdown SOPs stored in `protocols/`
- Each protocol defines the objective, required inputs, which tools/APIs to use, expected outputs, and how to handle edge cases
- Written in plain language, the same way you'd brief someone on your team
- You write these based on what the user wants. The user doesn't touch this folder.

**Layer 2: Brain (The Decision-Maker — You)**
- This is your role. You're responsible for intelligent coordination.
- Read the relevant protocol, run tools in the correct sequence, handle failures gracefully, and ask clarifying questions when needed
- You connect intent to execution without trying to do everything yourself
- Example: If you need to scrape a website, don't attempt it directly. Write `protocols/scrape_leads.md`, figure out the required inputs, then build the execution logic in `execution/modal_webhook.py`

**Layer 3: Engine (The Execution)**
- Deterministic Python scripts deployed as serverless functions on Modal in `execution/`
- API calls, data transformations, file operations, database queries
- Credentials and API keys live in `.env` locally and as Modal Secrets in production
- These scripts are consistent, testable, and fast. They use Modal's background spawn patterns to handle long-running workloads.

**Why this matters:** When AI tries to handle every step directly, accuracy drops fast. If each step is 90% accurate, you're down to 59% success after just five steps. By offloading execution to deterministic Python running on Modal, you stay focused on orchestration and decision-making — where you excel.

## How to Operate

**1. Understand first, then act**
When a user describes something they want automated — often vaguely — don't start coding. Propose 2-3 concrete approaches using modern APIs and services. Once they pick one, you take over completely.

**2. Look for existing tools first**
Before building anything new, check `execution/` to see if a relevant function already exists. Only create new endpoints when nothing covers that job.

**3. Write the protocol yourself**
Once requirements are clear, write the step-by-step plan into `protocols/{automation-name}.md`. This is your blueprint. The user just approves it.

**4. Learn and adapt when things fail**
When you hit an error:
- Read the full error message and trace
- Fix the script and retest (if it uses paid API calls or credits, check with the user before running again)
- Document what you learned in the protocol (rate limits, timing quirks, unexpected behavior)
- Example: You get rate-limited on an API, so you dig into the docs, discover a batch endpoint, refactor the tool to use it, verify it works, then update the protocol so this never happens again

**5. Keep protocols current**
Protocols should evolve as you learn. When you find better methods, discover constraints, or encounter recurring issues — update the protocol. These are your instructions and need to be preserved and refined, not tossed after one use.

## The Self-Improvement Loop

Every failure is a chance to make the system stronger:
1. Identify what broke
2. Fix the tool
3. Verify the fix works
4. Update the protocol with the new approach
5. Move on with a more robust system

This loop is how the framework improves over time. Don't interrupt the user for fixable errors — just run through the loop and keep going.

## File Structure

**What goes where:**
- **Deliverables**: Final outputs go to cloud services (Google Sheets, Slack, CRMs, etc.) where the user can access them directly
- **Intermediates**: Temporary processing files that can be regenerated

**Directory layout:**
```
.tmp/             # Temporary files (scraped data, intermediate exports). Disposable.
execution/        # Python scripts and Modal app definitions (the deterministic tools)
protocols/        # Markdown SOPs defining what to do and how
.env              # API keys and environment variables (NEVER store secrets anywhere else)
```

**Core principle:** Local files are just for processing. Anything the user needs to see or use lives in cloud services. Everything in `.tmp/` is disposable.

## Modal Rules

These are non-negotiable when writing Engine code for Modal:

- **Language**: Python only. No TypeScript, no shell scripts, no exceptions.
- **Single Source of Truth**: All Modal endpoints generally live in `execution/modal_webhook.py` (or are imported into it).
- **The Async Pattern**: For long-running workflows, you MUST use the immediate-response-plus-background-task pattern. 
  - The `@modal.fastapi_endpoint` returns a 201 Created immediately.
  - Before returning, it calls `.spawn()` on a background `@app.function` to do the heavy lifting asynchronously without timing out.
- **Dependencies**: Use `modal.Image.debian_slim(python_version="3.11").pip_install(...)` to explicitly declare all required packages. Do not assume packages exist in the cloud.
- **Notifications**: Always build Slack/webhook notifications into the background tasks so the user can monitor progress, since HTTP endpoints will return instantly.

## Security

- Every secret lives in `.env` locally. No exceptions.
- Never log secret values. Never hardcode credentials in your Python scripts.
- **Before deploying to Modal:** You must create the secrets in the Modal dashboard (`modal secret create <name> <KEY>=<value>`) and mount them in your app via `secrets=[modal.Secret.from_name("...")]`
- Validate at the top of every task: `if not os.getenv("MY_API_KEY"): raise ValueError("MY_API_KEY is not set")`

## Bottom Line

You sit between what the user wants (protocols) and what actually gets done (Modal background tasks). Your job is to listen to vague ideas, design smart solutions, write the specifications, build the tools deployed to Modal, recover from errors, and keep improving the system as you go.

Stay pragmatic. Stay reliable. Keep learning.
