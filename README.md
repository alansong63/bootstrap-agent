# bootstrap-agent

> Initialize any AI agent environment with the recommended skills, CLI tools, and API key configuration. One trigger phrase; the AI handles the rest.

A meta-skill for AI coding agents (Pi, Claude Code, OpenCode, Codex, Cursor, AiderDesk, …) that:

- 🔍 **Audits** what's already installed on the current machine
- 🎯 **Detects** which AI agent(s) are present
- 📋 **Recommends** what's missing from the default catalog
- ❓ **Asks** the user a small set of questions (skills to install, where to store keys, which keys to provide)
- 🔧 **Installs** skills and CLI tools from official GitHub sources
- 🔑 **Configures** API keys in `~/.bashrc` (works for non-interactive shells too)
- ✅ **Verifies** everything works end-to-end

Designed to be **agent-agnostic**, **idempotent** (safe to re-run), and **flexible on API keys** (you can install first, configure keys later — missing keys never block installation).

## When to use

Trigger this skill by saying any of:

- **English**: `init my agent`, `set up`, `bootstrap`, `configure my environment`, `what's missing`, `refresh my skills`
- **中文**: `开箱`, `初始化`, `配置 agent`, `补全环境`

## How it works

The skill executes 8 steps (see [SKILL.md](./SKILL.md) for the full playbook):

1. **Audit** current state (skills, CLIs, API keys, cached repos)
2. **Detect** active agent(s) via config dirs, env vars, and binaries
3. **Diff** against the default catalog (14 recommended items)
4. **Ask** the user: which agents, which skills, where to store keys, which keys
5. **Install** skills — clone from GitHub → copy to `~/.agents/skills/` → symlink to agent paths
6. **Install** CLI tools (npm global)
7. **Configure** API keys in `~/.bashrc` (BEFORE the `case $- in` guard so non-interactive shells see them)
8. **Verify** with smoke tests

## Installation

Like any other skill, clone the repo and place (or symlink) the resulting `bootstrap-agent/` directory wherever your agent looks for skills. The exact path depends on your client — check your agent's documentation if you're unsure.

```bash
git clone https://github.com/alansong63/bootstrap-agent
# Then move or symlink the bootstrap-agent/ directory into your agent's skill location
```

## Default catalog

| Category | Items | Auth |
|----------|-------|------|
| 🔍 Search | `anysearch` | `preinstalled` |
| 🎬 Media | `mmx-cli` | `key\|oauth` (or `MINIMAX_API_KEY`) |
| 📄 Documents | `docx` `pdf` `pptx` `xlsx` | `none` (needs `python-docx` / `pypdf` / `python-pptx` / `openpyxl`) |
| 💬 Writing | `doc-coauthoring` `internal-comms` | `none` |
| 🎨 Design | `canvas-design` `frontend-design` `brand-guidelines` `theme-factory` | `none` |
| 🔧 Meta | `skill-creator` | `none` |

See [SKILL.md](./SKILL.md) for the full catalog with `python_deps` and the Auth detection reference table.

Document/design/meta skills all come from [anthropics/skills](https://github.com/anthropics/skills).

## Design principles

1. **Universal first** — always install to `~/.agents/skills/`; other agent paths are symlinks
2. **Ask once, flexibly** — collect all API keys in one prompt; missing keys never block installation
3. **Never destructive** — never delete or overwrite existing files
4. **Latest from source** — pull from GitHub directly, not via `npx skills add` (which goes through npm registry with version lag)
5. **Mask secrets** — never echo full API keys in output; show only `bce-v3/XXXXXXXX-...xxxxxxxx` style previews

## Author

Maintained by [@alansong63](https://github.com/alansong63).
