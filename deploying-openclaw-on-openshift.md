# Deploying OpenClaw on OpenShift with Model as a Service

This guide walks through deploying OpenClaw — an open-source personal AI assistant framework — on Red Hat OpenShift, using an internally hosted LLM accessed via API and Telegram as the messaging channel.

## What is OpenClaw?

OpenClaw is an open-source personal AI assistant framework built with Node.js and TypeScript. You install it locally with `npm install -g openclaw@latest`, run the onboard wizard, and it connects to messaging platforms like Telegram, Slack, WhatsApp, Discord, and more.

The onboard wizard creates a configuration file called `openclaw.json` that controls everything the agent does. On Kubernetes, there's no interactive wizard — so we create that config manually.

## Why OpenShift?

Running OpenClaw on a local machine ties it to a single device. If that machine goes down, the assistant goes with it. OpenShift gives you automatic restarts via health probes, persistent storage via PVCs, network isolation via NetworkPolicies, and the ability to manage everything declaratively with YAML. No dedicated hardware needed.

## The Model

This deployment uses **llama-scout-17b** (Llama 4 Scout), hosted internally on a separate OpenShift cluster and exposed through a LiteLLM proxy as Model as a Service (MaaS). We access it via a standard OpenAI-compatible API endpoint (`/v1/chat/completions`) — OpenClaw doesn't know or care that it's not talking to OpenAI.

Key properties:

- **17B parameters** — sufficient for agent tasks
- **Function calling support** — required for OpenClaw to trigger tools like web search
- **327K token context window** — large enough to handle the system prompt and conversation history
- **Clean output** — no reasoning leakage or internal tags in responses
- **~2 second inference time** — fast enough for conversational use via Telegram

## Understanding openclaw.json

Before diving into manifests, it helps to understand what `openclaw.json` controls. It's the single master configuration file for the entire system:

- **gateway** — how the server runs (port, bind address, authentication)
- **agents.defaults** — the infrastructure layer shared by all agents (which model, workspace path, timeouts). Think of this as the "plumbing."
- **agents.list** — the agent definitions (name, personality, behavior overrides, tool restrictions). Think of this as "who are the agents."
- **models.providers** — where the AI brains come from. Your MaaS endpoint, OpenAI, Anthropic, or local models.
- **channels** — which messaging platforms to connect (Telegram, Slack, WhatsApp, etc.) with tokens and access control
- **tools** — what the agent is allowed to do (shell commands, file read/write, browser)
- **skills** — extra plugins from ClawHub (community skills)
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

```
oc apply -f manifests/01-namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw
```

The **ingress policy** allows traffic only from pods within the same namespace and from the OpenShift router (so Routes can deliver traffic). Everything else is blocked.

```
oc apply -f manifests/02-networkpolicy-deny-ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: openclaw
spec:
  podSelector: {}
  ingress:
    - from:
        - podSelector: {}
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
  policyTypes:
    - Ingress
```

The **egress policy** allows outbound traffic to the internet (for Telegram, MaaS APIs) and cluster DNS, but blocks all private IP ranges. OpenClaw cannot reach pods in other namespaces.

```
oc apply -f manifests/03-networkpolicy-allow-egress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress-only
  namespace: openclaw
spec:
  podSelector: {}
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-dns
      ports:
        - protocol: UDP
          port: 5353
        - protocol: TCP
          port: 5353
  policyTypes:
    - Egress
```

Notice OpenShift DNS runs on port 5353, not the standard 53. The egress rule explicitly allows DNS resolution while blocking all internal cluster traffic.

## Step 2 — Secret for API Keys

The Secret stores your MaaS API key, MaaS endpoint URL, and Telegram bot token. These get injected as environment variables into the pod. The `openclaw.json` references `${OPENAI_API_KEY}` and `${TELEGRAM_BOT_TOKEN}` which OpenClaw resolves from the environment at runtime.

```
oc apply -f manifests/04-secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openclaw-secrets
  namespace: openclaw
type: Opaque
stringData:
  OPENAI_API_KEY: "your-litellm-api-key-here"
  OPENAI_BASE_URL: "https://your-litellm-endpoint/v1"
  TELEGRAM_BOT_TOKEN: "your-telegram-bot-token-here"
```

Replace the placeholder values with your real keys before applying. `stringData` accepts plain text — Kubernetes base64-encodes it automatically.

## Step 3 — Persistent Storage

A 10Gi PVC mounted at `/.openclaw` inside the container. This is where OpenClaw stores its configuration, workspace files, conversation history, and session data. The PVC persists across pod restarts.

```
oc apply -f manifests/05-pvc.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openclaw-data
  namespace: openclaw
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

On OpenShift, the PVC is owned by the pod's UID. OpenShift assigns a consistent UID per namespace, so data remains accessible even after restarts.

`ReadWriteOnce` means only one pod can mount it at a time — this is why the Deployment uses `strategy: Recreate` instead of `RollingUpdate`.

## Step 4 — ConfigMap with openclaw.json

Since there's no interactive onboard wizard in Kubernetes, we create the config manually in a ConfigMap. This is the full working configuration:

```
oc apply -f manifests/06-configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: openclaw-config
  namespace: openclaw
data:
  openclaw.json: |
    {
      "gateway": {
        "mode": "local",
        "port": 18789,
        "bind": "lan",
        "controlUi": {
          "enabled": true,
          "dangerouslyAllowHostHeaderOriginFallback": true
        },
        "auth": {
          "mode": "token"
        }
      },
      "agents": {
        "defaults": {
          "workspace": "/.openclaw/workspace",
          "model": {
            "primary": "maas/llama-scout-17b"
          }
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
  AGENTS.md: |
    You are MyClaw, a helpful personal AI assistant running on OpenShift.
    You MUST always reply to the user. Never respond with NO_REPLY to a direct user message.
    If a user says hello, greet them back. If they ask a question, answer it.
    Be concise and friendly.
```

Key settings:

- **bind: "lan"** — makes the gateway listen on `0.0.0.0` (all interfaces). In Kubernetes, the kubelet probes, Service, and Route all hit the pod IP, not localhost. Without `lan`, the gateway only listens on `127.0.0.1` and nothing outside the container can reach it.
- **dangerouslyAllowHostHeaderOriginFallback: true** — this is a CORS (Cross-Origin Resource Sharing) setting for the Control UI. When a browser opens the Control UI, it sends an `Origin` header (e.g., `https://openclaw-openclaw.apps.rosa...`). OpenClaw validates this Origin against a whitelist to prevent cross-site request forgery (CSRF) attacks. When binding to `lan` (0.0.0.0), OpenClaw enforces this strictly — it refuses to start unless you either provide an explicit `allowedOrigins` list or set this fallback flag. With the fallback enabled, OpenClaw trusts the `Host` header from the incoming request as the allowed origin. This is acceptable for internal or development deployments behind an OpenShift Route. For production, replace this with `"allowedOrigins": ["https://your-route-hostname"]` so only your specific Route URL is trusted.
- **tools.allow** — critical for smaller models. OpenClaw ships 27 tools by default, adding ~12,600 tokens of JSON schemas to every API call. For models like `llama-scout-17b`, this overwhelms the prompt. Restricting to 3 essential tools keeps the prompt manageable.
- **api: "openai-completions"** — despite the name, this is the setting for `/v1/chat/completions`. Don't use `openai-responses` — that calls `/v1/responses` which LiteLLM doesn't support.
- **channels.telegram.allowFrom** — an array of numeric Telegram user IDs. Only these users can talk to the bot.
- **AGENTS.md** — the system prompt seed. The explicit "Never respond with NO_REPLY" instruction is critical for smaller models (explained later).

## Step 5 — The Deployment

The Deployment uses a standard Kubernetes pattern called **config seeding with an init container**. Here's why it's needed:

OpenClaw expects its config file at `/.openclaw/openclaw.json` and it **writes back to that file** at runtime — it auto-generates a gateway token, adds metadata, and updates internal state. In Kubernetes, ConfigMaps are mounted **read-only**. If we mount the ConfigMap directly at `/.openclaw/openclaw.json`, OpenClaw crashes when it tries to write to it.

The solution is a two-step process. First, an init container runs, copies the config from the read-only ConfigMap mount into the writable PVC. Then it exits. Second, the main container starts and finds a writable copy of the config on the PVC. It reads it, modifies it as needed, and runs normally.

This is the same pattern used by databases (PostgreSQL, MySQL), content management systems (WordPress), and any application that needs to modify its own configuration. It's standard Kubernetes — not specific to OpenClaw.

**Init container** — a lightweight UBI image that runs `cp` and exits. It has access to both the ConfigMap volume (read-only source) and the PVC volume (writable destination).

**Main container** — runs OpenClaw. It finds the config at `/.openclaw/openclaw.json`, starts the gateway, connects to Telegram via long-polling, and begins listening.

```
oc apply -f manifests/07-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw
  namespace: openclaw
  labels:
    app: openclaw
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: openclaw
  template:
    metadata:
      labels:
        app: openclaw
    spec:
      initContainers:
        - name: copy-config
          image: registry.access.redhat.com/ubi9/ubi-minimal:latest
          command:
            - sh
            - -c
            - |
              cp /config/openclaw.json /.openclaw/openclaw.json
              cp /config/AGENTS.md /.openclaw/AGENTS.md
          volumeMounts:
            - name: openclaw-data
              mountPath: /.openclaw
            - name: openclaw-config
              mountPath: /config
      containers:
        - name: openclaw
          image: ghcr.io/openclaw/openclaw:latest
          ports:
            - containerPort: 18789
              name: gateway
          envFrom:
            - secretRef:
                name: openclaw-secrets
          volumeMounts:
            - name: openclaw-data
              mountPath: /.openclaw
          livenessProbe:
            httpGet:
              path: /healthz
              port: gateway
            initialDelaySeconds: 120
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: gateway
            initialDelaySeconds: 30
            periodSeconds: 5
          resources:
            requests:
              memory: "2Gi"
              cpu: "500m"
            limits:
              memory: "4Gi"
              cpu: "2"
      volumes:
        - name: openclaw-data
          persistentVolumeClaim:
            claimName: openclaw-data
        - name: openclaw-config
          configMap:
            name: openclaw-config
```

Important details:

- **strategy: Recreate** — the PVC is `ReadWriteOnce`, so only one pod can mount it at a time. `Recreate` ensures the old pod is terminated before the new one starts.
- **livenessProbe initialDelaySeconds: 120** — OpenClaw takes over a minute to fully boot. The liveness probe waits 2 minutes before its first check. Too short and the probe kills the pod before it's ready.
- **envFrom: secretRef** — injects ALL keys from the Secret as environment variables. OpenClaw resolves `${OPENAI_API_KEY}` and `${TELEGRAM_BOT_TOKEN}` from these.
- **resources: 2Gi RAM request, 4Gi limit** — sufficient for the gateway and agent processing.
- **No securityContext needed** — everything runs under OpenShift's default `restricted-v2` SCC. No `anyuid`, no `privileged`, no custom SCCs.

## Step 6 — Service and Route

A ClusterIP Service on port 18789 gives the pod a stable internal DNS name.

```
oc apply -f manifests/08-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: openclaw
  namespace: openclaw
  labels:
    app: openclaw
spec:
  selector:
    app: openclaw
  ports:
    - port: 18789
      targetPort: gateway
      name: gateway
```

A Route with TLS edge termination exposes it externally via HTTPS for the Control UI.

```
oc apply -f manifests/09-route.yaml
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: openclaw
  namespace: openclaw
spec:
  to:
    kind: Service
    name: openclaw
  port:
    targetPort: gateway
  tls:
    termination: edge
```

Verify the Route is working:

```
oc get route openclaw -n openclaw -o jsonpath='{.spec.host}'
curl -sk https://$(oc get route openclaw -n openclaw -o jsonpath='{.spec.host}')/healthz
```

You should see `{"ok":true,"status":"live"}`.

## Step 7 — Access the Control UI

After deployment, OpenClaw auto-generates a gateway token and saves it to the config file on the PVC. Retrieve it with:

```
oc exec deployment/openclaw -n openclaw -- cat /.openclaw/openclaw.json
```

Look for `gateway.auth.token` in the output. Open the Route URL in your browser and paste the token to authenticate.

The first time you connect, the Control UI shows "pairing required." OpenClaw requires new devices to be approved:

```
oc exec deployment/openclaw -n openclaw -- openclaw devices list
```

You'll see a table with a "Pending" section. The "Request" column contains the UUID. Approve it:

```
oc exec deployment/openclaw -n openclaw -- openclaw devices approve <uuid-from-request-column>
```

Refresh the browser after approving — you're in. If you see "unknown requestId", the pairing request expired. Go back to the browser, click Connect again, re-run `devices list`, and approve the new UUID quickly.

## Step 8 — Telegram Integration

Telegram is the easiest messaging channel to set up because it uses a simple bot token — no QR codes, no phone linking, no Apple device required.

**Create a bot:** Message @BotFather on Telegram, send `/newbot`, choose a name and username. BotFather gives you a token like `1234567890:ABCdefGhIjKlMnOpQrStUvWxYz`.

**Get your Telegram user ID:** Message @userinfobot on Telegram — it replies with your numeric ID (e.g., `987654321`). This goes in the `allowFrom` array in the ConfigMap so only you can talk to the bot.

**Update the Secret and ConfigMap:** Put the bot token in `04-secret.yaml` as `TELEGRAM_BOT_TOKEN`. Put your numeric user ID in `06-configmap.yaml` in the `allowFrom` array. Apply both and restart:

```
oc apply -f manifests/04-secret.yaml
oc apply -f manifests/06-configmap.yaml
oc rollout restart deployment/openclaw -n openclaw
```

**How it works under the hood:** OpenClaw uses Telegram's long-polling API (`getUpdates`). It keeps an HTTP connection open to `api.telegram.org`, waiting for new messages. When you send "Hello" in Telegram, the flow is:

1. You type "Hello" → Telegram servers queue it
2. OpenClaw's long-poll receives it via `getUpdates`
3. OpenClaw wraps it with metadata and sends it to `llama-scout-17b`
4. The model responds
5. OpenClaw calls `sendMessage` on the Telegram API to deliver the reply

No webhooks, no public endpoints needed. The bot initiates all connections outbound — perfect for a pod behind NetworkPolicies.

**Verify Telegram connectivity from inside the pod:**

```
oc exec deployment/openclaw -n openclaw -- curl -s \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getWebhookInfo"
```

This is a safe diagnostic that won't interfere with polling. If `pending_update_count` is growing, messages are queuing up and something is wrong with the bot's polling loop.

## The Tool Token Problem — Why Smaller Models Need Restrictions

This was the hardest issue to debug. After configuring Telegram, messages were delivered to the bot but the LLM responded with `NO_REPLY` to everything — "Hello", "How are you?", even simple greetings.

The investigation revealed the root cause through A/B testing the actual API calls:

- **With 27 tools** (OpenClaw default) → ~19,000 prompt tokens → model says `NO_REPLY`
- **Without tools** → ~6,400 prompt tokens → model says "Hello! How can I help?"

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

These files live on the PVC, not in the ConfigMap. They survive pod restarts. You can edit them freely — they're meant to be customized by you or by the agent itself over time. To trim them:

```
oc exec deployment/openclaw -n openclaw -- sh -c 'cat > /.openclaw/workspace/AGENTS.md << "EOF"
# MyClaw - Personal Assistant
You are MyClaw, a helpful personal AI assistant running on OpenShift.
## Rules
- Always reply to the user. Never respond with NO_REPLY to a direct user message.
- If a user says hello, greet them back warmly.
- If they ask a question, answer it.
- Be concise and friendly.
- You can search the web if needed.
EOF'
```

Repeat for the other files with minimal content, or delete the ones you don't need (like `BOOTSTRAP.md`).

## What's Next

The full loop is working: Telegram → OpenClaw → llama-scout-17b → Telegram. Next steps:

- Add Slack with separate chat and alerts channels
- Set up cron jobs for scheduled news summaries
- Configure the web search skill (the tool is enabled, just needs a Brave API key)
- Build the Second Brain with markdown notes and semantic search using the free nomic embedding model
- Add Llama-Guard safety filter
- Try enabling more tools as the model proves capable

All manifests are in the [manifests/ directory on GitHub](https://github.com/nirjhar17/openclaw-on-openshift), numbered in apply order. Full troubleshooting guide and detailed explanations are in the README.

> **Disclaimer:** This article reflects a personal learning exercise. OpenClaw is a fictional project name used for demonstration purposes. The Kubernetes manifests, patterns, and debugging approaches described here are real and applicable to deploying any Node.js-based agent framework on OpenShift. Model names, API endpoints, and configurations may differ in your environment. Always verify container images and review security settings before deploying to production clusters.

*Written by Nirjhar Jajodia — Platform Engineer exploring AI agent deployment on Kubernetes. Connect with me on [GitHub](https://github.com/nirjhar17).*
