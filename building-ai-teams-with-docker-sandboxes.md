# How to Build AI Agent Teams with Docker Sandboxes

> **Note:** This guide replaces the older blog post at docker.com/blog/building-ai-teams-docker-sandboxes-agent, which used the deprecated `docker sandbox` CLI. All commands here use the current `sbx` CLI.

---

## Overview

Docker Sandboxes let you run AI coding agents in isolated microVM environments. Each sandbox gets its own Docker daemon, filesystem, and network — agents can install packages, run tests, and modify files without touching your host system.

Combined with **Docker Agent** (a multi-agent orchestration tool), you can define a team of specialized AI agents — product manager, designer, engineer, QA — and run them autonomously in a secure sandbox.

---

## Prerequisites

- macOS or Windows (Linux Ubuntu also supported)
- A Docker account (free tier is fine)
- API keys for one or more model providers (OpenAI, Anthropic, Google, etc.)
- The `sbx` CLI installed and authenticated

### Install and sign in

**macOS**
```console
brew install docker/tap/sbx
sbx login
```

**Windows**
```powershell
winget install -h Docker.sbx
sbx login
```



---

## Step 1: Store your API keys as secrets

Secrets are stored securely and injected into the sandbox at runtime — no need to `export` them manually each time.

```console
sbx secret set -g openai       # for OpenAI / GPT models
sbx secret set -g anthropic    # for Claude models
sbx secret set -g google       # for Gemini models
```

You only need to configure the providers you plan to use. Docker Agent auto-detects available credentials and routes requests to the right provider.

Alternatively, you can export keys in your current shell before running the sandbox:
```console
export OPENAI_API_KEY=sk-...
export ANTHROPIC_API_KEY=sk-ant-...
```

---

## Step 2: Define your agent team

Save the following as `dev-team.yaml` inside your project directory. This defines a four-agent team: a product manager (root), a UI designer, an engineer, and a QA specialist.

```yaml
models:
  openai:
    provider: openai
    model: gpt-4o

agents:
  root:
    model: openai
    description: Product Manager - Leads the development team
    instruction: |
      Break user requirements into small iterations. Coordinate designer → engineer → QA.
      - Define features and acceptance criteria
      - Ensure each iteration delivers a complete, testable feature
      - Prioritize based on value and dependencies
    sub_agents: [designer, engineer, qa]
    toolsets:
      - type: filesystem
      - type: think
      - type: todo

  designer:
    model: openai
    description: UI/UX Designer - Creates designs and wireframes
    instruction: |
      Create wireframes and mockups for features.
      - Use consistent patterns and modern design principles
      - Specify colors, fonts, interactions, and mobile layout
    toolsets:
      - type: filesystem
      - type: think

  engineer:
    model: openai
    description: Engineer - Implements features based on designs
    instruction: |
      Build features based on designs. Write clean, tested code.
      - Install dependencies as needed
      - Implement both frontend and backend as required
    toolsets:
      - type: filesystem
      - type: shell
      - type: think

  qa:
    model: openai
    description: QA Specialist - Tests features and identifies bugs
    instruction: |
      Test features and identify bugs. Report issues clearly.
      - Review test results, error messages, and stack traces
      - Confirm that acceptance criteria are met
    toolsets:
      - type: filesystem
      - type: shell
      - type: think
```

**How the team works:**

- **root** acts as product manager, receiving your prompt and delegating to sub-agents
- **designer** produces wireframes and UI specs
- **engineer** implements the code using a `shell` toolset (can run `pip install`, `npm install`, etc.)
- **qa** tests the result and reports findings back to root

Each agent has its own context, model, and set of tools. You can mix models across agents — for example, use `anthropic/claude-sonnet-4-6` for engineering tasks and a lighter model for design.

---

## Step 3: Run Docker Agent in a sandbox

Navigate to your project directory and launch the sandbox:

```console
cd ~/my-project
sbx run docker-agent
```

The workspace defaults to the current directory. You can also specify it explicitly:

```console
sbx run docker-agent ~/my-project
```

By default, the sandbox starts Docker Agent with `run --yolo`. To load your custom agent config file, pass it after `--`:

```console
sbx run docker-agent -- run --yolo dev-team.yaml
```

> **What `--yolo` does:** Allows Docker Agent to execute shell commands without prompting for confirmation at each step — appropriate for sandboxed environments where the agent is already isolated from your host.

---

## Step 4: Describe what you want to build

Once the sandbox is running and Docker Agent is interactive, describe your feature in plain language:

```
Create a bank application using Python. The app should support:
- Account creation
- Check balance
- Deposit and withdraw funds
- Build the UI using Gradio
- Put all files in a directory called app/
```

From there, the agent team takes over:

1. **root** breaks the request into iterations and delegates to designer
2. **designer** generates wireframes and a UI specification
3. **engineer** implements the Gradio app, installing dependencies via `pip install gradio`
4. **qa** runs the app and validates functionality
5. **root** confirms acceptance criteria are met

All of this happens inside the microVM sandbox — your host filesystem is untouched.

---

## Step 5: List and manage your sandboxes

```console
# See all running sandboxes
sbx ls

# Stop a specific sandbox
sbx stop <sandbox-id>

# Remove a sandbox
sbx rm <sandbox-id>
```

Running `sbx run docker-agent` again from the same directory will reuse the existing sandbox for that workspace.

---

## Key concepts

### Why microVM isolation matters

Docker Sandboxes run inside dedicated microVMs — stronger isolation than a container alone. Each sandbox has its own Docker daemon, filesystem, and network. An agent can make mistakes (install the wrong package, write bad files) without any of that affecting your host.

### Workspace mounting

Your project directory is mounted into the sandbox at the same absolute path. This means hard-coded paths in scripts, relative imports, and error messages all work exactly as they would locally — the agent just can't escape the mounted workspace.

### Credential handling

Environment variables from your shell session are **not** inherited by the sandbox. Use `sbx secret set` for persistent credentials, or pass keys explicitly via `-e` flags if needed.

### Agent configuration scope

Sandboxes only see project-level configuration (files in your working directory). User-level agent config from your host (e.g. `~/.config/docker-agent/`) is not available inside the sandbox. Keep your `dev-team.yaml` alongside your project code.

---

## Supported agents

Beyond Docker Agent, `sbx run` supports these agents directly:

```console
sbx run claude          # Claude Code
sbx run codex           # OpenAI Codex
sbx run gemini          # Gemini CLI
sbx run copilot         # GitHub Copilot
sbx run cursor          # Cursor
sbx run kiro            # Kiro
sbx run opencode        # OpenCode
sbx run shell           # Plain shell (bring your own tooling)
```

---

## Further reading

- [Docker Sandboxes documentation](https://docs.docker.com/ai/sandboxes/)
- [Docker Agent documentation](https://docs.docker.com/ai/docker-agent/)
- [sbx CLI reference](https://docs.docker.com/reference/cli/sbx)
- [Sandbox security model](https://docs.docker.com/ai/sandboxes/security/)
- [Docker Agent configuration reference](https://docs.docker.com/ai/docker-agent/reference/config/)
- [Agent team examples on GitHub](https://github.com/docker/cagent/tree/main/examples)
