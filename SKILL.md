---
name: bootstrap-agent
description: Initialize or refresh an AI agent environment with the recommended skills, CLI tools, and API key configuration. Detects which agent(s) are present (Pi, Claude Code, OpenCode, Codex, Cursor, AiderDesk, etc.), audits what's already installed, recommends what's missing, and installs from official GitHub sources. Use when the user says "init my agent", "set up", "bootstrap", "configure my environment", "开箱", "初始化", "补全环境", or runs on a fresh machine. Safe to re-run; fully idempotent.
---

# Bootstrap Agent Environment

A meta-skill that initializes any AI agent environment. It audits the current state, detects the active agent(s), asks the user a small set of questions, then installs skills, tools, and API keys from official sources.

Designed to be:
- **Agent-agnostic**: works on Pi, Claude Code, OpenCode, Codex, Cursor, AiderDesk, and any agent that reads from `~/.agents/skills/`.
- **Idempotent**: re-running on a fully-configured system is a no-op.
- **Flexible on API keys**: can install skills first, configure keys later. Missing keys never block installation.

## When to use

Activate when the user says any of:
- English: "init my agent", "set up", "bootstrap", "configure my environment", "what's missing", "refresh my skills"
- Chinese: "开箱", "初始化", "配置 agent", "补全环境", "装一下默认的"

Do NOT use when:
- The user only wants to install one specific skill (use the skill's own instructions)
- The user is debugging an existing skill (use the skill's SKILL.md as reference)

## Core principles

1. **Universal first**: always copy physical files to `~/.agents/skills/`. This is the path 14+ agents read from directly. Other agent paths are symlinks/views.
2. **Ask once, flexibly**: collect all API keys in one prompt, but allow the user to skip any or all. Missing keys never block installation.
3. **Never destructive**: never delete or overwrite existing files. If something is in the way, warn and skip.
4. **Latest from source**: pull from GitHub directly, not via `npx skills add`. The npm registry has version lag.
5. **Mask secrets in output**: never echo full API keys. Show only `bce-v3/XXXXXXXX-...xxxxxxxx` style previews.

## Workflow

Run these 8 steps in order. After each step, briefly report findings before moving to the next.

---

### Step 1 — Audit current state

Run all of these checks and collect results. Report in a compact table.

```bash
# Universal skill location
echo "=== ~/.agents/skills/ ==="
ls ~/.agents/skills/ 2>/dev/null

# Agent-specific skill locations
for path in ~/.pi/agent/skills ~/.claude/skills ~/.config/opencode/skills ~/.codex/skills ~/.cursor/skills ~/.aider-desk/skills ~/.gemini/skills; do
  if [ -d "$(dirname "$path")" ]; then
    echo "=== $path ==="
    ls "$path" 2>/dev/null
  fi
done

# CLI tools
echo "=== CLI tools ==="
for tool in mmx anysearch npx git python3 npm; do
  if command -v "$tool" >/dev/null 2>&1; then
    echo "  ✓ $tool: $(command -v $tool)"
  else
    echo "  ✗ $tool: missing"
  fi
done

# API keys currently in env
echo "=== API keys in env ==="
env | grep -iE 'API_KEY|TOKEN' | sort

# API keys declared in bashrc
echo "=== API keys in ~/.bashrc ==="
grep -iE 'API_KEY|TOKEN' ~/.bashrc 2>/dev/null

# Cached source repos
echo "=== Cached source repos ==="
ls -d /tmp/*-skills 2>/dev/null
```

---

### Step 2 — Detect active agent(s)

Run multiple signals in parallel; collect all that match.

```bash
# Config directories
echo "=== Agent config dirs ==="
for dir in ~/.pi/agent ~/.claude ~/.config/opencode ~/.codex ~/.cursor ~/.aider-desk ~/.gemini; do
  [ -d "$dir" ] && echo "  DIR: $dir"
done

# Environment variables
echo "=== Agent env vars ==="
env | grep -iE '^(PI|CLAUDE|CODEX|OPENCODE|CURSOR|AIDER)_' | sort

# CLI binaries on PATH
echo "=== Agent CLI binaries ==="
for cmd in pi claude opencode codex cursor aider; do
  command -v "$cmd" >/dev/null 2>&1 && echo "  BIN: $cmd"
done

# Parent process (the current agent's runtime)
echo "=== Parent process ==="
ps -o comm= -p "$PPID" 2>/dev/null
```

Map detected signals to agents using this table:

| Agent | Detection signals | Skill install path | Needs symlink? |
|-------|-------------------|--------------------|----------------|
| Pi | `~/.pi/agent/`, `$PI_*` env, `pi` binary | `~/.pi/agent/skills/` | ✓ |
| Claude Code | `~/.claude/`, `$CLAUDE_*` env, `claude` binary | `~/.claude/skills/` | ✓ |
| OpenCode | `~/.config/opencode/`, `$OPENCODE_*` env, `opencode` binary | `~/.agents/skills/` (universal) | ✗ |
| Codex | `~/.codex/`, `$CODEX_*` env, `codex` binary | `~/.agents/skills/` (universal) | ✗ |
| Cursor | `~/.cursor/` | `~/.cursor/skills/` | ✓ |
| AiderDesk | `~/.aider-desk/` | `~/.aider-desk/skills/` | ✓ |
| Gemini CLI | `~/.gemini/` | `~/.agents/skills/` (universal) | ✗ |
| Cline / Amp / Antigravity / Zed / Warp / Copilot | various | `~/.agents/skills/` (universal) | ✗ |

**Decision rule:**
- 0 agents detected → install to `~/.agents/skills/` only, note "no agent detected"
- 1+ detected → proceed to Step 3 with the full detected list (will ask user in Step 4 which to target)

---

### Step 3 — Diff against default catalog

Default catalog (the recommended set):

| Category | Skill/Tool | Source | Auth | Env var | python_deps |
|----------|-----------|--------|------|---------|-------------|
| 🔍 Search | `anysearch` | (preinstalled only) | `preinstalled` | (per provider, see anysearch SKILL.md) | (n/a) |
| 🎬 Media | `mmx-cli` | npm global | `key\|oauth` | `MINIMAX_API_KEY` | (none) |
| 📄 Documents | `docx` | github.com/anthropics/skills | `none` | — | `python-docx` |
| | `pdf` | github.com/anthropics/skills | `none` | — | `pypdf` |
| | `pptx` | github.com/anthropics/skills | `none` | — | `python-pptx` |
| | `xlsx` | github.com/anthropics/skills | `none` | — | `openpyxl` |
| 💬 Writing | `doc-coauthoring` | github.com/anthropics/skills | `none` | — | (none) |
| | `internal-comms` | github.com/anthropics/skills | `none` | — | (none) |
| 🎨 Design | `canvas-design` | github.com/anthropics/skills | `none` | — | (none) |
| | `frontend-design` | github.com/anthropics/skills | `none` | — | (none) |
| | `brand-guidelines` | github.com/anthropics/skills | `none` | — | (none) |
| | `theme-factory` | github.com/anthropics/skills | `none` | — | (none) |
| 🔧 Meta | `skill-creator` | github.com/anthropics/skills | `none` | — | (none) |

**Auth types** (column meaning):

| Auth value | Meaning | Detection (configured = skip) | Configuration action |
|------------|---------|-------------------------------|---------------------|
| `none` | No auth needed | (always skip) | (none) |
| `env-var` | Single env var | `bash -c 'source ~/.bashrc 2>/dev/null; [ -n "\$VAR_NAME" ]'` or `grep -q "^export VAR_NAME=" ~/.bashrc` | Q5 ask → write to `~/.bashrc` (Step 7) |
| `key\|oauth` | Either env var or OAuth | `mmx auth status >/dev/null 2>&1` (logged in) **or** env var set | Q5 ask user to pick: OAuth (browser) or API key. TTY-aware default: interactive → OAuth, non-interactive → key |
| `oauth` | OAuth required | `mmx auth status >/dev/null 2>&1` (logged in) | Q5 prompt: run the login command (e.g., `mmx auth login`) |
| `preinstalled` | Not installed by this skill | (always skip) | (none — see target skill's own docs) |

**Status per item:** for each catalog entry, classify as one of:
- ✅ installed & configured
- ⚠️ installed but missing API key
- ❌ not installed

Present as a table to the user. This becomes the recommendation they confirm in Step 4.

**Extra skills found** (not in default catalog, just for awareness):

After building the catalog diff, scan `~/.agents/skills/` and list any items not in the catalog (e.g., `minimax-*`, custom user skills). Report them as a short note, don't try to install or remove them:

> ℹ️  Found N skills not in default catalog: `[minimax-docx, minimax-pdf, minimax-multimodal-toolkit, minimax-music-gen, minimax-xlsx, minimax-skills]`. Leaving them as-is.

---

### Step 4 — Ask the user (in this order)

Ask questions one at a time. Every question has a sensible default (press enter to accept).

**Q1: Target agents**
> Detected agents: `[pi, claude-code]`. Configure skills for which?
> (default: all detected, comma-separated, or "all" for everything the catalog supports)

**Q2: Skills to install**
> Missing skills: `[pdf, docx, mmx-cli, ...]`. Install which?
> (default: all missing, or comma-separated, or "all" for the full catalog)

**Q3: CLI tools to install**
> Missing CLI tools: `[mmx]`. Install which?
> (default: all missing)

**Q4: API key storage location**
> Where to store API keys?
> 1. `~/.bashrc` as env var (recommended for local dev) — DEFAULT
> 2. `~/.env` file (recommended for containers/CI, needs `set -a; source ~/.env; set +a` discipline)
> 3. Skill's own config.json (not recommended; leaks easily, hard to migrate)
> Pick 1/2/3:

**Q5: Auth (detect first, then ask)**

For each catalog entry with Auth in {`env-var`, `key|oauth`, `oauth`}, run the detection command from the Auth types table above. Group results into two buckets:

- **Bucket A — already configured** (detection succeeds): mark ✅, do NOT ask. Just report in the final summary.
- **Bucket B — needs configuration**: handle by Auth type:
  - `env-var`: ask for the value, write to `~/.bashrc` (Step 7). Press Enter to skip.
  - `key|oauth`: ask the user to pick. **TTY-aware default**:
    ```bash
    if [[ -t 0 ]]; then default=OAuth; else default=key; fi
    ```
    Display: `mmx-cli: OAuth (browser) or API key? [$default]`. Enter accepts default, type `skip` to skip entirely.
  - `oauth`: print the login command (e.g., `mmx auth login`) and ask the user to run it manually. Press Enter to skip.

For `none` and `preinstalled` Auth types: skip entirely (no detection, no prompt).

**Important:** every prompt in Q5 is fully optional. The user can press Enter to skip everything, and the skill still installs. They'll just need to configure auth manually before those features work.

**Q6: Confirm to proceed**
> About to: install N skills, M CLI tools, configure K API keys, create L symlinks.
> Proceed? (Y/n)

---

### Step 5 — Install skills

For each skill in the install list:

```bash
# 1. Clone source repo if not cached
case "<skill_source>" in
  github.com/anthropics/skills)
    [ -d /tmp/anthropics-skills ] || git clone --depth 1 https://github.com/anthropics/skills /tmp/anthropics-skills
    src_dir="/tmp/anthropics-skills/skills/<skill>"
    ;;
  preinstalled-only)
    # e.g., anysearch — preinstalled by another tool, not installable from this skill
    echo "  ⏭  <skill>: preinstalled-only, skipping (not in scope for this skill)"
    continue
    ;;
  *)
    echo "Unknown source for <skill>, skipping"
    continue
    ;;
esac

# 2. Copy to universal path (skip if already exists — never overwrite)
if [ -d ~/.agents/skills/<skill> ]; then
  echo "  ⏭  <skill>: already in ~/.agents/skills/, skipping"
else
  cp -r "$src_dir" ~/.agents/skills/
  echo "  ✓ <skill>: installed to ~/.agents/skills/"
fi

# 3. Create symlink for each target agent that needs one
for agent_path in ~/.pi/agent/skills ~/.claude/skills ~/.cursor/skills ~/.aider-desk/skills; do
  [ -d "$(dirname "$agent_path")" ] || continue  # agent not present
  if [ -L "$agent_path/<skill>" ]; then
    :  # symlink already exists, fine
  elif [ -e "$agent_path/<skill>" ]; then
    echo "  ⚠️  $agent_path/<skill> exists as real dir, not overwriting (your data is safe)"
  else
    mkdir -p "$agent_path"
    ln -s ~/.agents/skills/<skill> "$agent_path/<skill>"
    echo "  ✓ symlink: $agent_path/<skill>"
  fi
done
```

**Skip the symlink step entirely** for agents marked "✗" in the detection table (OpenCode, Codex, Cline, Amp, etc.) — they read `~/.agents/skills/` directly.

---

### Step 5.5 — Install Python dependencies

For each installed skill that has `python_deps` declared in the catalog, install via `pip3`. Run after Step 5 (skill files in place), before Step 6 (CLI tools).

```bash
# Flatten the catalog's python_deps into a deduplicated list.
# Example: for the default catalog → requests, python-docx, pypdf, python-pptx, openpyxl
declare -a PYTHON_DEPS=(
  # ← populated from catalog's python_deps column, deduped
)

for pkg in "${PYTHON_DEPS[@]}"; do
  [ -z "$pkg" ] && continue
  if pip3 show "$pkg" >/dev/null 2>&1; then
    echo "  ✓ $pkg: already installed"
    continue
  fi
  # Three-layer fallback: user → user+break-system-packages (PEP 668) → manual
  pip3 install --user "$pkg" 2>/dev/null \
    || pip3 install --user --break-system-packages "$pkg" 2>/dev/null \
    || echo "  ⚠️  $pkg failed, install manually: pip3 install --user $pkg"
done
```

**Why `pip show` for detection, not `import`?** Some packages have a different import name from their pip name (e.g., `python-docx` is imported as `docx`). `pip show` works uniformly.

### Step 6 — Install CLI tools

For each tool in the install list:

| Tool | Install command |
|------|-----------------|
| `mmx-cli` | `npm install -g mmx-cli` |
| `anysearch` | `npx skills add <source-url> --skill anysearch --yes --global` |

After install, verify with `command -v <tool>`.

---

### Step 7 — Configure API keys

⚠️ **CRITICAL CONSTRAINT:** env vars MUST be written to `~/.bashrc` BEFORE the `case $- in ... return` block (the standard Debian non-interactive guard). Otherwise, non-interactive shells — which AI agents and many CLI tools use internally — won't see them.

First, check the bashrc structure:

```bash
grep -n '^case \$- in' ~/.bashrc
```

If a case line exists at line N, insert exports BEFORE it. Otherwise, append to the end.

```bash
for entry in "FOO_API_KEY:value1" "BAR_API_KEY:value2"; do
  key="${entry%%:*}"
  value="${entry#*:}"
  [ -z "$value" ] && continue  # user skipped this key

  # Skip if already exported (idempotent)
  if grep -q "^export $key=" ~/.bashrc; then
    echo "  ⏭  $key: already in ~/.bashrc, skipping"
    continue
  fi

  case_line=$(grep -n '^case \$- in' ~/.bashrc | head -1 | cut -d: -f1)
  if [ -n "$case_line" ]; then
    # Defensive: normalize CRLF to LF (some Windows-edited bashrc have CRLF, which breaks sed insertion)
    if file ~/.bashrc | grep -q CRLF; then
      dos2unix -q ~/.bashrc 2>/dev/null || sed -i 's/\r$//' ~/.bashrc
      # Re-locate the case line after normalization
      case_line=$(grep -n '^case \$- in' ~/.bashrc | head -1 | cut -d: -f1)
    fi
    # Insert BEFORE the case line
    sed -i "${case_line}i\\
export $key=\"$value\"" ~/.bashrc
    # Verify and fallback if sed didn't take
    if ! grep -q "^export $key=" ~/.bashrc; then
      echo "export $key=\"$value\"" >> ~/.bashrc
    fi
    echo "  ✓ $key: configured in ~/.bashrc (before line $case_line)"
  else
    # No guard, append
    echo "export $key=\"$value\"" >> ~/.bashrc
    echo "  ✓ $key: appended to ~/.bashrc"
  fi
done
```

If the user chose storage option 2 (`~/.env`):
```bash
touch ~/.env && chmod 600 ~/.env
echo 'export FOO_API_KEY="value"' >> ~/.env
```

**Mask the value when echoing** to the user — show only first 13 + last 5 chars: `${value:0:13}...${value: -5}`.

---

### Step 8 — Verify

```bash
# 1. Env vars accessible in non-interactive shell
echo "=== Env var check (non-interactive shell) ==="
bash -c 'source ~/.bashrc 2>/dev/null; env | grep -E "API_KEY|TOKEN" | sort'

# 2. Skill files exist
echo "=== Skill files ==="
for skill in <installed_skills>; do
  if [ -d ~/.agents/skills/"$skill" ]; then
    echo "  ✓ $skill"
  else
    echo "  ✗ $skill: MISSING"
  fi
done

# 3. Test a skill that has a runnable script
echo "=== Skill smoke test (mmx-cli) ==="
mmx auth status 2>&1 | head -5

# 4. Test a CLI tool
echo "=== CLI smoke test (mmx) ==="
mmx --help 2>&1 | head -5
```

Report any failures. Print a final summary table:

```
| Action | Status |
|--------|--------|
| Skills installed | N |
| CLI tools installed | M |
| API keys configured | K |
| Symlinks created | L |
| Skills needing keys (skipped) | X |
| Verification | ✓ all pass / ⚠️ N issues |
```

---

## API key sources

| Service | Where to get the key |
|---------|---------------------|
| MiniMax | https://platform.MiniMax.io (refer to mmx-cli docs) |
| AnySearch | (per-provider, see anysearch SKILL.md) |

---

## Adding new skills to the catalog

To extend the default catalog (e.g., when new skills are released):

1. Edit the "Default catalog" table in this file (Step 3).
2. If the source isn't `github.com/anthropics/skills`, add a new `case` branch in Step 5.
3. If the skill needs an API key, add it to:
   - The catalog table (with env var name and format)
   - The "API key sources" table at the bottom
   - The Q5 prompt template (Step 4)

---

## Notes

- **Why not `npx skills add`?** It works, but it goes through the npm registry which can lag behind GitHub releases. Pulling source from GitHub directly guarantees the latest.
- **Why universal-first install?** 14+ agents already read `~/.agents/skills/` directly. By making that the canonical physical location, we get cross-agent compatibility for free. Other paths are just symlinks.
- **Why ask all keys at once?** Better UX than asking per-skill mid-flow. The user can always skip — that's why missing keys never block installation.
- **Caching:** source repos in `/tmp/` survive across runs. Subsequent bootstraps skip the git clone. Offer to clean them up at the end if size matters.
- **Python dependencies:** skills that need Python packages declare them in the catalog's `python_deps` column. Step 5.5 auto-installs them with `pip3 install --user` (with fallbacks for PEP 668 systems and missing pip). If auto-install fails, the user is warned and given the manual install command. To opt out for a given skill, leave `python_deps` empty (`(none)`) in the catalog.
