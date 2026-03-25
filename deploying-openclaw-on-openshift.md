# Deploying OpenClaw on OpenShift with Model as a Service

What if your AI personal assistant didn't live on your laptop but ran as a pod on Kubernetes — restarting itself, isolating its network traffic, and persisting its memory across crashes? That's exactly what this guide covers.

We deploy OpenClaw, an open-source AI assistant framework, on Red Hat OpenShift. It connects to Telegram, talks to an internally hosted LLM via API, and runs without any elevated security privileges.

## What is OpenClaw?

OpenClaw is an open-source personal AI assistant framework built with Node.js and TypeScript. You install it locally with `npm install -g openclaw@latest`, run the onboard wizard, and it connects to messaging platforms like Telegram, Slack, WhatsApp, Discord, and more.

The onboard wizard creates a configuration file called `openclaw.json` that controls everything the agent does. On Kubernetes, there's no interactive wizard — so we create that config manually.

## Why OpenShift?

Running OpenClaw on a local machine ties it to a single device. If that machine goes down, the assistant goes with it.

OpenShift gives you automatic restarts via health probes, persistent storage via PVCs, network isolation via NetworkPolicies, and the ability to manage everything declaratively with YAML. No dedicated hardware needed.

## The Model

This deployment uses **llama-scout-17b** (Llama 4 Scout), hosted internally on a separate OpenShift cluster and exposed through a LiteLLM proxy as Model as a Service (MaaS). We access it via a standard OpenAI-compatible API endpoint (`/v1/chat/completions`).

- **17B parameters** — sufficient for agent tasks
- **Function calling support** — required for OpenClaw to trigger tools like web search
- **327K token context window** — large enough to handle the system prompt and conversation history
- **Clean output** — no reasoning leakage or internal tags in responses
- **~2 second inference time** — fast enough for conversational use via Telegram

## Understanding openclaw.json

Before diving into manifests, it helps to understand what `openclaw.json` controls. It's the single master configuration file for the entire system.

- **gateway** — how the server runs (port, bind address, authentication)
- **agents.defaults** — the infrastructure layer shared by all agents (which model, workspace path, timeouts)
- **agents.list** — the agent definitions (name, personality, behavior overrides, tool restrictions)
- **models.providers** — where the AI brains come from (MaaS endpoint, OpenAI, Anthropic, or local models)
- **channels** — which messaging platforms to connect (Telegram, Slack, WhatsApp) with tokens and access control
- **tools** — what the agent is allowed to do (shell commands, file read/write, browser)
- **cron** — scheduled tasks (news summaries every 3 hours)
- **hooks** — external event triggers (Gmail webhooks, GitHub notifications)

There's also a file called **AGENTS.md** — the system prompt injected into every LLM call. OpenClaw also auto-generates several workspace files (`SOUL.md`, `IDENTITY.md`, `USER.md`, `BOOTSTRAP.md`, `TOOLS.md`, `HEARTBEAT.md`) that all get injected too. More on that later.

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

All 9 manifests are available in the [GitHub repo](https://github.com/nirjhar17/openclaw-on-openshift) and as a [Gist](https://gist.github.com/nirjhar17/f11b29a6587404e192f34074189c1b91).

## Step 1 — Namespace and Network Isolation

Create a namespace and lock it down with two NetworkPolicies.

The **ingress policy** allows traffic only from pods within the same namespace and from the OpenShift router (so Routes can deliver traffic). The key label is `network.openshift.io/policy-group: ingress` — this tells OpenShift to let router traffic through.

The **egress policy** allows outbound traffic to the internet (for Telegram, MaaS APIs) and cluster DNS on port 5353 (OpenShift uses 5353, not the standard 53). It blocks all private IP ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), so OpenClaw cannot reach pods in other namespaces.

```
oc apply -f manifests/01-namespace.yaml
oc apply -f manifests/02-networkpolicy-deny-ingress.yaml
oc apply -f manifests/03-networkpolicy-allow-egress.yaml
```

## Step 2 — Secret for API Keys

The Secret stores your MaaS API key, endpoint URL, and Telegram bot token. These get injected as environment variables into the pod. The `openclaw.json` references `${OPENAI_API_KEY}` and `${TELEGRAM_BOT_TOKEN}` which OpenClaw resolves from the environment at runtime.

```
oc apply -f manifests/04-secret.yaml
```

Use `stringData` instead of `data` — Kubernetes base64-encodes it automatically. Replace placeholder values with your real keys before applying.

## Step 3 — Persistent Storage

A 10Gi PVC mounted at `/.openclaw` inside the container. This is where OpenClaw stores its configuration, workspace files, conversation history, and session data.

```
oc apply -f manifests/05-pvc.yaml
```

On OpenShift, the PVC is owned by the pod's UID. OpenShift assigns a consistent UID per namespace, so data remains accessible even after restarts. `ReadWriteOnce` means only one pod can mount it at a time — this is why the Deployment uses `strategy: Recreate`.

## Step 4 — ConfigMap with openclaw.json

Since there's no interactive onboard wizard in Kubernetes, we create the config manually in a ConfigMap. Key settings worth calling out:

**bind: "lan"** — makes the gateway listen on `0.0.0.0` (all interfaces). Without this, the gateway only listens on `127.0.0.1` and Kubernetes probes, Services, and Routes all get connection refused.

**dangerouslyAllowHostHeaderOriginFallback: true** — this is a CORS setting for the Control UI. When a browser opens the Control UI, it sends an `Origin` header. OpenClaw validates this against a whitelist to prevent CSRF attacks. When binding to `lan`, OpenClaw refuses to start unless you either provide an explicit `allowedOrigins` list or set this fallback flag. With the fallback enabled, OpenClaw trusts the `Host` header from the incoming request as the allowed origin. This is fine for internal deployments. For production, use `"allowedOrigins": ["https://your-route-hostname"]` instead.

**tools.allow** — critical for smaller models. OpenClaw ships 27 tools by default, adding ~12,600 tokens of JSON schemas to every API call. Restricting to 3 essential tools keeps the prompt manageable.

**api: "openai-completions"** — despite the name, this is the setting for `/v1/chat/completions`. Don't use `openai-responses` — that calls `/v1/responses` which LiteLLM doesn't support.

```
oc apply -f manifests/06-configmap.yaml
```

See the full ConfigMap in the [GitHub repo](https://github.com/nirjhar17/openclaw-on-openshift/blob/main/manifests/06-configmap.yaml).

## Step 5 — The Deployment

The Deployment uses a standard Kubernetes pattern called **config seeding with an init container**.

OpenClaw expects its config file at `/.openclaw/openclaw.json` and it **writes back to that file** at runtime — it auto-generates a gateway token, adds metadata, and updates internal state. In Kubernetes, ConfigMaps are mounted read-only. If we mount the ConfigMap directly, OpenClaw crashes when it tries to write.

The solution is a two-step process. First, an init container copies the config from the read-only ConfigMap into the writable PVC. Then it exits. Second, the main container starts and finds a writable copy of the config on the PVC. This is the same pattern used by databases (PostgreSQL, MySQL) and CMS platforms (WordPress) — standard Kubernetes, not specific to OpenClaw.

Important deployment settings:

- **strategy: Recreate** — the PVC is ReadWriteOnce, so the old pod must terminate before the new one starts
- **livenessProbe initialDelaySeconds: 120** — OpenClaw takes over a minute to boot. Too short and the probe kills the pod before it's ready
- **envFrom: secretRef** — injects all Secret keys as environment variables
- **No securityContext needed** — everything runs under OpenShift's default restricted-v2 SCC

```
oc apply -f manifests/07-deployment.yaml
```

See the full Deployment in the [GitHub repo](https://github.com/nirjhar17/openclaw-on-openshift/blob/main/manifests/07-deployment.yaml).

## Step 6 — Service and Route

A ClusterIP Service on port 18789 gives the pod a stable internal DNS name. A Route with TLS edge termination exposes it externally via HTTPS for the Control UI.

```
oc apply -f manifests/08-service.yaml
oc apply -f manifests/09-route.yaml
```

Verify with:

```
curl -sk https://$(oc get route openclaw -n openclaw -o jsonpath='{.spec.host}')/healthz
```

You should see `{"ok":true,"status":"live"}`.

## Step 7 — Access the Control UI

OpenClaw auto-generates a gateway token on first boot and saves it to the config file on the PVC. Retrieve it:

```
oc exec deployment/openclaw -n openclaw -- cat /.openclaw/openclaw.json
```

Look for `gateway.auth.token`. Open the Route URL in your browser and paste the token.

The first time you connect, the Control UI shows "pairing required." OpenClaw requires new browser devices to be approved:

```
oc exec deployment/openclaw -n openclaw -- openclaw devices list
oc exec deployment/openclaw -n openclaw -- openclaw devices approve <uuid>
```

The UUID comes from the "Request" column. If you see "unknown requestId", the pairing request expired — click Connect again in the browser and approve the new UUID quickly.

## Step 8 — Telegram Integration

Telegram is the easiest channel because it uses a simple bot token — no QR codes, no phone linking.

**Create a bot:** Message @BotFather on Telegram, send `/newbot`, choose a name. You'll receive a token like `1234567890:ABCdefGhIjKlMnOpQrStUvWxYz`.

**Get your user ID:** Message @userinfobot — it replies with your numeric ID. This goes in the `allowFrom` array so only you can talk to the bot.

**Apply and restart:**

```
oc apply -f manifests/04-secret.yaml
oc apply -f manifests/06-configmap.yaml
oc rollout restart deployment/openclaw -n openclaw
```

OpenClaw uses Telegram's long-polling API (`getUpdates`). It keeps an outbound HTTPS connection open to `api.telegram.org`, waiting for messages. No webhooks, no inbound ports needed — perfect for a pod behind NetworkPolicies.

The message flow: You type "Hello" → Telegram queues it → OpenClaw's poll receives it → wraps with metadata → sends to llama-scout-17b → model responds → OpenClaw calls `sendMessage` → reply appears in Telegram.

## The Tool Token Problem

This was the hardest issue to debug. After configuring Telegram, messages arrived but the LLM responded with `NO_REPLY` to everything.

A/B testing the API calls revealed the root cause:

- **With 27 tools** (default) → ~19,000 prompt tokens → model says `NO_REPLY`
- **Without tools** → ~6,400 prompt tokens → model says "Hello! How can I help?"

OpenClaw ships 27 tools by default. Each includes a full JSON schema in the API call, adding ~12,600 tokens. Combined with the system prompt and a `## Silent Replies` instruction that tells the model to say `NO_REPLY` when it has nothing to add, smaller models get overwhelmed and default to silence.

The fix: `"tools": {"allow": ["web_search", "web_fetch", "session_status"]}`. This drops from 27 tools to 3. GPT-5 or Claude wouldn't need this — but for smaller open-source models, tool restriction is essential.

## Auto-Generated Workspace Files

On first boot, OpenClaw creates 7 template files in `/.openclaw/workspace/` — totaling 13,184 characters. All get injected into the system prompt on every LLM call.

- **AGENTS.md** (7,874 chars) — group chat behavior, heartbeats, when to stay silent
- **SOUL.md** (1,673 chars) — personality essay
- **BOOTSTRAP.md** (1,471 chars) — first-run conversation script
- **TOOLS.md** (860 chars) — local tool notes template
- **IDENTITY.md** (636 chars) — name, emoji, vibe template
- **USER.md** (477 chars) — learning about the human
- **HEARTBEAT.md** (193 chars) — periodic check template

These are designed for GPT-5-class models. For llama-scout-17b, I trimmed them to 668 characters total. They live on the PVC, survive restarts, and are meant to be customized over time.

## What's Next

The full loop is working: Telegram → OpenClaw → llama-scout-17b → Telegram.

- Add Slack with separate chat and alerts channels
- Set up cron jobs for scheduled news summaries
- Configure web search (tool is enabled, just needs a Brave API key)
- Build the Second Brain with markdown notes and nomic embeddings
- Add Llama-Guard safety filter
- Gradually enable more tools as the model proves capable

All manifests are in the [GitHub repo](https://github.com/nirjhar17/openclaw-on-openshift), numbered in apply order. The README has a full troubleshooting guide covering 12 errors we encountered along the way.

## About Me

I work on OpenShift, OpenShift AI, and observability solutions, focusing on simplifying complex setups into practical, repeatable steps for platform and development teams.

GitHub: [github.com/nirjhar17](https://github.com/nirjhar17)

LinkedIn: [linkedin.com/in/nirjhar-jajodia](https://linkedin.com/in/nirjhar-jajodia)

## Disclaimer

The views and opinions expressed in this article are my own and do not necessarily reflect the official policy or position of my employer. This guide is provided for educational purposes, and I make no warranties about the completeness, reliability, or accuracy of this information.
