# Valor Digital — Paperclip Architecture

## Overview

Three-layer system for AI-agent company operations:

```
Paperclip (Company OS)     ← agents, heartbeats, budgets, governance
    ↕ hermes_local adapter
Hermes (Agent Runtime)     ← tools, skills, memory, MCP
    ↕ ValorBrain plugin  
ValorBrain (Institutional Memory) ← search, retrieve, write knowledge
    ↕ separate from
Multica (Dev Task Store)   ← issues, sprints, code review for dev projects
```

## What Each Layer Does

### Paperclip (`zeroinc.valor.digital`)
- **Role**: Company operating system — manages agents as employees
- **URL**: `https://zeroinc.valor.digital` (Cloudflare tunnel → `:3100`)
- **DB**: `paperclip` on PostgreSQL 5433
- **Auth**: email+senha (better-auth, `authenticated + public` mode)
- **Adapter**: `hermes-paperclip-adapter` (official NousResearch, MIT)
- **What it manages**:
  - Agent identity (name, role, title, reportsTo hierarchy)
  - Heartbeat scheduling (configurable per-agent cadence)
  - Budget tracking (monthly spend limits per agent)
  - Issue lifecycle (todo → checkout → in_progress → done)
  - Goal alignment (issues → goals → company mission)

### Hermes (Agent Runtime)
- **Role**: Execution engine — each agent IS a Hermes instance
- **Binary**: `/usr/local/bin/hermes` (symlink to venv)
- **Adapter**: `hermes chat -q` (single-query mode, session persistent)
- **Models**: GLM-5.2 (senior agents), GLM-5-turbo (junior agents), via ZAI
- **Tools**: terminal, file, web (configurable per agent)
- **Memory**: Persistent sessions via `--resume`
- **ValorBrain**: Built-in plugin (10 tools: 5 read + 5 write)

### ValorBrain (`valorbrain.valor.digital`)
- **Role**: Institutional memory — knowledge accessible to all agents
- **Port**: `:7438`
- **Access**: Via Hermes plugin (automatic when agent wakes)
- **Collections**: `org-catalog` (27 entries from knowledge-catalog)

### Multica (`agents.valor.digital`)
- **Role**: Dev project task management
- **Port**: `:8091`
- **Scope**: Development projects only (Climoo, ValorBrain, ValorCollab)
- **NOT used for**: Company operations, agent management, governance

## Agent Architecture

### 10 Agents (from knowledge-catalog personas)

| Agent | Model | Cadence | Role | Budget |
|-------|-------|---------|------|--------|
| Hermes | glm-5.2 | 30min | COO | R$500/mo |
| Val (Tech Lead) | glm-5.2 | 30min | Engineer | R$300/mo |
| Dan (Dev) | glm-5.2 | 30min | Engineer | R$200/mo |
| Iris (DevOps) | glm-5.2 | 60min | DevOps | R$150/mo |
| Rafa (Sales) | glm-5.2 | 60min | Sales | R$100/mo |
| Bi (SDR) | glm-5-turbo | 120min | Researcher | R$80/mo |
| Lena (CS Lead) | glm-5.2 | 60min | CS | R$100/mo |
| Theo (CS) | glm-5-turbo | 120min | CS | R$80/mo |
| Mari (Growth) | glm-5.2 | 60min | CMO | R$100/mo |
| Tav (Ops) | glm-5-turbo | 120min | CFO | R$80/mo |

### Hierarchy
```
Gus (CEO, board operator)
└── Hermes (COO)
    ├── Val (Tech Lead)
    │   ├── Dan (Dev)
    │   └── Iris (DevOps)
    ├── Rafa (Sales)
    │   └── Bi (SDR)
    ├── Lena (CS Lead)
    │   └── Theo (CS)
    ├── Mari (Growth)
    └── Tav (Ops)
```

## Heartbeat Flow

```
1. Paperclip scheduler ticks (every 30s)
   → Checks each agent's last_heartbeat_at vs intervalSec
   → If overdue, enqueues wakeup

2. Wakeup fires
   → Adapter calls: hermes chat -q "<prompt>" -m glm-5.2 --provider zai -t terminal,file,web
   → Hermes authenticates via JWT (PAPERCLIP_AGENT_JWT_SECRET)
   → Hermes checks inbox: GET /api/companies/:cid/issues?assigneeAgentId=X&status=todo
   → If issues found: checkout → work → complete
   → If no issues: reports idle status

3. Result captured
   → Adapter parses token usage, cost
   → Paperclip logs to heartbeat_runs + ndjson log file
   → Agent status updated
```

## Known Issues & Mitigations

1. **API Rate Limiting (HTTP 429)**
   - 10 agents waking simultaneously overwhelms ZAI API
   - **Mitigation**: Staggered heartbeat baselines (agents don't all fire at once)
   - **TODO**: Global concurrency limit in Paperclip

2. **Timeout for complex tasks**
   - GLM-5.2 code review tasks can take >5min
   - **Fix**: Senior agents use timeoutSec=600 (10min)

3. **Workspace path display bug (`/[]/`)**
   - Cosmetic only — agents can access filesystem normally
   - Root cause: `os.homedir()` stringification in log message

4. **Dashboard route**
   - UI calls `/api/companies/:id/dashboard`, server has `/api/dashboard/companies/:id/dashboard`
   - **Fixed**: Added proxy route in `companies.ts`

## Infrastructure

### Systemd: `paperclip.service`
```ini
Environment=HOME=/root
Environment=BETTER_AUTH_SECRET=<generated>
Environment=PATH=...:/root/.hermes/hermes-agent/venv/bin
ExecStart=tsx src/index.ts (from server/ directory)
```

### Cloudflare Tunnel
- Hostname: `zeroinc.valor.digital` → `http://127.0.0.1:3100`
- Config: `/etc/cloudflared/config.yml`

### Database
- DB: `paperclip` on PostgreSQL 5433 (user: `paperclip_app`)
- 57 tables, 38 migrations
- Backup: enabled (every 6h, 7-day retention)

## Integration Points

### Paperclip → Hermes
- Adapter: `hermes-paperclip-adapter` (npm package)
- Protocol: subprocess (`hermes chat -q`)
- Auth: JWT signed with `BETTER_AUTH_SECRET`

### Hermes → ValorBrain
- Plugin: `~/.hermes/plugins/valorbrain/`
- Tools: 10 (valorbrain_retrieve, valorbrain_get, valorbrain_write, etc.)
- Automatic: loaded on every Hermes startup

### Paperclip → Multica
- **No direct integration**
- Hermes can access Multica API via `terminal` tool (curl)
- Multica handles dev projects; Paperclip handles company operations

## Customizations (from upstream)

1. **Dashboard route proxy** — `companies.ts` line ~148
2. **Systemd service** — production deployment (tsx mode)
3. **Fork**: `github.com/agentxagi/paperclip`
