# Copilot CLI Plugins & Internal Marketplace — Enterprise Guide

> Compiled from official GitHub docs, validated by Opus 4.6 + GPT-5.4

---

## Step 1: Understand What a Plugin Is

A plugin is a **distributable, installable package** that bundles any combination of:

| Component        | Files                     | What It Does                                    |
|------------------|---------------------------|-------------------------------------------------|
| Custom Agents    | `*.agent.md` in `agents/` | Specialized AI assistants with persona + tools  |
| Skills           | `SKILL.md` in `skills/X/` | Discrete callable capabilities with instructions|
| Hooks            | `hooks.json`              | Event handlers (session start, pre/post tool)   |
| MCP Servers      | `.mcp.json`               | External tool integrations via MCP protocol     |
| LSP Servers      | `lsp.json`                | Language intelligence (go-to-def, diagnostics)  |
| Commands         | via `commands` field       | Custom CLI commands                             |

---

## Step 2: Create the Plugin Directory Structure

### Minimal plugin
```
my-plugin/
└── plugin.json          # Required — the only mandatory file
```

### Full enterprise plugin
```
contoso-engineering/
├── plugin.json          # Required manifest
├── agents/
│   ├── security-auditor.agent.md
│   └── incident-responder.agent.md
├── skills/
│   ├── pr-review/
│   │   └── SKILL.md
│   └── release-notes/
│       └── SKILL.md
├── hooks.json
├── .mcp.json
└── lsp.json
```

---

## Step 3: Write `plugin.json`

### Schema Reference

**Required:**
| Field  | Type   | Rules                                       |
|--------|--------|---------------------------------------------|
| `name` | string | Kebab-case, letters/numbers/hyphens, max 64 |

**Optional metadata:**
| Field         | Type     | Description                         |
|---------------|----------|-------------------------------------|
| `description` | string   | Max 1024 chars                      |
| `version`     | string   | Semver (e.g., `1.2.0`)             |
| `author`      | object   | `{ name, email?, url? }`           |
| `homepage`    | string   | URL                                 |
| `repository`  | string   | Source repo URL                     |
| `license`     | string   | e.g., `MIT`, `UNLICENSED`          |
| `keywords`    | string[] | Search keywords                     |
| `category`    | string   | Plugin category                     |
| `tags`        | string[] | Additional tags                     |

**Component paths:**
| Field        | Type              | Default     | Description                          |
|--------------|-------------------|-------------|--------------------------------------|
| `agents`     | string \| string[] | `agents/`   | Directories with `*.agent.md` files  |
| `skills`     | string \| string[] | `skills/`   | Directories with `SKILL.md` files    |
| `commands`   | string \| string[] | —           | Custom command directories            |
| `hooks`      | string \| object  | —           | Path to hooks.json or inline object  |
| `mcpServers` | string \| object  | —           | Path to .mcp.json or inline object   |
| `lspServers` | string \| object  | —           | Path to lsp.json or inline object    |

### Complete Example
```json
{
  "name": "contoso-engineering",
  "description": "Contoso engineering standards and tooling",
  "version": "1.0.0",
  "author": {
    "name": "Platform Team",
    "email": "platform@contoso.com"
  },
  "repository": "https://github.com/contoso/copilot-plugins",
  "license": "UNLICENSED",
  "keywords": ["enterprise", "security", "standards"],
  "agents": "agents/",
  "skills": ["skills/", "extra-skills/"],
  "hooks": "hooks.json",
  "mcpServers": ".mcp.json",
  "lspServers": "lsp.json"
}
```

The CLI searches for `plugin.json` in this order:
1. `.plugin/plugin.json`
2. `plugin.json` (root)
3. `.github/plugin/plugin.json`
4. `.claude-plugin/plugin.json`

---

## Step 4: Build Each Component

### 4a. Custom Agent — `agents/security-auditor.agent.md`
```markdown
---
name: security-auditor
description: Reviews code for security vulnerabilities. Use when security review is requested.
tools: ["bash", "view", "grep", "glob"]
---

You are a security auditor. When reviewing code:
1. Check for hardcoded secrets and credentials
2. Check for SQL injection, XSS, and command injection
3. Verify authentication and authorization patterns
4. Report findings with severity levels (Critical/High/Medium/Low)
```

### 4b. Skill — `skills/pr-review/SKILL.md`
```markdown
---
name: pr-review
description: Standardized PR review following Contoso guidelines. Use when reviewing PRs.
allowed-tools: shell
---

When performing a PR review:
1. Check that the PR has a linked work item
2. Verify test coverage meets the 80% threshold
3. Check for breaking API changes
4. Ensure CHANGELOG.md is updated
5. Run `./scripts/lint-check.sh` from this skill's directory
```

> ⚠️ **Security note:** `allowed-tools: shell` pre-approves shell execution.
> Only use this for skills you fully trust and have reviewed.

### 4c. Hooks — `hooks.json`
```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "type": "command",
        "bash": "echo \"[$(date -u +%Y-%m-%dT%H:%M:%SZ)] Session started\" >> ~/.copilot/audit.log",
        "powershell": "Add-Content ~/.copilot/audit.log \"[$(Get-Date -Format o)] Session started\"",
        "timeoutSec": 5
      }
    ],
    "preToolUse": [
      {
        "type": "command",
        "bash": "echo \"Tool: $TOOL_NAME\" >> ~/.copilot/audit.log",
        "timeoutSec": 5
      }
    ],
    "postToolUse": [],
    "sessionEnd": [],
    "userPromptSubmitted": [],
    "errorOccurred": []
  }
}
```

Available hook triggers:
- `sessionStart` / `sessionEnd`
- `userPromptSubmitted`
- `preToolUse` / `postToolUse`
- `errorOccurred` / `agentStop` / `subagentStop`

**Enterprise power feature:** `preToolUse` hooks can **deny** actions:
```json
{"permissionDecision": "deny", "permissionDecisionReason": "Dangerous command detected"}
```

### 4d. MCP Servers — `.mcp.json`
```json
{
  "mcpServers": {
    "contoso-internal-api": {
      "type": "http",
      "url": "https://mcp.internal.contoso.com/mcp",
      "headers": { "Authorization": "Bearer ${CONTOSO_MCP_TOKEN}" },
      "tools": ["*"]
    },
    "playwright": {
      "type": "local",
      "command": "npx",
      "args": ["@playwright/mcp@latest"],
      "tools": ["*"]
    }
  }
}
```

Server types: `local`/`stdio` (local process), `http` (Streamable HTTP), `sse` (legacy SSE).

### 4e. LSP Servers — `lsp.json`
```json
{
  "lspServers": {
    "typescript": {
      "command": "typescript-language-server",
      "args": ["--stdio"],
      "fileExtensions": { ".ts": "typescript", ".tsx": "typescript" }
    },
    "python": {
      "command": "pylsp",
      "args": [],
      "fileExtensions": { ".py": "python" }
    }
  }
}
```

---

## Step 5: Test the Plugin Locally

```bash
# Install from local directory
copilot plugin install ./contoso-engineering

# Verify installation
copilot plugin list

# Inside an interactive session, verify components loaded:
/agent                  # Should show your custom agents
/skills list            # Should show your skills
/mcp                    # Should show your MCP servers

# After making changes, reinstall (plugins are cached!)
copilot plugin install ./contoso-engineering

# Uninstall when done testing
copilot plugin uninstall contoso-engineering
```

---

## Step 6: Create the Internal Marketplace

### 6a. Marketplace Repository Structure

```
contoso/copilot-enterprise-toolkit/
├── .github/
│   └── plugin/
│       └── marketplace.json       # Marketplace manifest
├── plugins/
│   ├── contoso-engineering/
│   │   ├── plugin.json
│   │   ├── agents/
│   │   ├── skills/
│   │   ├── hooks.json
│   │   └── .mcp.json
│   ├── frontend-toolkit/
│   │   ├── plugin.json
│   │   ├── agents/
│   │   └── skills/
│   └── incident-response/
│       ├── plugin.json
│       └── agents/
└── README.md
```

### 6b. Write `marketplace.json`

**Schema:**

| Field      | Type   | Required | Description                                  |
|------------|--------|----------|----------------------------------------------|
| `name`     | string | **Yes**  | Kebab-case, max 64 chars                     |
| `owner`    | object | **Yes**  | `{ name, email? }`                           |
| `plugins`  | array  | **Yes**  | List of plugin entries                        |
| `metadata` | object | No       | `{ description?, version?, pluginRoot? }`    |

**Per-plugin entry:**

| Field    | Type             | Required | Description                               |
|----------|------------------|----------|-------------------------------------------|
| `name`   | string           | **Yes**  | Plugin name, must match plugin.json        |
| `source` | string \| object | **Yes**  | Relative path, GitHub ref, or URL          |
| Other    |                  | No       | Same metadata fields as plugin.json        |
| `strict` | boolean          | No       | Default `true`. Set `false` for relaxed validation |

**Complete example:**
```json
{
  "name": "contoso-enterprise",
  "owner": {
    "name": "Contoso Platform Team",
    "email": "platform@contoso.com"
  },
  "metadata": {
    "description": "Official Contoso engineering plugins for Copilot CLI",
    "version": "2.0.0"
  },
  "plugins": [
    {
      "name": "contoso-engineering",
      "description": "Core engineering standards: security audit, PR review, incident response",
      "version": "1.0.0",
      "author": { "name": "Platform Team" },
      "license": "UNLICENSED",
      "keywords": ["enterprise", "security", "standards"],
      "source": "./plugins/contoso-engineering"
    },
    {
      "name": "frontend-toolkit",
      "description": "React/TypeScript development tools following Contoso design system",
      "version": "2.1.0",
      "author": { "name": "UI Team" },
      "keywords": ["react", "design-system"],
      "source": "./plugins/frontend-toolkit"
    },
    {
      "name": "incident-response",
      "description": "Guided incident triage and response workflows",
      "version": "1.0.0",
      "author": { "name": "SRE Team" },
      "source": "./plugins/incident-response"
    }
  ]
}
```

Marketplace manifest search order:
1. `marketplace.json` (root)
2. `.plugin/marketplace.json`
3. `.github/plugin/marketplace.json`
4. `.claude-plugin/marketplace.json`

---

## Step 7: Host the Marketplace

### Option A: GitHub Repository (Recommended for most teams)
```bash
# Users register with owner/repo format
copilot plugin marketplace add contoso/copilot-enterprise-toolkit
```

### Option B: Any Git URL (Azure DevOps, GitLab, etc.)
```bash
copilot plugin marketplace add https://dev.azure.com/contoso/copilot-plugins/_git/marketplace
copilot plugin marketplace add https://gitlab.com/contoso/copilot-plugins.git
```

### Option C: Local / Shared Network Path (Air-gapped / on-prem)
```bash
# Linux/macOS
copilot plugin marketplace add /mnt/shared/copilot-marketplace

# Windows
copilot plugin marketplace add C:\shared\copilot-marketplace
```

### Pre-registered defaults
Copilot CLI ships with two marketplaces already registered:
- `copilot-plugins` → `github/copilot-plugins`
- `awesome-copilot` → `github/awesome-copilot`

---

## Step 8: Distribute to Your Team

### For plugin consumers (share these instructions):

```bash
# 1. Register the company marketplace (one-time)
copilot plugin marketplace add contoso/copilot-enterprise-toolkit

# 2. Browse available plugins
copilot plugin marketplace browse contoso-enterprise

# 3. Install a plugin
copilot plugin install contoso-engineering@contoso-enterprise

# 4. Verify everything loaded
copilot plugin list

# 5. Update later when new versions are published
copilot plugin update contoso-engineering

# 6. Uninstall if needed
copilot plugin uninstall contoso-engineering

# 7. Remove marketplace (--force to also uninstall its plugins)
copilot plugin marketplace remove contoso-enterprise --force
```

### Interactive slash commands (inside a copilot session):
```
/plugin list
/plugin install contoso-engineering@contoso-enterprise
/plugin marketplace list
/plugin marketplace add contoso/copilot-enterprise-toolkit
/plugin marketplace browse contoso-enterprise
```

---

## Step 9: Publishing Updates (No Separate Publish Command)

There is **no `copilot plugin publish` command**. Publishing is file-based:

1. Update plugin files in the marketplace repo
2. Bump `version` in both `plugin.json` and `marketplace.json`
3. Commit and push
4. Users run `copilot plugin update PLUGIN-NAME` to pull latest

### Recommended CI/CD workflow:
```
1. PR with plugin changes → code review (CODEOWNERS on marketplace repo)
2. Merge to main → version bump in marketplace.json
3. Tag release (e.g., v1.2.0)
4. Announce to eng team → users run `copilot plugin update`
```

---

## Precedence & Collision Rules (Critical for Enterprise)

```
┌──────────────────────────────────────────────────────────────────────┐
│  BUILT-IN (always present, CANNOT be overridden)                     │
│  • tools: bash, view, edit, glob, grep, task, ...                    │
│  • agents: explore, task, code-review, general-purpose, ...          │
└───────────────────────┬──────────────────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────────────────┐
│  AGENTS — first-found-wins (dedup by ID from filename)               │
│  1. ~/.copilot/agents/              (user — HIGHEST priority)        │
│  2. <project>/.github/agents/       (project)                        │
│  3. <parents>/.github/agents/       (monorepo inherited)             │
│  4. PLUGIN agents/ dirs             (by install order)               │
│  5. Remote org/enterprise agents    (LOWEST priority)                │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  SKILLS — first-found-wins (dedup by name field in SKILL.md)         │
│  1. Project: .github/skills/                                         │
│  2. Personal: ~/.copilot/skills/                                     │
│  3. PLUGIN skills/ dirs                                              │
│  4. COPILOT_SKILLS_DIRS env var                                      │
└──────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────┐
│  MCP SERVERS — last-wins (dedup by server name)                      │
│  1. ~/.copilot/mcp-config.json       (lowest priority)               │
│  2. .vscode/mcp.json                 (workspace)                     │
│  3. PLUGIN MCP configs               (plugins)                       │
│  4. --additional-mcp-config flag      (highest priority)             │
└──────────────────────────────────────────────────────────────────────┘
```

**Key takeaway:** Project/user agents/skills **shadow** plugin ones.
Plugin MCP servers **override** user-level ones.

---

## File Locations Reference

| Item                        | Path                                                             |
|-----------------------------|------------------------------------------------------------------|
| Installed plugins (market.) | `~/.copilot/installed-plugins/MARKETPLACE/PLUGIN-NAME/`          |
| Installed plugins (direct)  | `~/.copilot/installed-plugins/_direct/SOURCE-ID/`                |
| Marketplace cache (Windows) | `%LOCALAPPDATA%\copilot\marketplaces\`                           |
| Marketplace cache (macOS)   | `~/Library/Caches/copilot/marketplaces/`                         |
| Marketplace cache (Linux)   | `~/.cache/copilot/marketplaces/`                                 |
| Config directory            | `~/.copilot` (override: `COPILOT_HOME` env var)                  |

---

## Security Considerations for Enterprise

1. **Skills with `allowed-tools: shell`** — Pre-approves shell execution without confirmation.
   Only use for reviewed, trusted skills. A compromised skill = RCE on every dev machine.

2. **Hooks are powerful** — `preToolUse` hooks can deny actions (great for guardrails),
   but malicious hooks can also silently modify prompts and tool behavior.
   Treat hooks.json as privileged code.

3. **Name shadowing** — Users can override org agents/skills locally (user > project > plugin > org).
   Don't rely solely on naming for enforcement; use `preToolUse` hooks for guardrails.

4. **MCP servers** — Can access internal systems. Enforce least-privilege,
   audit enabled tools, and control secrets injection carefully.

5. **Supply chain** — Pin versions, use CODEOWNERS/branch protection on marketplace repos,
   require PR review for all plugin changes.

## Governance Recommendations

- **Branch protection** on the marketplace repo — require PR reviews for all changes
- **CODEOWNERS** — Platform/security team owns `marketplace.json` and `hooks.json`
- **Semver discipline** — bump major for breaking changes, communicate via changelog
- **Staging org** — test plugin changes in a staging GitHub org before production
- **Audit logging** — use `sessionStart` hooks to log usage for compliance
- **No built-in canary/gradual rollout** — use separate "stable" vs "canary" marketplace names,
  or version-based plugin names for staged rollout
