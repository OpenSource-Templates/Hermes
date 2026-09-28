# Deploy and Host Hermes Agent on Railway

Hermes Agent is an autonomous, self-improving AI agent from Nous Research. It lives on your infrastructure, talks to you over messaging platforms (Telegram, Discord, Slack, and more), runs tools and terminals, and grows more capable the longer it runs. This template deploys the official `nousresearch/hermes-agent` Docker image on Railway as a worker service with persistent state under `/data`, and always defaults to the `latest` image tag.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/new/template/hermes-telegram)

> Replace the deploy link above with your published Railway template URL once available.

## About Hosting Hermes Agent

Hosting Hermes Agent on Railway runs a single container using the official image `nousresearch/hermes-agent:latest`. A persistent volume is mounted at `/data` so configuration, sessions, pairing state, logs, and the default workspace survive redeploys. The container starts `hermes gateway`, which connects to your chosen messaging platforms and inference providers.

The web/admin surface is not the primary interface — you interact with the agent over Telegram, Discord, Slack, or other supported platforms. Railway SSH is available for status checks and one-off `hermes` commands.

**Important Railway notes:** This is a long-running worker, not a classic web app. There is no public HTTP port required for normal operation. Persist everything under `/data` (the template sets `HERMES_HOME=/data/.hermes` and uses `/data/workspace` as the default terminal working directory). Auto-updates inside the container are not used; upgrade by redeploying so you always pull a fresh official image.

## Common Use Cases

- Personal or team AI agent reachable from Telegram, Discord, or Slack
- Always-on assistant that keeps memory, skills, and session history on a volume you control
- Remote terminal and tool use without keeping a laptop online
- Experimentation with multiple inference providers (OpenRouter, OpenAI, Anthropic, xAI, and others)
- Privacy-focused alternative to hosted agent products — your keys and data stay in your Railway project

## Dependencies for Hermes Agent Hosting

- **Hermes Agent image:** `nousresearch/hermes-agent:latest` (always the default)
- **One persistent volume** mounted at `/data`
- **At least one messaging platform** (Telegram, Discord, or Slack tokens)
- **At least one inference provider** API key (OpenRouter recommended for a quick start)
- No external database required (Hermes manages its own state under `HERMES_HOME`)

Upstream: [Hermes Agent](https://github.com/NousResearch/hermes-agent) · [Docker Hub](https://hub.docker.com/r/nousresearch/hermes-agent) · [Nous Research](https://nousresearch.com)

### Implementation Details

| Item | Value |
|------|-------|
| Image | `nousresearch/hermes-agent:latest` |
| Default tag variable | `HERMES_IMAGE_VERSION=latest` |
| Config / state home | `/data/.hermes` (`HERMES_HOME`) |
| Default workspace | `/data/workspace` |
| Volume mount | `/data` |
| Start command | `hermes gateway` (via entrypoint) |
| Restart policy | `ON_FAILURE` |

The first boot creates the Hermes home directory layout and writes runtime environment into `/data/.hermes/.env`. Subsequent starts reuse existing state.

## Topology

| Service | Role | Volume | Public | Notes |
|---------|------|--------|--------|-------|
| **hermes** | Agent gateway + tools | `/data` | No (worker) | Messaging platforms connect outbound; use Railway SSH for CLI |

#### Volumes (drives) — what to mount

The volume is already defined in the template. It survives redeploys.

| Mount path | What is stored |
|------------|----------------|
| `/data` | Hermes home (`.hermes`), config, sessions, pairing, logs, workspace, lazy packages |

Do **not** remove or detach the `/data` volume — configuration, memory, and session history will be lost on the next deploy.

## Quick Start

1. Click the **Deploy on Railway** button above (or create a project from this repository / Dockerfile).
2. Sign in (or create a free Railway account) and deploy.
3. Confirm a volume is attached at `/data`.
4. Open the **hermes** service → **Variables** and set at least:
   - An inference key, e.g. `OPENROUTER_API_KEY`
   - A messaging platform, e.g. `TELEGRAM_BOT_TOKEN` and `TELEGRAM_ALLOWED_USERS`
5. Deploy / redeploy, wait for the service to become healthy.
6. Message your bot from an allowlisted account.

Default minimal example:

```env
OPENROUTER_API_KEY=""
TELEGRAM_BOT_TOKEN=""
TELEGRAM_ALLOWED_USERS=""
```

Allowlist values are comma-separated IDs (no brackets or quotes):

```env
TELEGRAM_ALLOWED_USERS=123456789,987654321
```

## Configuration

Most agent behavior is managed after startup (model selection, tools, pairing). Container-level variables that matter for Railway:

### Required / commonly used

| Variable | Default / Source | Notes |
|----------|------------------|-------|
| `OPENROUTER_API_KEY` | *(empty)* | Recommended quick-start inference provider |
| `TELEGRAM_BOT_TOKEN` | *(empty)* | Telegram bot token from BotFather |
| `TELEGRAM_ALLOWED_USERS` | *(empty)* | Comma-separated Telegram user IDs |
| `DISCORD_BOT_TOKEN` | *(empty)* | Alternative messaging platform |
| `DISCORD_ALLOWED_USERS` | *(empty)* | Comma-separated Discord user IDs |
| `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` | *(empty)* | Both required if using Slack |

You can use any inference provider and messaging platform supported by Hermes. See the [official Hermes repository](https://github.com/NousResearch/hermes-agent) for the full list; this template does not duplicate the upstream reference.

### Template-specific variables

| Variable | Default | Description |
|----------|---------|-------------|
| `HERMES_IMAGE_VERSION` | `latest` | Docker tag from `nousresearch/hermes-agent`. Always defaults to `latest`. Override with a release tag (e.g. `v2026.9.24`) only if you need a pinned version. |
| `HERMES_GIT_REF` | *(empty)* | Deprecated. Only used for legacy source builds. Leave empty to stay on the official image path. |
| `AGENT_CACHE_MEMORY_HIGH_MB` | *(unset)* | Optional positive integer mapped to `agent.agent_cache.memory_high_mb` before startup. |
| `HERMES_HOME` | `/data/.hermes` | Persistent Hermes state directory |
| `HERMES_WRITE_SAFE_ROOT` | `/data` | Safe write root for the agent |
| `HOME` | `/data` | Process home directory |

Because the default is `latest`, every redeploy (or Railway rebuild) pulls the newest published official image.

### Deprecated source-build compatibility

Existing deployments that still set `HERMES_GIT_REF` continue to build Hermes from that Git ref. To switch to the official image path:

1. Remove or empty `HERMES_GIT_REF`
2. Set `HERMES_IMAGE_VERSION=latest` (or omit it)
3. Redeploy

Legacy source builds fetch GitHub during uncached Railway builds and may fail with `429 Too Many Requests`. Prefer the official-image path.

### Custom Domain

Hermes is primarily a messaging worker, so a public HTTP domain is optional. If you expose an API server or other HTTP surface later:

1. Hermes service → **Settings** → **Networking** → **Custom Domain**.
2. Add your domain and follow Railway’s DNS instructions.
3. Railway provisions TLS automatically.

## Updating Hermes

Do **not** run `hermes update` inside the deployed container. Container filesystem changes do not survive a Railway redeploy and can leave persisted configuration ahead of the image version.

Instead:

1. Redeploy (with `HERMES_IMAGE_VERSION=latest` this automatically pulls the newest image).
2. If a release requires it, run `hermes config migrate` through Railway SSH.

To pin a specific version temporarily:

```env
HERMES_IMAGE_VERSION=v2026.9.24
```

Check the [Hermes Agent releases](https://github.com/NousResearch/hermes-agent/releases) and [Docker Hub tags](https://hub.docker.com/r/nousresearch/hermes-agent/tags) before upgrading.

## Railway SSH

Use Railway SSH to inspect Hermes or run commands manually:

```bash
hermes status
hermes config
hermes model
hermes pairing list
```

## Traps

**Ways this still fails or surprises people:**

- **Missing `/data` volume** — If the volume is detached or deleted, all state under `/data/.hermes` (config, sessions, pairing, memory) is lost on the next deploy.
- **No allowlist / deny-all** — Without `*_ALLOWED_USERS` (or an explicit allow-all flag), the gateway defaults to deny-all. Use DM pairing or set allowlists.
- **No inference provider** — The entrypoint refuses to start without a valid provider key (OpenRouter, OpenAI, Anthropic, etc.).
- **No messaging platform** — At least one of Telegram, Discord, or Slack must be configured or the container exits.
- **Running `hermes update` in the container** — Changes are ephemeral. Always upgrade by changing the image tag (or relying on `latest`) and redeploying.
- **Legacy `HERMES_GIT_REF` builds** — Source builds can hit GitHub rate limits (`429`). Remove `HERMES_GIT_REF` and use the official image path.
- **Private networking only for tools** — If Hermes needs to reach other Railway services (databases, download clients, etc.), use private domain references, not public URLs, when possible.

View live logs: service → **Deployments** → latest deployment → **View Logs**.

## Railway Infrastructure as Code

Railway configuration lives in `.railway/railway.ts`. After linking this repository to the intended Railway project and environment:

```bash
npm install
npm run railway:plan
npm run railway:apply
```

`railway:plan` is read-only. Review its output before running `railway:apply`.

## Local build

```bash
# Official image (always latest by default)
docker build -t hermes-railway-template .

# Pin a specific tag
docker build \
  --build-arg HERMES_IMAGE_VERSION=v2026.9.24 \
  -t hermes-railway-template .

# Deprecated source build (only if needed)
docker build \
  --build-arg HERMES_GIT_REF=v2026.9.24 \
  -t hermes-railway-template:legacy .
```

## Why Deploy Hermes Agent on Railway?

Railway is a singular platform to deploy your infrastructure stack. Railway will host your infrastructure so you don't have to deal with configuration, while allowing you to vertically and horizontally scale it.

By deploying Hermes Agent on Railway, you get an always-on autonomous agent with persistent memory and workspace, automatic container restarts, and simple secret/volume management — no servers to maintain.

---
