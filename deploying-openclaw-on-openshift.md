# Deploying OpenClaw on OpenShift with Model as a Service

A friend of mine runs OpenClaw on a dedicated Mac. His setup is impressive — Telegram, Slack (chat and alerts channels), Gmail, Google Drive, GitHub with its own account, web search via Brave and Perplexity, and a "Second Brain" built from markdown docs auto-synced to a private GitHub repo. He uses GPT-5 as the primary model and manages the whole thing via Claude Code over SSH. He even does travel research with Google Maps pins shared with his son.

I wanted to build something similar — in my own way. Instead of a dedicated Mac, I have an OpenShift cluster. Instead of GPT-5, my company provides Model as a Service. Here's how I got OpenClaw running on OpenShift and talking through Telegram.

## What is OpenClaw?

OpenClaw is an open-source personal AI assistant framework. It's a Node.js/TypeScript project with 334K+ stars on GitHub, MIT licensed. You install it locally with `npm install -g openclaw@latest`, run the onboard wizard, and it connects to messaging platforms like Telegram, Slack, WhatsApp, Discord, and more.

The onboard wizard creates a configuration file called `openclaw.json` that controls everything the agent does. On Kubernetes, there's no interactive wizard — so we create that config manually.

## Why OpenShift?

My friend's setup is tied to one machine. If it goes down, everything stops. OpenShift gives me automatic restarts via health probes, persistent storage via PVCs, network isolation via NetworkPolicies, and the ability to manage everything declaratively with YAML. No dedicated hardware needed.

## Choosing the Right Model

My company's MaaS platform offers several models through a LiteLLM proxy. For an agent like OpenClaw, the most important capabilities are **function calling** and a large enough **context window** to handle the system prompt.

I initially chose **deepseek-r1-distill-qwen-14b** — it had the largest advertised context window (4M tokens) and supported function calling. But it turned out to have issues — the actual context window via LiteLLM was only 16K tokens, and the model leaked its internal reasoning process (showing `<think>` tags in responses).

The final choice was **llama-scout-17b**:

- Function calling support
- 327K token context window (verified working)
- 17B parameters — clean responses without reasoning leakage
- Fast inference (~2 second response time)

The MaaS endpoint is OpenAI-compatible (`/v1/chat/completions`), which is exactly what OpenClaw expects. No adapter needed — just point it at the endpoint.

I also noted some free supporting models for future use:

- **Llama-Guard-3-1B** — safety filter layer (free)
- **nomic-embed-text-v1-5** — semantic search for the Second Brain (free)
- **Docling** — document conversion (free)

## Understanding openclaw.json

Before diving into manifests, it helps to understand what `openclaw.json` controls. It's the single master configuration file for the entire system:

- **gateway** — how the server runs (port, bind address, authentication)
- **agents.defaults** — the infrastructure layer shared by all agents (which model, workspace path, timeouts). Think of this as the "plumbing."
- **agents.list** — the agent definitions (name, personality, behavior overrides, tool restrictions). Think of this as "who are the agents."
- **models.providers** — where the AI brains come from. Your MaaS endpoint, OpenAI, Anthropic, or local models.
- **channels** — which messaging platforms to connect (Telegram, Slack, WhatsApp, etc.) with tokens and access control
- **tools** — what the agent is allowed to do (shell commands, file read/write, browser)
- **skills** — extra plugins from ClawHub (5,700+ community skills)
- **cron** — scheduled tasks (this is how you get news summaries every 3 hours)
- **hooks** — external event triggers (Gmail webhooks, GitHub notifications)

There's also a file called **AGENTS.md** — this is the system prompt injected into every LLM call. It tells the model who it is, how to behave, and what it can do. But OpenClaw also auto-generates several other workspace files (`SOUL.md`, `IDENTITY.md`, `USER.md`, `BOOTSTRAP.md`, `TOOLS.md`, `HEARTBEAT.md`) that all get injected too. More on that later.

## The Architecture

Everything lives in a single namespace with no cluster-scoped resources:

```
openclaw (namespace)
├── NetworkPolicy: deny-from-other-namespaces
├── NetworkPolicy: allow-external-egress-only
├── Secret: openclaw-secrets (API keys + Telegram bot token)
├── PVC: openclaw-data (10Gi persistent storage)
├── ConfigMap: openclaw-config (openclaw.json + AGENTS.md)
├── Deployment: openclaw (init container + gateway)
├── Service: openclaw (ClusterIP on port 18789)
└── Route: openclaw (TLS edge termination)
```

## Step 1 — Namespace and Network Isolation

Create a namespace and lock it down immediately with two NetworkPolicies.

The **ingress policy** allows traffic only from pods within the same namespace and from the OpenShift router (so Routes can deliver traffic). Everything else is blocked.

The **egress policy** allows outbound traffic to the internet (for Telegram, MaaS APIs) and cluster DNS, but blocks all private IP ranges. OpenClaw cannot reach pods in other namespaces.

## Step 2 — Secret for API Keys

The Secret stores your MaaS API key and Telegram bot token. These get injected as environment variables into the pod. The `openclaw.json` references `${OPENAI_API_KEY}` and `${TELEGRAM_BOT_TOKEN}` which OpenClaw resolves from the environment at runtime.

## Step 3 — Persistent Storage

A 10Gi PVC mounted at `/.openclaw` inside the container. This is where OpenClaw stores its configuration, workspace files, conversation history, and session data. The PVC persists across pod restarts.

On OpenShift, the PVC is owned by the pod's UID. OpenShift assigns a consistent UID per namespace, so data remains accessible even after restarts.

## Step 4 — ConfigMap with openclaw.json

Since there's no interactive onboard wizard in Kubernetes, we create the config manually in a ConfigMap:

```json
{
  "gateway": {
    "mode": "local",
    "port": 18789,
    "bind": "lan",
    "controlUi": { "enabled": true },
    "auth": { "mode": "token" }
  },
  "agents": {
    "defaults": {
      "workspace": "/.openclaw/workspace",
      "model": { "primary": "maas/llama-scout-17b" }
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "identity": {
          "name": "MyClaw",
          "theme": "helpful assistant",
          "emoji": "🤖"
        },
        "tools": {
          "allow": ["web_search", "web_fetch", "session_status"]
        }
      }
    ]
  },
  "models": {
    "mode": "merge",
    "providers": {
      "maas": {
        "baseUrl": "https://your-litellm-endpoint/v1",
        "apiKey": "${OPENAI_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "llama-scout-17b",
            "name": "Llama 4 Scout 17B",
            "reasoning": false,
            "input": ["text"],
            "contextWindow": 327680,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "${TELEGRAM_BOT_TOKEN}",
      "allowFrom": ["YOUR_TELEGRAM_USER_ID"]
    }
  }
}
```

Key settings:

- **bind: "lan"** — makes the gateway listen on `0.0.0.0` (all interfaces). In Kubernetes, the kubelet probes, Service, and Route all hit the pod IP, not localhost. Without `lan`, the gateway only listens on `127.0.0.1` and nothing outside the container can reach it.
- **agents.defaults** — the infrastructure layer. Model and workspace shared by all agents.
- **agents.list** — defines one agent called "main" with its own identity and restricted tools.
- **tools.allow** — critical for smaller models. OpenClaw ships 27 tools by default, adding ~12,600 tokens of JSON schemas to every API call. For models like `llama-scout-17b`, this overwhelms the prompt. Restricting to 3 essential tools keeps the prompt manageable.
- **models.providers.maas** — registers your MaaS endpoint as a custom provider.
- **channels.telegram** — enables Telegram with a bot token and an allowlist of authorized user IDs.

## Step 5 — The Deployment

The Deployment has two containers:

**Init container** — runs first, copies `openclaw.json` and `AGENTS.md` from the ConfigMap into the PVC. This is necessary because OpenClaw modifies its own config at runtime (auto-generating a gateway token, adding metadata). The ConfigMap is read-only, but the PVC is read-write.

**Main container** — runs OpenClaw. It finds the config at `/.openclaw/openclaw.json`, starts the gateway, connects to Telegram via long-polling, and begins listening.

Important deployment settings:

- **strategy: Recreate** — the PVC is `ReadWriteOnce`, so only one pod can mount it at a time. `Recreate` ensures the old pod is terminated before the new one starts.
- **livenessProbe initialDelaySeconds: 120** — OpenClaw takes over a minute to fully boot. The liveness probe waits 2 minutes before its first check.
- **resources: 2Gi RAM request, 4Gi limit** — sufficient for the gateway and agent processing.

## Step 6 — Service and Route

A ClusterIP Service on port 18789 gives the pod a stable internal DNS name. A Route with TLS edge termination exposes it externally via HTTPS for the Control UI.

## Step 7 — Access the Control UI

After deployment, OpenClaw auto-generates a gateway token and saves it to the config file on the PVC. Retrieve it with:

```
oc exec deployment/openclaw -n openclaw -- cat /.openclaw/openclaw.json
```

Look for `gateway.auth.token` in the output. Open the Route URL in your browser and paste the token to authenticate.

The first time you connect, the Control UI shows "pairing required." OpenClaw requires new devices to be approved. List pending requests and approve:

```
oc exec deployment/openclaw -n openclaw -- openclaw devices list
oc exec deployment/openclaw -n openclaw -- openclaw devices approve <uuid>
```

The UUID comes from the "Request" column in the pending devices table. After approval, refresh the browser — you're in.

## Step 8 — Telegram Integration

Telegram is the easiest messaging channel to set up because it uses a simple bot token — no QR codes, no phone linking, no Apple device required.

**Create a bot:** Message @BotFather on Telegram, send `/newbot`, choose a name and username. BotFather gives you a token like `8674739244:AAFy5MMWiDgc3dQOw2SauvGQ92896-8SfWQ`.

**Get your Telegram user ID:** Message @userinfobot on Telegram — it replies with your numeric ID. This goes in the `allowFrom` array so only you can talk to the bot.

**Add to Secret:** The bot token goes in the Kubernetes Secret as `TELEGRAM_BOT_TOKEN`. OpenClaw resolves `${TELEGRAM_BOT_TOKEN}` from the environment.

**How it works under the hood:** OpenClaw uses Telegram's long-polling API (`getUpdates`). It keeps an HTTP connection open to `api.telegram.org`, waiting for new messages. When you send "Hello" in Telegram, the flow is:

1. You type "Hello" → Telegram servers queue it
2. OpenClaw's long-poll receives it via `getUpdates`
3. OpenClaw wraps it with metadata and sends it to `llama-scout-17b`
4. The model responds
5. OpenClaw calls `sendMessage` on the Telegram API to deliver the reply

No webhooks, no public endpoints needed. The bot initiates all connections outbound — perfect for a pod behind NetworkPolicies.

## The Tool Token Problem — Why Smaller Models Need Restrictions

This was the hardest issue to debug. After configuring Telegram, messages were delivered to the bot but the LLM responded with `NO_REPLY` to everything — "Hello", "How are you?", even simple greetings.

The investigation revealed the root cause through A/B testing the actual API calls:

- **With 27 tools** (OpenClaw default) → ~19,000 prompt tokens → model says `NO_REPLY`
- **Without tools** → ~6,400 prompt tokens → model says "Hello Nirjhar! How can I help?"

OpenClaw ships 27 tools enabled by default (file read/write, shell exec, browser control, cron, messaging, image generation, PDF analysis, etc.). Each tool includes a JSON schema in the API call. Together they add **~12,600 tokens** to every request. Combined with the system prompt, workspace files, and the `## Silent Replies` instruction (which tells the model to say `NO_REPLY` when it has nothing to add), the model gets overwhelmed and defaults to silence.

The fix: `"tools": {"allow": ["web_search", "web_fetch", "session_status"]}` in the agent config. This drops from 27 tools to 3, keeping the prompt well within what `llama-scout-17b` handles cleanly.

GPT-5 or Claude wouldn't have this problem — they handle 100K+ token prompts with ease. But for smaller open-source models via MaaS, tool restriction is essential.

## Auto-Generated Workspace Files

On first boot, OpenClaw creates 7 template files in `/.openclaw/workspace/`:

- **AGENTS.md** (7,874 chars) — master instructions about group chat behavior, heartbeats, memory management, when to stay silent
- **SOUL.md** (1,673 chars) — personality ("you're not a chatbot, you're becoming someone")
- **BOOTSTRAP.md** (1,471 chars) — first-run conversation script
- **TOOLS.md** (860 chars) — template for local tool notes
- **IDENTITY.md** (636 chars) — template for name, emoji, vibe
- **USER.md** (477 chars) — template for learning about the human
- **HEARTBEAT.md** (193 chars) — periodic check template

All of these get injected into the system prompt on every LLM call. That's 13,184 characters of instructions designed for GPT-5-class models. For `llama-scout-17b`, I trimmed them down to 668 characters total — just the essentials.

These files live on the PVC, not in the ConfigMap. They survive pod restarts. You can edit them freely — they're meant to be customized by you or by the agent itself over time.

## What's Next

The full loop is working: Telegram → OpenClaw → llama-scout-17b → Telegram. Next steps:

- Add Slack with separate chat and alerts channels
- Set up cron jobs for scheduled news summaries
- Configure the web search skill (the tool is enabled, just needs a Brave API key)
- Build the Second Brain with markdown notes and semantic search using the free nomic embedding model
- Add Llama-Guard safety filter
- Try enabling more tools as the model proves capable

All manifests are in the `manifests/` directory, numbered in apply order. Full troubleshooting guide and detailed explanations are in the README.
