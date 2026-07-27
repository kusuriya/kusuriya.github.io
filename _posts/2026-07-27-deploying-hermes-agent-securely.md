---
title: "Deploying Hermes Agent securely: a practical hardening guide"
subtitle: "How to install, isolate, expose, and operate a tool-using agent without handing prompt injection your workstation"
description: "A threat-model-first guide to installing, isolating, exposing, and operating Hermes Agent without mistaking guardrails for security boundaries."
author: "Jason Barbier"
date: "2026-07-27"
updated: "2026-07-27"
tags:
  - hermes-agent
  - security
  - ai-agents
  - self-hosting
  - hardening
---

# Deploying Hermes Agent securely

Agents can be dangerous but with a bit of care they can be deployed safely and securely. Lets talk about how we could get Hermes to be deployed in a safe and sane fashion

<!--more-->

> **Reviewed against Hermes Agent v0.19.0 and the official documentation on July 27, 2026.** Hermes changes quickly. Before applying an option from this guide, compare it with the current [configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration), [security guide](https://hermes-agent.nousresearch.com/docs/user-guide/security), and the `--help` output from your installed version.

Hermes Agent can read files, run shell commands, operate browsers, call external services, remember information across sessions, accept messages from chat platforms, and schedule unattended work. That is why it is useful. It is also why deploying it like an ordinary chatbot is a mistake.

A chatbot receives text and returns text. An agent receives text and may turn it into operating-system activity. A malicious web page, email, attachment, MCP response, project instruction file, or chat message can become input to the model. If the model has a terminal and your normal user account, the distance between "untrusted text" and "my SSH keys" may be one tool call.

The central rule is blunt:

> **Treat the model as potentially adversarial and the operating system as the security boundary.**

That wording is not alarmism. It is the position in Hermes Agent's own [security policy](https://github.com/NousResearch/hermes-agent/security): approval prompts, output redaction, scanners, and tool allowlists are useful guardrails, but none of them contains an adversarial model. Real containment comes from an OS account, container, VM, sandbox, filesystem permissions, and network policy.

This guide turns that trust model into a deployment you can operate.

## The secure baseline in one page

If you only keep one section, keep this one.

1. **Run Hermes as a non-root, dedicated OS user or inside a whole-process container/VM.** Do not run a network-connected gateway under your everyday desktop account.
2. **Use whole-process isolation for untrusted input.** `terminal.backend: docker` confines terminal and file operations; it does not necessarily confine plugins, hooks, skills, MCP subprocesses, or every code path inside the main Python process. Running the entire Hermes process tree inside a container, VM, or OpenShell sandbox is the stronger posture.
3. **Expose nothing by default.** Keep the dashboard and API on loopback. Reach them through SSH or a private overlay network. If you bind to a non-loopback address, require authentication, TLS, and a firewall or VPN.
4. **Allowlist every caller.** Never set `GATEWAY_ALLOW_ALL_USERS=true` or a platform's `*_ALLOW_ALL_USERS=true` in production. Use explicit IDs or DM pairing, then revoke pairing you no longer need.
5. **Do not mistake gateway admin tiers for tool isolation.** Current admin/user tiers primarily gate slash commands. Plain chat still reaches the agent. Separate trust classes with separate Hermes profiles or, for stronger isolation, separate OS users or containers.
6. **Enable only the tools required by that profile.** A research bot usually does not need terminal, computer control, cron management, or arbitrary file writes. A coding bot probably does not need access to personal mail.
7. **Keep secrets out of prompts, repositories, skills, and `config.yaml`.** Use `~/.hermes/.env`, Hermes auth storage, or an external secret provider. Lock state files to the service account. Forward only the credentials a sandboxed task actually needs.
8. **Review executable extensions.** Skills, plugins, hooks, and local MCP servers are code, not prompt snippets. Read their scripts and dependency manifests before installation. Pin versions where possible.
9. **Keep approvals and redaction enabled.** Use `approvals.mode: smart` or `manual`, keep `approvals.cron_mode: deny`, and leave `security.redact_secrets: true`. These controls reduce mistakes even though they are not containment.
10. **Patch, audit, back up, and test recovery.** Run `hermes doctor`, `hermes security audit`, review enabled tools and MCP servers, create encrypted backups, and verify that you can restore without accidentally publishing the backup's credentials.

## Start with a threat model, not a YAML file

Before choosing settings, write down what can enter the agent and what the agent can reach.

### Assets worth protecting

A typical Hermes home contains more than configuration:

- provider API keys and OAuth refresh tokens;
- Telegram, Discord, Slack, email, or other gateway credentials;
- MCP OAuth tokens and service credentials;
- session transcripts, tool results, memories, and user profile data;
- installed skills, plugins, hooks, and scheduled jobs;
- project source, cloud credentials, SSH keys, browser sessions, and CLI logins reachable through the host account;
- the ability to send messages or perform actions as the operator.

The `~/.hermes/` directory is therefore a credential store, a conversation archive, an extension directory, and a persistence mechanism. Back it up like sensitive application state, not like a dotfile theme.

### Input surfaces

Assume all of these can contain hostile instructions:

- direct chat messages, even from an authorized user whose account was compromised;
- group-channel messages and quoted/replied content;
- email bodies and attachments;
- web search results and pages opened by browser automation;
- repository files such as `.hermes.md`, `AGENTS.md`, `CLAUDE.md`, or `.cursorrules`;
- documents, PDFs, images, and OCR output;
- MCP tool descriptions and responses;
- terminal output, logs, issue comments, CI output, and pasted stack traces;
- persistent memory, skill content, and cron prompts.

Authorization answers *who may send input*. It does not make the input safe. An allowlisted engineer can paste a poisoned issue description. A trusted mailbox can receive an attacker-controlled email.

### Trust boundaries

Use this hierarchy:

| Layer | What it does | Is it a security boundary? |
|---|---|---:|
| Prompt instructions and `SOUL.md` | Tell the model how to behave | No |
| Prompt-injection scanners and Skills Guard | Detect known suspicious patterns | No |
| Tool allowlists and approval rules | Reduce reachable actions and catch common mistakes | No, but valuable least privilege |
| Secret and PII redaction | Reduce accidental disclosure in context and logs | No |
| Separate Hermes profile | Separates Hermes state such as config, sessions, memory, and skills | Not an OS sandbox |
| Dedicated Unix account | Limits access to that account's files and privileges | Yes |
| Terminal backend container/remote host | Confines terminal and file-tool execution to exposed resources | Yes, for the paths routed through it |
| Whole-process container, VM, or OpenShell sandbox | Constrains the agent process tree and its extensions | Yes, subject to mounts, privileges, and network policy |

The distinction matters. If you configure `terminal.backend: docker` but install a malicious plugin into the host-side Hermes process, the terminal sandbox is solving the wrong problem. The plugin runs with the privileges of the agent process.

## Choose a deployment posture deliberately

There is no single secure configuration because "secure" depends on the inputs and assets.

### Posture A: local interactive assistant

Use this when one trusted operator launches Hermes manually, reviews approvals, and avoids untrusted inputs.

A reasonable baseline is a non-root account, loopback-only network surfaces, `approvals.mode: manual` or `smart`, narrow tools, and no unattended gateway. The default local terminal backend still gives the agent the same filesystem access as the account running it.

This posture is convenient, not isolated. Do not browse arbitrary pages, process unknown attachments, or read hostile repositories while authenticated to high-value cloud and developer accounts.

### Posture B: personal messaging gateway

A Telegram or Discord bot connected to a single operator is still remotely reachable and usually unattended. Put it under a dedicated OS account or whole-process container, configure an explicit caller allowlist, keep the dashboard/API private, and enable hard stops for repeated tool loops.

The gateway's authorization controls are necessary. They do not protect against a compromised operator account or poisoned content forwarded by the operator. The agent still needs containment.

### Posture C: shared or production deployment

If Hermes accepts inbound email, participates in multi-user rooms, reads open-web content, connects to untrusted MCP servers, or serves an organization, use whole-process isolation. Prefer a VM, a carefully hardened container, or [NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell) when you need declarative filesystem, process, egress, and inference policies.

Hermes describes itself as a **single-tenant personal agent**. If several people need different capabilities, do not bolt role-based access control onto one powerful instance and call it multi-tenant. Run separate profiles at minimum. For meaningful isolation, give each trust class its own OS account, container, state directory, credentials, toolset, and network policy.

## Install from a source you can identify

The official installer is convenient:

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

For a security-sensitive host, do not pipe network content directly into a shell without looking at it. Download, inspect, then run it:

```bash
install -d -m 0700 "$HOME/tmp/hermes-install"
curl -fsSLo "$HOME/tmp/hermes-install/install.sh" \
  https://hermes-agent.nousresearch.com/install.sh
less "$HOME/tmp/hermes-install/install.sh"
bash "$HOME/tmp/hermes-install/install.sh"
```

This does not cryptographically prove the script is safe; it prevents blind execution and gives you a reviewable artifact. For reproducible production deployments, build from a reviewed commit or use the official container image pinned to an immutable digest after testing it in staging. Avoid floating `latest` tags in unattended production.

After installation:

```bash
hermes --version
hermes doctor
hermes security audit
hermes config path
hermes config env-path
hermes tools list
hermes mcp list
hermes plugins list
```

`hermes security audit` scans the active Hermes Python environment, Python dependencies declared by plugins, and pinned `npx`/`uvx` MCP servers against OSV.dev. It does not scan every globally installed package or browser/editor extension, so keep your host's normal vulnerability management in place.

Update on a defined cadence, not whenever an agent decides to update itself:

```bash
hermes backup --quick -o "$HOME/hermes-pre-update.zip"
hermes update
hermes doctor
hermes security audit
```

Hermes can create pre-update snapshots itself, but an external backup and tested rollback are still useful. Review release notes and security advisories before broad rollout.

## Isolate the operating-system account

On a host install, create a dedicated unprivileged account for a long-running gateway. The exact service-management commands vary by distribution, but the properties should not:

- no root login;
- no passwordless `sudo`;
- no membership in `docker`, `libvirt`, or other root-equivalent groups unless the deployment explicitly requires it;
- no access to your personal home, SSH agent, browser profile, cloud CLI directories, or desktop session;
- a private `HERMES_HOME` owned by that account;
- a narrow working directory containing only the projects or data the agent needs.

The Docker group is effectively root on most Linux systems. Putting a "sandboxed" agent in that group so it can launch arbitrary containers can collapse the boundary you were trying to build. Prefer a pre-created execution environment, rootless container runtime, remote worker, or whole-process container managed outside the agent.

For an existing single-user installation, verify state permissions without printing secrets:

```bash
stat -c '%a %U:%G %n' \
  "$HOME/.hermes" \
  "$HOME/.hermes/.env" \
  "$HOME/.hermes/auth.json" 2>/dev/null
```

A sensible target is `0700` for private state directories and `0600` for secret-bearing files. Apply permissions to paths that exist:

```bash
chmod 0700 "$HOME/.hermes"
chmod 0600 "$HOME/.hermes/.env" "$HOME/.hermes/auth.json" 2>/dev/null || true
```

Do not make `/etc/hermes/.env` a high-value secret store merely because managed scope exists. The documented managed-scope v1 example uses root-owned but world-readable `0644` files and explicitly warns that its managed `.env` is unsuitable for high-sensitivity secrets. Managed scope pins operator policy; it is not an agent sandbox or secret vault.

### Run the gateway under a service manager

Hermes can install either a per-user or system gateway service:

```bash
# Run as the dedicated Hermes account.
hermes gateway install
hermes gateway status

# Alternative for centrally managed hosts; inspect the generated unit.
sudo hermes gateway install --system
hermes gateway status --system
```

A per-user systemd service avoids a root-owned agent process. On a server, enable lingering for the dedicated account so its user manager starts at boot and survives logout:

```bash
sudo loginctl enable-linger hermes
```

If you use the system service, verify that its unit names an unprivileged `User=` and `Group=`, loads a protected environment file, and starts in a controlled working directory. A system unit should not imply that Hermes itself runs as root.

For systemd-managed gateways, an optional event-loop watchdog can restart a process whose asyncio loop stops making progress:

```yaml
gateway:
  systemd_watchdog_seconds: 120
```

Regenerate the user unit after changing it with `hermes gateway install --force`. The watchdog improves availability; it does not detect a compromised but responsive agent.

## Pick the right isolation layer

### The local backend has no isolation

With the default configuration:

```yaml
terminal:
  backend: local
```

commands run as the Hermes OS user. On host installs, `terminal.home_mode: auto` preserves the account's real `HOME`, which means tools such as `ssh`, `gh`, `az`, `npm`, and `git` can see that user's existing configuration and credentials. This is convenient for a personal coding assistant and dangerous for a remote gateway under your primary account.

If you use host-side profiles and need separate CLI identities, set:

```yaml
terminal:
  home_mode: profile
```

That changes subprocess `HOME` to a profile-specific directory. It separates normal CLI configuration between profiles, but it is still not a filesystem sandbox. Hermes exposes `HERMES_REAL_HOME` to subprocesses, and the OS user may still be able to reach the real home by absolute path.

### A Docker terminal backend is partial isolation

Hermes can run terminal and file operations in a persistent execution container:

```yaml
terminal:
  backend: docker
  docker_network: false
  docker_persist_across_processes: false
  docker_volumes:
    - "/srv/hermes-work:/workspace"
    - "/srv/reference-data:/data:ro"
```

`docker_network: false` cuts network access from the execution container. Mount only the paths required for the task, use read-only mounts where possible, and do not forward credentials by default. If a task needs one token, forward that one token rather than the host environment.

The persistent backend is useful but easy to misunderstand:

- the default Docker execution container may survive sessions and process restarts;
- sessions and subagents can share it, so installed packages, files, and background processes can persist;
- a writable host mount lets container commands alter the mounted host data;
- `--network=host`, broad bind mounts, `--privileged`, extra capabilities, and a mounted Docker socket weaken or destroy isolation;
- according to the Hermes security policy, terminal-backend isolation is narrower than whole-process isolation and must not be assumed to contain in-process plugins, hooks, skill loading, or MCP code.

Use it to constrain tool execution from an otherwise trusted agent process. Do not advertise it as complete protection against hostile inputs.

### Whole-process containers are the production baseline

Running Hermes itself in a container puts the agent process, tools, extension code, and subprocesses inside the same OS boundary. Hermes publishes an official image and a Compose file. The image runs services as a non-root `hermes` user and keeps mutable state under `/opt/data`.

A minimal setup flow from the [Docker guide](https://hermes-agent.nousresearch.com/docs/user-guide/docker) is:

```bash
mkdir -p "$HOME/.hermes"
chmod 0700 "$HOME/.hermes"

docker run -it --rm \
  -v "$HOME/.hermes:/opt/data" \
  nousresearch/hermes-agent setup
```

Then run the gateway without publishing ports you do not need:

```bash
docker run -d \
  --name hermes \
  --restart unless-stopped \
  -v "$HOME/.hermes:/opt/data" \
  nousresearch/hermes-agent gateway run
```

The official quick-start examples are starting points, not a complete production policy. Harden the runtime for your environment:

- pin the image by digest;
- keep the runtime non-root;
- never mount `/var/run/docker.sock`;
- avoid `--privileged`, host PID namespace, unnecessary devices, and unnecessary capabilities;
- avoid host networking unless a specific integration requires it;
- publish only required ports and bind them to loopback or a private interface;
- mount only `/opt/data` and explicit work directories;
- keep sensitive reference data read-only;
- set CPU, memory, PID, and storage limits;
- restrict outbound network access to model providers and services the profile actually uses;
- store backups outside the live writable volume.

A whole-process container still has full access to whatever you mount and whatever the network permits. `/opt/data` contains credentials and executable extensions, so compromise of the agent can alter its own future configuration or persistence unless you add further controls. A VM or OpenShell policy can provide a stronger boundary when you need strict egress control or separation from the container host.

## Apply least privilege to tools

Hermes exposes toolsets through `hermes tools`. Tool changes take effect on a new session, so reset or restart after changing them.

```bash
hermes tools list
hermes tools disable terminal
hermes tools disable computer_use
hermes tools disable cronjob
hermes tools disable browser
```

The exact least-privilege set depends on the role:

| Profile | Usually needed | Usually avoid |
|---|---|---|
| Read-only research | web search/extract, perhaps file read in a sandbox | terminal, computer control, cron, messaging writes |
| Repository assistant | file and terminal inside a project sandbox | personal email, home automation, broad browser sessions |
| Messaging concierge | messaging and calendar APIs with narrow scopes | shell access to the host, arbitrary MCP servers |
| Monitoring bot | a fixed script or read-only API, local delivery | full agent loop when a deterministic watchdog suffices |
| Administrative agent | narrowly scoped admin tools in a dedicated environment | sharing the same instance with ordinary users |

Tool filtering reduces attack surface and prompt size. It also creates a simpler review question: "Why can this profile do that?" If there is no current answer, disable the tool.

Use separate profiles for different duties:

```bash
hermes profile create research
hermes profile create coding
hermes profile create gateway
```

Profiles separate config, credentials, memory, sessions, skills, cron jobs, and state databases. They do **not** create OS isolation. On a local backend, all profiles still run as the same Unix user unless you place them in separate accounts or containers.

## Keep secrets scoped and short-lived

Hermes stores ordinary API keys and bot tokens in `~/.hermes/.env`; OAuth and credential pools may live in `auth.json`; MCP OAuth tokens live under `~/.hermes/mcp-tokens/` with restrictive file modes. Treat the entire Hermes home as sensitive.

Rules that hold up in practice:

1. Never put live credentials in `SOUL.md`, project instruction files, skill documentation, cron prompts, or chat messages.
2. Never commit `.env`, `auth.json`, MCP tokens, session databases, or profile exports.
3. Use separate service credentials for Hermes instead of reusing human administrator tokens.
4. Scope tokens at the provider: repository-specific GitHub access, read-only database users, limited cloud roles, narrow OAuth scopes.
5. Prefer short-lived credentials and rotate bot/API tokens after an incident.
6. Do not forward the whole host environment into containers or MCP servers.
7. Keep secret redaction enabled:

```bash
hermes config set security.redact_secrets true
```

Secret redaction applies when tool output enters conversation context and logs. It is defense in depth, not a guarantee. A determined component can encode or fragment a secret. The setting is loaded early; restart Hermes after changing it.

For gateway privacy, consider:

```bash
hermes config set privacy.redact_pii true
```

This hashes user IDs and strips phone numbers from model context. It does not erase the original data from the messaging platform or retroactively sanitize old sessions.

If your installation supports external secret providers, `hermes secrets` can keep selected credentials out of a flat `.env`. That improves storage and rotation but does not make a credential safe after you deliberately inject it into an agent or subprocess that can use it.

## Secure the messaging gateway

The gateway connects remote users to an agent with tools. Its allowlist is an authentication boundary.

### Deny by default

Hermes documents deny-by-default behavior for unpaired and non-allowlisted users. Configure explicit IDs for every platform you enable:

```dotenv
TELEGRAM_ALLOWED_USERS=123456789
DISCORD_ALLOWED_USERS=123456789012345678
SLACK_ALLOWED_USERS=U0123456789
EMAIL_ALLOWED_USERS=trusted@example.com
```

Do not use these production shortcuts:

```dotenv
GATEWAY_ALLOW_ALL_USERS=true
TELEGRAM_ALLOW_ALL_USERS=true
DISCORD_ALLOW_ALL_USERS=true
SLACK_ALLOW_ALL_USERS=true
```

For DM pairing:

```bash
hermes pairing list
hermes pairing approve telegram XKGH5N7P
hermes pairing revoke telegram 123456789
```

Pairing codes expire and are rate-limited, but pairing is still account authorization. Approve through a separate trusted channel when identity matters.

### Restrict groups, rooms, and mentions

A group allowlist can mean either "these senders may use the bot" or "any member of this approved room may use the bot," depending on the platform and option. The latter inherits the room's entire membership and moderation model. Prefer sender IDs when practical, restrict allowed channels/rooms, and require mentions so ordinary ambient conversation does not become agent input.

For Discord, for example, use explicit users and channels, require mentions, and leave arbitrary attachment acceptance disabled unless you have an isolated document-processing workflow. Similar controls exist for Telegram, Matrix, Mattermost, and other adapters in the [messaging documentation](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/).

### Understand the admin/user split

Hermes supports per-scope admin lists and `user_allowed_commands`. This is useful for preventing ordinary users from invoking sensitive slash commands. The current documentation is explicit that this split gates slash commands; it does not yet provide per-user tool capability isolation for plain conversation.

If Alice should be allowed to research but Bob can administer infrastructure, do not rely on one shared agent and two slash-command tiers. Give them separate instances with different tools and credentials.

### Approvals over chat

Gateway users can approve dangerous commands remotely. An attacker who controls an authorized chat account may therefore control both the request and the approval channel. Keep high-risk profiles single-user, protect the messaging account with strong authentication, and use `approvals.cron_mode: deny` for unattended work.

## Keep the dashboard and API private

### Dashboard

The dashboard can edit configuration and API keys, inspect sessions, manage skills and MCP servers, and open a live Hermes TUI. It is an administrative control plane.

By default it binds to `127.0.0.1:9119`:

```bash
hermes dashboard --host 127.0.0.1 --port 9119 --no-open
```

For remote access, the simplest strong pattern is an SSH tunnel:

```bash
ssh -N -L 9119:127.0.0.1:9119 hermes-host
```

Then browse to `http://127.0.0.1:9119` locally. A private overlay network such as Tailscale is another option, provided the service binds only to the intended interface and host firewall rules are narrow.

Current Hermes releases fail closed when a non-loopback dashboard has no authentication provider. If you deliberately expose it, configure username/password for a trusted private network or OIDC/OAuth for a broader deployment, add TLS at a reverse proxy, and verify that the auth gate is active. The official dashboard guide documents:

```dotenv
HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=<strong unique password>
HERMES_DASHBOARD_BASIC_AUTH_SECRET=<stable random signing secret>
```

Generate the signing secret outside the agent process:

```bash
openssl rand -base64 32
```

Do not treat basic authentication over plain HTTP as safe on an untrusted network. Use HTTPS and a VPN or firewall. Do not expose the dashboard directly to the public internet if a private access path will do.

### OpenAI-compatible API server

The API server can submit work to an agent with its full enabled toolset. Enable it only when required:

```dotenv
API_SERVER_ENABLED=true
API_SERVER_HOST=127.0.0.1
API_SERVER_KEY=<long random bearer token>
```

The bearer key is required even on loopback. If a reverse proxy or remote frontend needs access, bind carefully, require the key, use TLS, restrict source networks, and keep CORS disabled unless a browser genuinely needs it. If browser access is necessary, list exact origins:

```dotenv
API_SERVER_CORS_ORIGINS=https://ui.example.internal
```

Never use `*` for production CORS merely to make a frontend work. Server-to-server clients do not need browser CORS.

Treat the public `/health` endpoint as liveness only. Use the authenticated detailed health endpoint for readiness and avoid publishing extra administrative endpoints through a proxy unless required.

### Webhooks

Every public webhook needs transport authentication: HMAC, provider signature validation, a strong route secret, replay controls where supported, and rate limiting at the edge. Keep the listener private behind a reverse proxy when possible. Telegram webhook mode requires `TELEGRAM_WEBHOOK_SECRET`; Twilio SMS should validate Twilio signatures and must not use `SMS_INSECURE_NO_SIGNATURE=true` outside local development. The general webhook adapter uses `WEBHOOK_SECRET` as its HMAC secret.

## Treat prompt injection as an expected input condition

You cannot solve prompt injection by adding "ignore malicious instructions" to `SOUL.md`. You can make successful injection less consequential.

Use four layers:

1. **Minimize authority.** Disable tools and credentials that are not needed.
2. **Constrain execution.** Put file, process, and network access behind OS policy.
3. **Separate trust domains.** Do not mix open-web research, personal email, production administration, and private memory in one profile.
4. **Require human confirmation for consequential actions.** Keep approvals on, and design workflows that stage rather than send, draft rather than publish, and propose rather than delete.

Hermes scans project context files and scheduled prompts for suspicious patterns. Keep those scanners enabled. For a security-sensitive service, also consider failing closed if the Tirith command scanner itself errors:

```yaml
security:
  tirith_enabled: true
  tirith_fail_open: false
```

This turns scanner failure into a blocked command rather than silent continuation. It does not turn Tirith into containment; assume an attacker can bypass pattern matching through encoding, indirection, images, tool output, or a novel phrasing.

Persistent memory and agent-written skills deserve their own approval gate because they can influence future sessions. Current Hermes releases can stage these writes for review:

```yaml
memory:
  write_approval: true

skills:
  write_approval: true
```

Review staged changes with `/memory pending` and `/skills pending`, including the full skill diff before approval. For high-sensitivity or disposable workloads, disabling persistent memory may be simpler:

```bash
hermes config set memory.memory_enabled false
```

If an agent behaves unexpectedly, restart it with `hermes --safe-mode`. Safe mode disables user configuration, injected rules and memory, plugins, shell hooks, and MCP servers for diagnosis. `--ignore-rules` is narrower: it skips project rules, `SOUL.md`, memory, and preloaded skills. Neither mode replaces OS isolation.

A useful design rule is **read untrusted, write trusted, never both in the same unconstrained turn**. For example, a research profile may read the open web and write only to a staging directory. A separate publishing workflow reviews and promotes the result. Do not let the same open-web agent hold production deployment credentials.

## Audit skills, plugins, hooks, and MCP servers

Hermes is extensible because it can load and execute third-party components. That makes extension review part of deployment security.

### Skills

A skill may include Python, shell scripts, templates, binaries, or instructions that cause later tool calls. Read the entire skill directory, not just `SKILL.md`. Check:

- install and update source;
- scripts and imported modules;
- required environment variables;
- network destinations;
- file paths it reads or writes;
- commands it asks the agent to run;
- whether it declares or forwards credentials;
- whether it installs dependencies at runtime.

Skills Guard is a review aid, not a trust seal.

### Plugins

Plugins load into the Hermes process with the agent's privileges. They can register tools, hooks, routes, background services, and configuration. A compromised plugin is equivalent to compromised application code. Pin a reviewed revision, inspect updates before applying them, and disable anything unused:

```bash
hermes plugins list
hermes plugins disable <name>
```

### Hooks

Hooks are shell code triggered by events. Do not use `--accept-hooks`, `HERMES_ACCEPT_HOOKS=1`, or broad automatic hook approval on an environment that processes untrusted repositories. Review the hook body and the event that triggers it. Headless convenience should not silently become persistence.

### MCP servers

A local stdio MCP server is a subprocess. A remote MCP server is a network trust relationship and can return attacker-controlled content. Catalog review helps, but installation may still clone repositories and run package-manager bootstrap commands.

Before installing an MCP server:

```bash
hermes mcp catalog
hermes mcp test <server>
hermes mcp configure <server>
```

Inspect its manifest or command, pin package versions instead of using floating `@latest`, scope its credentials, filter out mutating tools you do not need, and disable MCP sampling for untrusted servers unless you intend to let them trigger model calls:

```yaml
mcp_servers:
  untrusted_server:
    url: "https://mcp.example.com/mcp"
    sampling:
      enabled: false
    tools:
      include:
        - search
        - read_item
```

Do not place bearer tokens directly in a shareable YAML file. Use environment substitution or OAuth storage, and protect client certificate keys with restrictive permissions.

## Make unattended automation boring

Cron turns an agent action into persistence. A bad interactive decision runs once; a bad scheduled job runs until noticed.

Keep the safe default:

```yaml
approvals:
  cron_mode: deny
```

Cron jobs cannot recursively create more cron jobs, and Hermes scans job prompts at creation and update. Still review the job list, attached skills, scripts, model/provider, work directory, output destination, and execution history:

```bash
hermes cron list
hermes cron status
hermes cron runs --limit 20
```

Prefer deterministic `no-agent` scripts for threshold checks and watchdogs. They use no LLM and can stay silent unless a fixed condition is met. If a job does need reasoning:

- pin the model and provider;
- give it the minimum toolsets through a dedicated profile;
- use an explicit absolute `workdir`;
- avoid injecting broad project instruction files unintentionally;
- deliver only to intended recipients;
- keep credentials read-only and narrowly scoped;
- put side effects behind idempotency checks;
- cap time, cost, and retries.

Enable hard stops for unattended tool loops:

```yaml
tool_loop_guardrails:
  hard_stop_enabled: true
  hard_stop_after:
    exact_failure: 5
    idempotent_no_progress: 5
```

These are availability and cost controls, not isolation, but unattended agents need them.

## A conservative baseline configuration

This is a pattern, not a drop-in universal file. Keep secrets in `.env`, adapt toolsets per profile, and verify keys against your installed version.

```yaml
terminal:
  # For a host-side agent, use docker/ssh/modal/daytona/singularity
  # when shell and file operations need a separate boundary.
  backend: docker
  docker_network: false
  docker_persist_across_processes: false
  docker_volumes:
    - "/srv/hermes-work:/workspace"
    - "/srv/hermes-reference:/reference:ro"
  # Forward no secrets unless a task explicitly needs one.
  docker_forward_env: []

approvals:
  mode: smart
  timeout: 300
  cron_mode: deny
  mcp_reload_confirm: true
  destructive_slash_confirm: true
  deny:
    - "git push --force*"
    - "*curl*|*sh*"

security:
  redact_secrets: true
  tirith_enabled: true
  tirith_fail_open: false
  allow_private_urls: false
  allow_lazy_installs: false

privacy:
  redact_pii: true

memory:
  write_approval: true

skills:
  write_approval: true

tool_loop_guardrails:
  hard_stop_enabled: true
  hard_stop_after:
    exact_failure: 5
    idempotent_no_progress: 5

gateway:
  delivery_ledger: true

session_reset:
  mode: idle
  idle_minutes: 240
```

Why these choices:

- networkless execution is safer for local transformations and can be relaxed per task when network access is truly required;
- non-persistent execution reduces cross-session residue, though mounted data still persists;
- approvals catch cooperative mistakes and cron fails closed;
- Tirith failure blocks commands instead of failing open;
- memory and skill writes are staged for human review;
- redaction stays on;
- private/internal URL access stays off unless the deployment needs it, reducing SSRF reach;
- lazy dependency installation is disabled for controlled production images;
- tool-loop hard stops bound unattended failure;
- idle session reset reduces stale conversational context, though it does not delete stored history.

If you run Hermes entirely inside a constrained whole-process container, you may choose a local terminal backend *inside that container*. The outer container or VM is then the boundary. Keep the in-process guardrails anyway.

## Logging, backups, and incident response

### Logs

Hermes logs help reconstruct tool calls, gateway activity, errors, and cron execution. They can also contain sensitive paths, message content, and partial tool output. Restrict log access, rotate logs, and do not upload debug bundles publicly without reviewing them.

Useful checks:

```bash
hermes status --all
hermes logs gateway --since 1h
hermes logs errors --since 24h
hermes doctor
hermes security audit
```

Secret redaction is enabled by default, and debug sharing applies forced redaction before upload. Redaction still cannot guarantee that every sensitive value is detected.

### Backups

A full backup can contain `.env`, auth state, sessions, memories, skills, plugins, and cron jobs. Encrypt it at rest, restrict access, and keep an offline or immutable copy. Test restore on a separate account or disposable machine.

```bash
hermes backup -o "$HOME/hermes-backup.zip"
hermes backup --quick -o "$HOME/hermes-critical-state.zip"
```

Do not place those ZIP files in a public sync folder or repository. If you restore after a compromise, inspect extensions and cron jobs before starting the gateway, and rotate credentials rather than assuming the backup is clean.

### Incident response checklist

If you suspect prompt injection led to unauthorized activity:

1. Stop the gateway and API/dashboard exposure.
2. Preserve logs, session state, cron execution history, and container metadata before cleanup.
3. Revoke bot tokens, API keys, OAuth grants, MCP tokens, and any credentials mounted or forwarded to the agent.
4. Review `cron`, plugins, skills, hooks, MCP configuration, shell startup files, SSH authorized keys, and user services for persistence.
5. Rebuild the runtime from a known-good image or commit instead of cleaning an unknown process in place.
6. Restore only reviewed state. Treat memories and instruction files as potentially poisoned.
7. Re-enable one surface at a time behind explicit allowlists and monitoring.
8. Report a boundary-crossing vulnerability privately through [GitHub Security Advisories](https://github.com/NousResearch/hermes-agent/security/advisories/new) or `security@nousresearch.com`.

A successful prompt injection is not automatically a software vulnerability. A bypass of a claimed OS boundary, unauthorized external access, or credential leakage contrary to the documented trust model may be.

## Verification checklist before going live

Run this review for every profile, not just once per machine.

### Identity and isolation

- [ ] Hermes runs as non-root.
- [ ] The OS account cannot read personal or unrelated service data.
- [ ] Untrusted input is handled inside whole-process isolation.
- [ ] Containers have no Docker socket, privileged mode, broad host mounts, or unnecessary host networking.
- [ ] Egress is restricted to required destinations.

### Access

- [ ] Every messaging adapter has an explicit allowlist or approved pairing.
- [ ] All `*_ALLOW_ALL_USERS` and `GATEWAY_ALLOW_ALL_USERS` settings are absent or false.
- [ ] Group/room and mention rules are explicit.
- [ ] Different trust classes use separate instances.
- [ ] Dashboard and API bind to loopback or a private interface.
- [ ] Remote HTTP access has authentication, TLS, and firewall/VPN controls.
- [ ] API keys are long, unique, and rotated on a schedule.
- [ ] CORS uses exact origins or remains disabled.

### Capability

- [ ] `hermes tools list` contains only required toolsets.
- [ ] Terminal, computer control, messaging writes, cron, and admin tools have a documented reason to exist.
- [ ] `approvals.mode` is `smart` or `manual`.
- [ ] `approvals.cron_mode` is `deny`.
- [ ] `security.redact_secrets` is true.
- [ ] `security.tirith_fail_open` is false on security-sensitive services.
- [ ] Memory and skill writes require review, or persistence is disabled.
- [ ] Unattended profiles have tool-loop hard stops.

### Supply chain and persistence

- [ ] Hermes version/image/commit is pinned and recorded.
- [ ] `hermes doctor` and `hermes security audit` pass or findings are accepted explicitly.
- [ ] Skills, plugins, hooks, and MCP servers were reviewed beyond their descriptions.
- [ ] Package and MCP versions are pinned where possible.
- [ ] Runtime lazy installs are disabled in controlled production images.
- [ ] Cron jobs, attached skills, scripts, and delivery targets were reviewed.

### Data and recovery

- [ ] Hermes state and secrets have restrictive ownership and modes.
- [ ] Backups are encrypted, access-controlled, and tested.
- [ ] Logs are protected and retained long enough for investigation.
- [ ] Credential rotation and clean rebuild procedures are documented.
- [ ] The team can stop all gateway/API traffic quickly.

## Final perspective

The dangerous way to deploy an agent is to start with features: connect every tool, mount the home directory, add chat platforms, turn off approvals when they become annoying, and expose the dashboard so it is easy to reach.

The secure way starts with consequences. Decide what the agent may read, write, execute, contact, remember, and persist. Put those decisions into OS boundaries and credentials before giving the model tools. Add only the capabilities required for one role. Treat remote input and retrieved content as hostile. Keep human approval as a brake, not as your containment strategy.

Hermes can be deployed responsibly, but only if you preserve the distinction its own security policy makes: **guardrails help a cooperative model avoid mistakes; isolation limits what happens when cooperation fails.**

## Sources and further reading

1. [Hermes Agent security policy](https://github.com/NousResearch/hermes-agent/security)
2. [Hermes Agent security guide](https://hermes-agent.nousresearch.com/docs/user-guide/security)
3. [Configuration reference](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
4. [Docker deployment guide](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
5. [Messaging and gateway documentation](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/)
6. [Web dashboard documentation](https://hermes-agent.nousresearch.com/docs/user-guide/features/web-dashboard)
7. [OpenAI-compatible API server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server)
8. [MCP integration and trust model](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp)
9. [Profiles and their isolation limits](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
10. [Cron and unattended automation](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
11. [Environment variable reference](https://hermes-agent.nousresearch.com/docs/reference/environment-variables)
12. [Built-in tools reference](https://hermes-agent.nousresearch.com/docs/reference/tools-reference)
13. [Managed scope](https://hermes-agent.nousresearch.com/docs/user-guide/managed-scope)
14. [NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell)
