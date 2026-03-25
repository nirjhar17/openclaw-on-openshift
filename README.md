# OpenClaw on OpenShift — Complete Reference

This directory contains everything needed to deploy OpenClaw on OpenShift, plus a detailed troubleshooting guide covering every issue encountered during the initial deployment.

## Directory Structure

```
openclaw/
├── README.md                              ← You are here (full reference + troubleshooting)
├── deploying-openclaw-on-openshift.md     ← Blog post (clean installation guide)
└── manifests/
    ├── 01-namespace.yaml
    ├── 02-networkpolicy-deny-ingress.yaml
    ├── 03-networkpolicy-allow-egress.yaml
    ├── 04-secret.yaml
    ├── 05-pvc.yaml
    ├── 06-configmap.yaml
    ├── 07-deployment.yaml
    ├── 08-service.yaml
    └── 09-route.yaml
```

## Quick Deploy

```bash
# Login
oc login -u <user> -p <password> <cluster-url>

# Edit 04-secret.yaml with your real API key and Telegram bot token first
# Edit 06-configmap.yaml to set your Telegram user ID in allowFrom
# Then:
oc apply -f manifests/
```

## Background and Motivation

A friend built an OpenClaw setup on a dedicated Mac with these integrations:

- **Telegram** (primary messaging channel)
- **iMessage** (via iCloud account — only possible because it runs on macOS)
- **Slack** — chat channel and alerts channel
- **News summaries** every 3 hours or daily via cron
- **Second Brain** — markdown docs auto-synced to a private GitHub repo, with qmd for better indexing
- **GitHub** — has its own GitHub account, uses gh CLI
- **Gmail and Google Drive**
- **Web search** via Brave and Perplexity — used for travel research/planning including Google Maps pin itineraries
- **GPT-5** as primary model via API
- **Dedicated Mac hardware** — managed remotely via Claude Code over SSH (like Ansible — no agent on the managed machine)

The goal was to build something similar but on OpenShift, using the company's Model as a Service (MaaS) instead of GPT-5, and without needing dedicated hardware.

## iMessage — Why It Doesn't Apply Here

iMessage is Apple's proprietary messaging service — like WhatsApp but locked to Apple devices. There's no official API, no Linux client, no open protocol. On a Mac, OpenClaw can integrate with iMessage via:

- AppleScript calling Messages.app
- The `imsg` CLI tool (github.com/steipete/imsg)
- Reading the SQLite database at `~/Library/Messages/chat.db`

None of these work in a Linux container on OpenShift. If you need iMessage, you must keep a Mac in the loop as a bridge. Since this deployment doesn't use iMessage, there's no macOS dependency — everything runs on Linux.

## Model Selection Analysis

The company's MaaS offers these models via a LiteLLM proxy:

**Can power an agent (has function calling):**

- **llama-scout-17b** — FINAL CHOICE. Function calling, 327K context (verified), 17B parameters. Clean responses, no reasoning leakage. Fast ~2s inference. Handles OpenClaw's system prompt well when tools are restricted.
- **deepseek-r1-distill-qwen-14b** — Initially chosen but abandoned. Dashboard shows 4M context, but LiteLLM caps it at 16K tokens. Also leaks internal reasoning with `<think>...</think>` tags in responses. Function calling with parallel functions and tool choice.

**Chat only (no function calling — cannot reliably power an agent):**

- granite-3-2-8b-instruct
- granite-4-0-h-tiny
- microsoft-phi-4
- codellama-7b-instruct (also only 4K context)
- qwen3-14b (capabilities not fully listed)

**Special purpose (not for agent use):**

- Docling — document conversion (free)
- Llama-Guard-3-1B — safety/content moderation filter (free)
- nomic-embed-text-v1-5 — embeddings for semantic search (free)

**Why function calling matters:** OpenClaw needs to decide when to call Telegram, search the web, read GitHub, send emails, etc. Without function calling, the model can't reliably trigger these actions.

**Current stack:** llama-scout-17b as the brain, with plans to add Llama-Guard as a safety filter, nomic for Second Brain search, and Docling for document conversion — all from the same MaaS platform.

## What is MaaS (Model as a Service)?

The company hosts LLMs behind an API endpoint. It's OpenAI-compatible — same `/v1/chat/completions` and `/v1/completions` endpoints. OpenClaw doesn't know or care that it's not talking to OpenAI.

Advantages over commercial APIs:

- **No external API costs** — model is already running on company infrastructure
- **Lower latency** — if MaaS is in the same network, no internet round trip
- **Data stays internal** — messages, notes, and Second Brain content never leave company infrastructure
- **No rate limits** — depends on your MaaS tier/quota, but typically more generous

The MaaS endpoint for this deployment: `https://litellm-prod.apps.maas.redhatworkshops.io/v1`

## Understanding openclaw.json In Depth

`openclaw.json` is the single master configuration file. When installed locally, the `openclaw onboard` wizard creates it interactively. In Kubernetes, we create it manually via a ConfigMap.

### Full Section Breakdown

| Section | Purpose |
|---------|---------|
| `gateway` | Server config — port, bind address, auth mode, Control UI settings |
| `agents.defaults` | Infrastructure layer — model, workspace, timeouts. Shared by ALL agents |
| `agents.list` | Agent definitions — identity, personality, per-agent overrides |
| `models.providers` | Where AI brains come from — MaaS, OpenAI, Anthropic, local models |
| `channels` | Messaging platforms — Telegram, Slack, WhatsApp, Discord, Signal, iMessage |
| `tools` | Capabilities — shell exec, file read/write, browser, elevated permissions |
| `skills` | Plugins from ClawHub — 5,700+ community skills |
| `cron` | Scheduled tasks — news summaries, health checks, periodic reports |
| `hooks` | External triggers — Gmail webhooks, GitHub notifications |
| `session` | Conversation management — per-sender, reset triggers, history limits |
| `routing` | Message routing — group chat mention patterns, queue modes |
| `logging` | Log levels, file paths, sensitive data redaction |
| `messages` | Response formatting — prefixes, reactions, acknowledgements |
| `auth` | Provider authentication — OAuth profiles, API key fallback order |

### agents.defaults vs agents.list

Think of it as infrastructure vs identity:

**agents.defaults** = the infrastructure layer. Answers "how do agents operate?"
- Which model to use
- Where to store files
- Timeout settings
- Shared by every agent

**agents.list** = the agent definitions. Answers "who are the agents?"
- Name, personality, emoji
- Can override defaults (different model, different thinking level)

Example with multiple agents:

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "maas/llama-scout-17b" },
      "workspace": "/.openclaw/workspace"
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "identity": { "name": "MyClaw", "theme": "helpful assistant" },
        "tools": { "allow": ["web_search", "web_fetch", "session_status"] }
      },
      {
        "id": "researcher",
        "identity": { "name": "ResearchBot", "theme": "deep research analyst" },
        "tools": { "allow": ["web_search", "web_fetch", "file_read"] }
      }
    ]
  }
}
```

Both agents share the workspace from defaults. "main" uses the default model. "researcher" overrides with a different model.

### What is AGENTS.md?

AGENTS.md is the system prompt. It gets injected into every LLM call:

```
AGENTS.md content             → system prompt
+ workspace .md files         → additional context (SOUL.md, IDENTITY.md, etc.)
+ OpenClaw internal prompt    → Silent Replies, Tooling, Heartbeat instructions
+ user's message              → user prompt
+ tool definitions            → function calling JSON schemas
                              → all sent to the LLM together
```

This is where you define personality, behavior rules, and capabilities. But note that `AGENTS.md` from the ConfigMap is only the seed — OpenClaw auto-generates a much larger version in `/.openclaw/workspace/AGENTS.md` on first boot, plus 6 additional `.md` files. All of these contribute to the system prompt. For smaller models, you must trim them down (see Troubleshooting, Error 12).

### gateway.bind — "lan" vs "loopback"

`loopback` = listen on `127.0.0.1` only. Only processes inside the same container can reach the gateway.

`lan` = listen on `0.0.0.0` (all interfaces). Any device on the network can reach it.

In Kubernetes, three things need to reach the gateway via the pod IP (not localhost):
1. Liveness/readiness probes from the kubelet
2. The Service routing traffic
3. The Route delivering external traffic

If the gateway only listens on loopback, all three get `connection refused`. Use `lan` for any Kubernetes deployment.

### gateway.auth.token — Auto-Generated

If you set `"auth": { "mode": "token" }` without providing a token value, OpenClaw generates one on first startup and writes it back to the config file on the PVC. This is why the PVC file differs from the ConfigMap — OpenClaw modifies its own config at runtime.

To retrieve the token:

```bash
oc exec deployment/openclaw -n openclaw -- cat /.openclaw/openclaw.json
```

Look for `gateway.auth.token`. If you prefer to control the token, set it explicitly in the ConfigMap:

```json
"auth": {
  "mode": "token",
  "token": "your-own-secret-token-here"
}
```

## Node.js in the Container Image

OpenClaw is a TypeScript/Node.js project (88.6% TypeScript). The container image `ghcr.io/openclaw/openclaw:latest` is built on top of the official Node.js image. Inside you'll find:

```
/usr/local/bin/
├── node          ← Node.js runtime
├── npm           ← package manager
├── pnpm          ← alternative package manager
├── openclaw      ← the OpenClaw CLI
└── docker-entrypoint.sh
```

When installed locally, you use your machine's Node.js (`npm install -g openclaw@latest`). In the container, everything is bundled.

## Verifying Image Legitimacy

Before applying anything to a cluster, verify the sources are real:

- **GitHub repo**: `github.com/openclaw/openclaw` — 334K+ stars, 360 contributors, MIT license, latest release v2026.3.23
- **Official container image**: `ghcr.io/openclaw/openclaw:latest` (GitHub Container Registry, NOT Docker Hub)
- **Community images exist** on Docker Hub (`fourplayers/openclaw`, `cloudlookup/openclaw`) but stick with the official one
- **Official docs**: `docs.openclaw.ai`

Always verify before applying. The user caught us using a potentially wrong image path during this deployment.

## Security — No Elevated SCCs

Everything runs under OpenShift's default `restricted-v2` SCC:

- Random non-root UID assigned per namespace
- No privilege escalation
- Read-only root filesystem
- All Linux capabilities dropped

No `anyuid`, `privileged`, or custom SCCs needed. The PVC mount at `/.openclaw` provides the writable space the application needs.

## Troubleshooting Guide

### Error 1: Permission denied, mkdir '/.openclaw'

```
Gateway failed to start: Error: EACCES: permission denied, mkdir '/.openclaw'
```

**Cause**: OpenShift's `restricted-v2` SCC forces containers to run as a random non-root UID. This user cannot write to the root filesystem `/`. OpenClaw tries to `mkdir /.openclaw` at startup.

**Fix**: Mount a PVC at `/.openclaw`. When Kubernetes mounts a volume, it creates the directory and sets ownership to the pod's UID. The random user owns the mounted directory and can write freely.

**Why the test pod didn't catch this**: The test pod used `command: ["sleep", "3600"]` which overrides OpenClaw's entrypoint. `sleep` doesn't write to the filesystem, so it ran fine. A better test would run the actual application with the volume mount and no command override.

### Error 2: Liveness probe failed — connection refused

```
Liveness probe failed: Get "http://10.129.2.185:18789/healthz": dial tcp 10.129.2.185:18789: connect: connection refused
Container openclaw failed liveness probe, will be restarted
```

**Cause**: Two overlapping issues.

Issue A — **Bind address**: OpenClaw defaults to `127.0.0.1` (loopback). The probe hits the pod IP (`10.129.2.x`). Connection refused because nobody is listening on that interface.

Issue B — **Timing**: Even if bind was correct, `initialDelaySeconds: 30` is too short. OpenClaw needs over a minute to boot. The liveness probe with `failureThreshold: 3` and `periodSeconds: 10` kills the pod at ~60 seconds (30 + 3×10).

**How to diagnose**: Three pieces of evidence:
1. `Last State: Exit Code: 0` — graceful shutdown, not a crash. Something external killed it.
2. Events show "failed liveness probe, will be restarted" — the kubelet is killing it.
3. Logs show `listening on ws://127.0.0.1:18789` but probes hit pod IP — wrong interface.

**Fix**: Set `"bind": "lan"` in openclaw.json and increase `initialDelaySeconds` to 120.

**failureThreshold math**: `initialDelaySeconds + (failureThreshold × periodSeconds)` = time before kill. With 30 + (3 × 10) = 60s. With 120 + (3 × 10) = 150s. Give the app enough headroom.

### Error 3: gateway.mode=local not set

```
Gateway start blocked: set gateway.mode=local (current: unset) or pass --allow-unconfigured.
```

**Cause**: No openclaw.json config file existed. We were trying to use command-line flags instead.

**Fix**: Create a proper openclaw.json via ConfigMap instead of passing flags one at a time.

### Error 4: controlUi.allowedOrigins not set

```
Gateway failed to start: Error: non-loopback Control UI requires gateway.controlUi.allowedOrigins
```

**Cause**: When binding to non-loopback (lan), the Control UI requires explicit allowed origins for security, or a fallback flag.

**Fix**: Set `"dangerouslyAllowHostHeaderOriginFallback": true` in the gateway.controlUi config. For production, set explicit `allowedOrigins` instead.

### Error 5: Config schema validation failure

```
Config invalid
- identity: identity was moved; use agents.list[].identity instead
- agent: agent.* was moved; use agents.defaults instead
```

**Cause**: OpenClaw v2026.3.23 changed the config schema. The `identity` top-level key moved to `agents.list[].identity`. The `agent` key moved to `agents.defaults`. Many online examples still use the old schema.

**Fix**: Use the current schema. OpenClaw's error messages tell you exactly what to change. Always check the version you're running against the docs.

### Error 6: Route returns "Application is not available"

**Cause 1 — Service missing**: The Service was never created or got deleted. The Route pointed to nothing. Check with `oc get svc -n openclaw`.

**Fix**: Create the Service. The Route needs a backend to forward traffic to.

**Cause 2 — NetworkPolicy blocking the router**: The `deny-from-other-namespaces` NetworkPolicy blocked ALL ingress from other namespaces — including the OpenShift router pods in `openshift-ingress`.

**Fix**: Add an ingress rule allowing traffic from namespaces labeled `network.openshift.io/policy-group: ingress`:

```yaml
ingress:
  - from:
      - podSelector: {}
  - from:
      - namespaceSelector:
          matchLabels:
            network.openshift.io/policy-group: ingress
```

**How to verify**: curl the health endpoint: `curl -sk https://<route-hostname>/healthz` should return `{"ok":true,"status":"live"}`. Also test from inside the pod: `oc exec deployment/openclaw -- curl -s http://127.0.0.1:18789/healthz`.

### Error 7: Device pairing required

**What it is**: OpenClaw requires new devices (browsers) to be approved before they can connect to the Control UI. This is a security feature.

**How to approve**:

```bash
# List pending requests
oc exec deployment/openclaw -n openclaw -- openclaw devices list

# The output shows:
# Request column = UUID (e.g., afafd3c4-bec2-40f8-9ee9-6ff4e6ab588a)
# Device column  = browser fingerprint (long hex string)

# Approve using the UUID from the Request column
oc exec deployment/openclaw -n openclaw -- openclaw devices approve <uuid>
```

**Important**: Pairing requests expire. If you see "unknown requestId" when approving, the request has expired. Go back to the browser, click Connect again, run `devices list` for the new UUID, and approve quickly.

### Error 8: 404 NotFoundError from LiteLLM — model not found

```
404 litellm.NotFoundError: OpenAIException - {"detail":"Not Found"}
Received Model Group=deepseek-r1-distill-qwen-14b
```

**Cause**: The `api` field in the models provider config was set to `openai-responses`, which uses the `/v1/responses` endpoint (newer OpenAI Responses API). LiteLLM only supports the `/v1/chat/completions` endpoint.

**Valid api options**: `openai-completions`, `openai-responses`, `openai-codex-responses`, `anthropic-messages`, `google-generative-ai`, `github-copilot`, `bedrock-converse-stream`, `ollama`

**Fix**: Change `"api": "openai-responses"` to `"api": "openai-completions"` in the ConfigMap. Despite the name, `openai-completions` is the one that uses `/v1/chat/completions`.

**How to verify**: curl the endpoint directly from inside the pod:

```bash
oc exec deployment/openclaw -n openclaw -- curl -s -X POST \
  https://litellm-prod.apps.maas.redhatworkshops.io/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-r1-distill-qwen-14b","messages":[{"role":"user","content":"Hello"}]}'
```

If this returns a valid response but OpenClaw gets 404, the api field is wrong.

### Error 9: "unknown requestId" when approving

```
GatewayClientRequestError: unknown requestId
```

**Cause**: The pairing request UUID has expired or was already handled.

**Fix**: Click Connect in the browser to generate a new request, run `openclaw devices list` to get the new UUID, approve it immediately.

### Error 10: Context overflow — prompt too large for the model

**Cause**: The `deepseek-r1-distill-qwen-14b` model advertised a 4M token context on the MaaS dashboard, but LiteLLM was actually configured with a 16K token limit. OpenClaw's system prompt (~8K tokens) plus a high `maxTokens` request (32,000) exceeded this. The model also leaked internal reasoning with `<think>` tags.

**Fix**: Switch to `llama-scout-17b` which has a verified 327K context window, produces clean responses, and handles the system prompt without overflow. Update the model ID in both `agents.defaults.model.primary` and `models.providers.maas.models[0]`.

### Error 11: Telegram polling 409 Conflict

```
Telegram getUpdates conflict: terminated by other getUpdates request; make sure that only one bot instance is running
```

**Cause**: Another process called `getUpdates` on the same bot token while OpenClaw's long-polling was active. In our case, a diagnostic `curl` to `getUpdates` from inside the pod caused the conflict.

**Fix**: Restart the pod with `oc rollout restart deployment/openclaw -n openclaw`. Never call `getUpdates` manually when OpenClaw is running — use `getWebhookInfo` instead (it's safe and shows `pending_update_count`).

### Error 12: Model responds NO_REPLY to every message

**Symptoms**: Messages arrive in Telegram, the bot shows "typing..." briefly, but no response appears. Session journal files at `/.openclaw/agents/main/sessions/*.jsonl` show the LLM returning `NO_REPLY` for every user message.

**Cause**: OpenClaw's system prompt includes a `## Silent Replies` section that instructs the model: "When you have nothing to say, respond with ONLY: NO_REPLY." Combined with 27 tool JSON schemas (~12,600 tokens) and 7 auto-generated workspace files (~13K characters), the total prompt reaches ~19,000 tokens. Smaller models like `llama-scout-17b` get overwhelmed and default to `NO_REPLY` for everything.

**How to diagnose**: Read the session journal:

```bash
oc exec deployment/openclaw -c openclaw -n openclaw -- \
  cat /.openclaw/agents/main/sessions/*.jsonl | tail -5
```

Look for `"text":"NO_REPLY"` in assistant messages. Confirm with A/B testing:

```bash
# With tools → NO_REPLY
oc exec deployment/openclaw -n openclaw -- curl -s -X POST \
  "$MAAS_URL/v1/chat/completions" \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"llama-scout-17b","messages":[{"role":"system","content":"...full prompt..."}],"tools":[...27 tools...]}'

# Without tools → "Hello! How can I help?"
# Same request but without the tools array
```

**Fix (two parts)**:

1. Restrict tools: Add `"tools": {"allow": ["web_search", "web_fetch", "session_status"]}` to the agent in `openclaw.json`. This drops from 27 tools (~12,600 tokens) to 3.

2. Trim workspace files: Replace the auto-generated files in `/.openclaw/workspace/` with minimal versions. The default `AGENTS.md` alone is 7,874 characters of group-chat and heartbeat instructions irrelevant for a simple Telegram bot.

3. Clear old sessions: `oc exec deployment/openclaw -n openclaw -- sh -c 'rm -rf /.openclaw/agents/main/sessions/*'` — old sessions contain accumulated `NO_REPLY` history that the model follows as a pattern.

## OpenClaw Auto-Generated Workspace Files

On first boot, OpenClaw creates 7 template files in `/.openclaw/workspace/`. These are NOT from the ConfigMap — they're built into the Docker image and generated by OpenClaw's bootstrap process.

| File | Default Size | Purpose | Injected into System Prompt? |
|------|-------------|---------|------------------------------|
| AGENTS.md | 7,874 chars | Master instructions: group chat rules, heartbeats, memory, when to stay silent | Yes |
| SOUL.md | 1,673 chars | Personality essay ("you're becoming someone") | Yes |
| BOOTSTRAP.md | 1,471 chars | First-run conversation script | Yes |
| TOOLS.md | 860 chars | Template for camera names, SSH hosts, TTS prefs | Yes |
| IDENTITY.md | 636 chars | Template for name, emoji, vibe | Yes |
| USER.md | 477 chars | Template for learning about "your human" | Yes |
| HEARTBEAT.md | 193 chars | Periodic check template | Yes |

**All 7 files are injected into the system prompt on every LLM call.** That's 13,184 characters of instructions. For GPT-5 this is fine. For `llama-scout-17b`, it's too much.

**Who manages them?** You do. They're designed to be edited by the user or by the agent itself. The workspace even has a `.git` directory — OpenClaw initializes a git repo for version tracking.

**Do they survive restarts?** Yes — they live on the PVC. They only regenerate if the PVC is deleted.

**Our trimmed versions:** We replaced all files with minimal content (668 chars total, 95% reduction), keeping only essential instructions like "always reply to the user."

## Telegram Integration Details

**Bot creation**: Message @BotFather on Telegram → `/newbot` → choose name and username → receive bot token.

**User ID**: Message @userinfobot → receive your numeric Telegram ID.

**How Telegram connects**: OpenClaw uses long-polling (`getUpdates`), not webhooks. It opens an outbound HTTPS connection to `api.telegram.org` and keeps it open. No inbound ports, no public endpoints needed. The egress NetworkPolicy must allow traffic to the internet.

**Sending messages via API** (useful for testing):

```bash
oc exec deployment/openclaw -n openclaw -- curl -s \
  "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<YOUR_ID>" -d "text=Test message"
```

**Safe diagnostic** (won't break polling):

```bash
oc exec deployment/openclaw -n openclaw -- curl -s \
  "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

This shows `pending_update_count` — if messages are queuing up, polling is broken.

**Unsafe diagnostic** (WILL break polling):

```bash
# DO NOT run while OpenClaw is active — causes 409 Conflict
curl "https://api.telegram.org/bot<TOKEN>/getUpdates"
```

## PVC vs emptyDir — What Survives Restarts

**PVC (PersistentVolumeClaim)** — data survives restarts. The PVC is a separate disk that lives independently. New pod mounts the same PVC and finds everything intact. Used for the real deployment.

**emptyDir** — data is lost on restart. Tied to the pod's lifecycle. Good for test pods, bad for real deployments.

## UID Consistency on OpenShift

OpenShift assigns each namespace a fixed UID range (e.g., `1000680000/10000`). Check with:

```bash
oc get namespace openclaw -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'
```

Every pod in the namespace gets a UID from that range. For a single-container pod, it consistently gets the first UID. So across restarts, the UID stays the same — PVC data remains accessible.

Even if the UID changed, `fsGroup` ensures group-level permissions keep the volume accessible.

## Init Container Pattern

The init container copies config from a ConfigMap (read-only) into the PVC (read-write) before the main container starts. This is necessary because:

1. ConfigMaps are mounted read-only in Kubernetes
2. OpenClaw modifies its own config at runtime (adds gateway token, updates metadata)
3. The PVC provides the writable location for the live config

Flow:

```
ConfigMap (seed config, no token)
    ↓ init container copies to PVC
PVC: /.openclaw/openclaw.json (still no token)
    ↓ OpenClaw starts, reads config
    ↓ auto-generates gateway token
    ↓ writes token back to PVC
PVC: /.openclaw/openclaw.json (now has token + meta fields)
```

The PVC file is the live config. The ConfigMap is the initial seed.

## Recreate Deployment Strategy

The Deployment uses `strategy: Recreate` instead of `RollingUpdate` because the PVC has `accessMode: ReadWriteOnce`. Only one pod can mount a RWO PVC at a time. With `RollingUpdate`, the new pod would try to mount the PVC while the old pod still has it — and fail. `Recreate` kills the old pod first, then starts the new one.

## Security Best Practices

- **Never share cluster credentials in plain text** — rotate passwords if shared accidentally
- **Never share API keys in chat** — rotate keys if exposed
- **Use Secrets for sensitive data** — never put API keys in ConfigMaps
- **Set explicit controlUi.allowedOrigins** in production instead of `dangerouslyAllowHostHeaderOriginFallback`
- **Verify container image sources** before pulling — use `ghcr.io/openclaw/openclaw` (official) not random Docker Hub images
- **NetworkPolicies are not optional** — by default any pod can talk to any other pod across namespaces

## Cluster Details

- **Platform**: ROSA (Red Hat OpenShift Service on AWS)
- **API**: `https://api.rosaacs.lnna.p3.openshiftapps.com:443`
- **Route URL**: `https://openclaw-openclaw.apps.rosa.rosaacs.lnna.p3.openshiftapps.com/`
- **MaaS Endpoint**: `https://litellm-prod.apps.maas.redhatworkshops.io/v1`
- **Model**: `llama-scout-17b` (switched from `deepseek-r1-distill-qwen-14b` due to context overflow)
- **OpenClaw Version**: v2026.3.23
- **Image**: `ghcr.io/openclaw/openclaw:latest`
- **Telegram Bot**: @Claw_nj_bot
- **Tools enabled**: `web_search`, `web_fetch`, `session_status` (3 of 27)

## What's Next

- Add Slack (chat + alerts channels)
- Set up cron jobs for scheduled news summaries
- Configure web search (Brave API key needed, tool already enabled)
- Build the Second Brain with markdown notes + nomic embeddings for semantic search
- Add Llama-Guard safety filter
- Add Docling for document conversion
- Gradually enable more tools as the model proves capable
