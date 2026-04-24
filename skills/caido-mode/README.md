# Caido Mode

Full SDK CLI for [Caido](https://caido.io) built on the official [`@caido/sdk-client`](https://github.com/caido/sdk-js) package. Search HTTP history, edit/replay requests with preserved auth, manage scopes/filters/environments, create findings, export curl commands, control intercept, and fuzz — all from the terminal.

## Why?

Cookies and auth tokens are huge. Instead of copy-pasting 2KB of session cookies into every test request, you:

1. Find an organic request in Caido's history that already has valid auth
2. Use `edit` to change just the path/method/body while keeping all auth intact
3. Send it — full response comes back, request shows up in Caido

## What's Covered

| Category | Commands |
|----------|----------|
| **HTTP History** | `search`, `recent`, `get`, `get-response`, `export-curl` |
| **Edit & Replay** | `edit`, `replay`, `send-raw` |
| **Sessions** | `create-session`, `rename-session`, `replay-sessions`, `delete-sessions` |
| **Collections** | `replay-collections`, `create-collection`, `rename-collection`, `delete-collection` |
| **Fuzzing** | `create-automate-session`, `fuzz` |
| **Scopes** | `scopes`, `create-scope`, `update-scope`, `delete-scope` |
| **Filter Presets** | `filters`, `create-filter`, `update-filter`, `delete-filter` |
| **Environments** | `envs`, `create-env`, `select-env`, `env-set`, `delete-env` |
| **Findings** | `findings`, `get-finding`, `create-finding`, `update-finding` |
| **Tasks** | `tasks`, `cancel-task` |
| **Projects** | `projects`, `select-project` |
| **Hosted Files** | `hosted-files`, `delete-hosted-file` |
| **Intercept** | `intercept-status`, `intercept-enable`, `intercept-disable` |
| **Info** | `viewer`, `plugins`, `health` |
| **Auth** | `setup`, `auth-status` |

## Setup

Requires [Node.js](https://nodejs.org) (v24+), a running Caido instance and a [PAT](https://docs.caido.io/dashboard/guides/create_pat.html).

```bash
# Install dependencies
npm install

# 1. Create a PAT in Dashboard → Developer → Personal Access Tokens
# 2. Setup (validates PAT via SDK and caches access token)
caido setup <your-pat>

# 3. Verify it works
caido health
caido recent --limit 1

# Or use env var instead
export CAIDO_PAT=<your-pat>
```

The `setup` command uses the SDK's device code flow (auto-approved by your PAT) to obtain an access token, then saves both the PAT and cached token to `~/.claude/config/secrets.json` via a custom `TokenCache` implementation. Subsequent runs load the cached token directly.

## File Structure

```
caido-client.ts          # CLI entry point — arg parsing + command dispatch
lib/
  client.ts              # SDK Client singleton, SecretsTokenCache, auth config
  graphql.ts             # gql documents for features not yet in SDK
  output.ts              # Output formatting (truncation, headers-only, raw→curl)
  types.ts               # Shared types (OutputOpts)
  commands/
    requests.ts          # search, recent, get, get-response, export-curl
    replay.ts            # replay, send-raw, edit, sessions, collections, automate, fuzz
    findings.ts          # findings, get-finding, create-finding, update-finding
    management.ts        # scopes, filters, environments, projects, hosted-files, tasks
    intercept.ts         # intercept-status, intercept-enable, intercept-disable
    info.ts              # viewer, plugins, health, setup, auth-status
```

## Usage

All commands output JSON. Run `caido --help` for the complete list.

### Search & Browse

```bash
# Search with HTTPQL (Caido's query language)
caido search 'req.method.eq:"POST" AND resp.code.eq:200'
caido search 'req.host.cont:"api"' --limit 50

# Get recent requests
caido recent --limit 10

# Full request details with raw HTTP
caido get <request-id>

# Just the response
caido get-response <request-id>
```

### Edit & Replay (the main feature)

Take an existing authenticated request and modify only what you need. Cookies, auth headers, User-Agent — everything else is preserved.

```bash
# Change the path (IDOR testing)
caido edit <id> --path /api/user/999

# Change method + body (privilege escalation)
caido edit <id> --method POST --body '{"role":"admin"}'

# Add/remove headers (bypass testing)
caido edit <id> --set-header "X-Forwarded-For: 127.0.0.1"
caido edit <id> --remove-header "X-CSRF-Token"

# Find/replace text anywhere in the request
caido edit <id> --replace "user123:::user456"
```

### Export to curl

```bash
caido export-curl <request-id>
```

### Findings

```bash
caido findings
caido get-finding <finding-id>
caido create-finding <request-id> \
  --title "IDOR in user profile" \
  --description "Can access other users' data" \
  --reporter "rez0"
caido update-finding <finding-id> --title "Updated title"
```

### Scopes

```bash
caido scopes
caido create-scope "Target" --allow "*.target.com" --deny "*.cdn.target.com"
caido update-scope <id> --allow "*.target.com,*.api.target.com"
caido delete-scope <id>
```

### Filter Presets

```bash
caido filters
caido create-filter "API Errors" --query 'req.path.cont:"/api/" AND resp.code.gte:400'
caido create-filter "Auth" --query 'req.path.regex:"/(login|auth)/"' --alias "auth"
caido delete-filter <id>
```

### Environments

```bash
caido envs
caido create-env "IDOR-Test"
caido env-set <env-id> victim_id "user_456"
caido select-env <env-id>
caido delete-env <id>
```

### Sessions & Collections

```bash
caido create-session <request-id>
caido rename-session <session-id> "idor-user-profile"
caido replay-sessions
caido delete-sessions <id1>,<id2>

caido replay-collections
caido create-collection "IDOR Tests"
caido rename-collection <id> "Auth Bypass"
caido delete-collection <id>
```

### Fuzzing

```bash
caido create-automate-session <request-id>
# Configure payload markers and wordlists in Caido UI first
caido fuzz <session-id>
```

### Tasks, Projects, Info & Health

```bash
caido tasks
caido cancel-task <task-id>
caido projects
caido select-project <id>
caido viewer
caido plugins
caido health
```

### Intercept Control

```bash
caido intercept-status
caido intercept-enable
caido intercept-disable
```

### Output Control

| Flag | Default | Description |
|------|---------|-------------|
| `--max-body <n>` | 200 | Max response body lines (0 = unlimited) |
| `--max-body-chars <n>` | 5000 | Max response body chars (0 = unlimited) |
| `--no-request` | off | Skip request raw in output |
| `--headers-only` | off | Show only HTTP headers, no body |
| `--compact` | off | Shorthand for `--no-request --max-body 50` |

## HTTPQL Quick Reference

Caido's query language for searching HTTP history. String values must be quoted, integers are not.

```
req.method.eq:"POST"                          # Match method
req.host.cont:"api"                           # Host contains
req.path.regex:"/users/[0-9]+/"               # Regex on path
resp.code.gte:400                             # Status code range
resp.len.gt:100000                            # Large responses
"password" OR "secret"                        # Search req+resp raw
req.method.eq:"POST" AND resp.code.eq:200     # Combine with AND/OR
source:"replay"                               # Filter by source
preset:"My Filter"                            # Use saved filter preset
```

## Architecture

Built on `@caido/sdk-client` v0.1.4+. Multi-file architecture with clean separation:

- **High-level SDK methods** for most features (requests, replay, findings, scopes, filters, environments, projects, hosted files, tasks, user)
- **`client.graphql.query()`/`mutation()`** with `gql` tagged templates for features not yet in SDK (intercept, plugins, automate/fuzz)
- **No raw fetch anywhere** — everything goes through the SDK

## Claude Code Integration

This repo is designed to work as a [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code). The `SKILL.md` file provides Claude with full context on how to use every command, HTTPQL syntax, and testing workflows.

To install as a skill:

```bash
cp -r . ~/.claude/skills/caido-mode/
cd ~/.claude/skills/caido-mode && npm install
```

## License

MIT
