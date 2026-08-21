---
name: gemini-notebook
description: Use this skill to query your Google NotebookLM notebooks directly from Claude Code or Cowork for source-grounded, citation-backed answers from Gemini. Browser automation, library management, persistent auth. Drastically reduced hallucinations through document-only responses.
---

# NotebookLM Research Assistant Skill

Interact with Google NotebookLM to query documentation with Gemini's source-grounded answers. Each question opens a fresh browser session, retrieves the answer exclusively from your uploaded documents, and closes.

## First-Time Setup (Do This Once)

This skill has no notebooks and no login yet. Do these steps in order, in a single session, before doing anything else with the skill.

### 1. Check whether login is needed

```bash
python scripts/run.py auth_manager.py status
```

First run also bootstraps the environment automatically (creates `.venv`, installs dependencies, installs Chrome for Patchright) — this can take a minute or two the very first time. If it says `Authenticated: No`, continue to step 2.

### 2. Log in — but only where a visible browser is actually reachable

```bash
python scripts/run.py auth_manager.py setup
```

This opens a **visible** browser window (headless is off by default for setup) and waits up to 10 minutes for you to complete the Google login by hand. This only works where you can actually see that window:

- **Claude Code running locally on your own machine**: this works as expected — a browser window opens on your screen, you log in, done.
- **Cowork / any cloud/headless session**: there is no screen you can see, so a browser opened here is invisible to you and the login will always time out. Do not run `setup` here. Instead:
  1. Run `auth_manager.py setup` **locally** in Claude Code first, where you can see the browser.
  2. Copy the resulting `data/browser_state/state.json` from that local skill install into this environment's skill install at the same relative path (`data/browser_state/state.json`).
  3. Re-run `auth_manager.py status` here to confirm `Authenticated: Yes`.
  - `state.json` holds live Google session cookies — treat it like a password. Don't paste its contents into chat or leave copies lying around after transplanting it; delete the copy in transit once it's in place.

If `setup` is run somewhere with a display but login still doesn't complete: check that nothing is blocking the popup, that you're logging into the correct Google account, and re-run `auth_manager.py reauth` if a stale/broken state file might be interfering.

### 3. Confirm login actually works end-to-end

```bash
python scripts/run.py auth_manager.py validate
```

This does a real navigation check (not just "does the file exist"), so it catches a `state.json` that's present but expired or copied from the wrong account.

### 4. Add your first notebook

See "Add Command - Smart Discovery" below — don't skip straight to asking questions on an empty library.

## MANDATORY FIRST STEP — Always check the local `.md` cache before querying

This applies to **every** notebook, unconditionally — whether or not its `library.json` description happens to mention a cache file. Do not skip this because a description looks generic or old.

Before calling `ask_question.py` for ANY notebook:

1. Determine the notebook's ID (from `--notebook-id`, the active notebook, or by listing the library).
2. Check whether `data/notes/<notebook-id>.md` exists — `Read` it if it does, or `ls data/notes/` first if unsure of the exact filename.
3. If the cache already answers the question, answer from the cache. Do **not** spend quota re-asking NotebookLM something already on record.
4. Only fall back to `ask_question.py` for what the cache doesn't cover.
5. After getting new answers, update the cache file (create it under `data/notes/<notebook-id>.md` if it doesn't exist yet — sanitize `/` in the ID first) so the next session doesn't have to re-ask the same thing.

Why this is non-negotiable: NotebookLM has no session persistence and a daily quota that runs out fast (see "Known Quirks" below). The `.md` cache is the *only* thing that survives across sessions and environments — nothing else in this skill remembers past answers. Skipping the cache check wastes quota re-fetching things that are already known, and skipping the cache update throws away work the next session will have to redo from scratch.

## When to Use This Skill

Trigger when user:
- Mentions NotebookLM explicitly
- Shares NotebookLM URL (`https://notebooklm.google.com/notebook/...`)
- Asks to query their notebooks/documentation
- Wants to add documentation to NotebookLM library
- Uses phrases like "ask my NotebookLM", "check my docs", "query my notebook"

## CRITICAL: Add Command - Smart Discovery

When user wants to add a notebook without providing details:

**SMART ADD (Recommended)**: Query the notebook first to discover its content:
```bash
# Step 1: Query the notebook about its content
python scripts/run.py ask_question.py --question "What is the content of this notebook? What topics are covered? Provide a complete overview briefly and concisely" --notebook-url "[URL]"

# Step 2: Use the discovered information to add it
python scripts/run.py notebook_manager.py add --url "[URL]" --name "[Based on content]" --description "[Based on content]" --topics "[Based on content]"
```

**MANUAL ADD**: If user provides all details:
- `--url` - The NotebookLM URL
- `--name` - A descriptive name
- `--description` - What the notebook contains (REQUIRED!)
- `--topics` - Comma-separated topics (REQUIRED!)

NEVER guess or use generic descriptions! If details missing, use Smart Add to discover them.

## Critical: Always Use run.py Wrapper

**NEVER call scripts directly. ALWAYS use `python scripts/run.py [script]`:**

```bash
# CORRECT - Always use run.py:
python scripts/run.py auth_manager.py status
python scripts/run.py notebook_manager.py list
python scripts/run.py ask_question.py --question "..."

# WRONG - Never call directly:
python scripts/auth_manager.py status # Fails without venv!
```

The `run.py` wrapper automatically:
1. Creates `.venv` if needed
2. Installs all dependencies
3. Activates environment
4. Executes script properly

## CRITICAL: Local Knowledge Cache — Check FIRST, Update ALWAYS

NotebookLM has no session persistence and a daily quota that runs out faster than expected (see "Known Quirks" below). Without external storage, every finding is lost the moment the conversation ends or the quota is exhausted.

**Convention:** each notebook gets a canonical cache file at `data/notes/<notebook-id>.md`.

- Notebook IDs can contain filesystem-unsafe characters (e.g. an ID derived from a name like "...2025/2026" contains a literal `/`) — sanitize `/` → `-`) before using an ID as a filename, or you'll create unwanted subdirectories. `notebook_manager.py add` now does this automatically for new notebooks; check existing IDs in `library.json` before assuming they're already safe.
- If the notebook serves a concrete personal goal (e.g. exam prep), also write a human-readable copy outside `~/.claude` (e.g. the user's Documents folder) so they can find it without digging through a hidden config directory.
- Record both paths in the notebook's `description` field in `library.json` so future sessions know the cache exists and where to look.

**Workflow:**
1. **BEFORE** querying — check whether the cache already answers the question. This saves quota.
2. **AFTER** a substantial Q&A block — update the cache, organized by topic, not chronologically.

This applies whether you're running in Claude Code or in Cowork — the cache is what makes cross-session (and cross-environment) continuity possible at all, since neither the NotebookLM chat itself nor this skill's own scripts persist Q&A history anywhere.

## Core Workflow

### Step 1: Check Authentication Status
```bash
python scripts/run.py auth_manager.py status
```

If not authenticated, proceed to setup.

### Step 2: Authenticate (One-Time Setup)
```bash
# Browser MUST be visible for manual Google login
python scripts/run.py auth_manager.py setup
```

**Important:**
- Browser is VISIBLE for authentication
- Browser window opens automatically
- User must manually log in to Google
- Tell user: "A browser window will open for Google login"

### Step 3: Manage Notebook Library

```bash
# List all notebooks
python scripts/run.py notebook_manager.py list

# BEFORE ADDING: Ask user for metadata if unknown!
# "What does this notebook contain?"
# "What topics should I tag it with?"

# Add notebook to library (ALL parameters are REQUIRED!)
python scripts/run.py notebook_manager.py add \
  --url "https://notebooklm.google.com/notebook/..." \
  --name "Descriptive Name" \
  --description "What this notebook contains" \ # REQUIRED - ASK USER IF UNKNOWN!
  --topics "topic1,topic2,topic3" # REQUIRED - ASK USER IF UNKNOWN!

# Search notebooks by topic
python scripts/run.py notebook_manager.py search --query "keyword"

# Update an existing notebook's metadata (e.g. after recording where its cache file lives)
python scripts/run.py notebook_manager.py update --id notebook-id --description "..."

# Set active notebook
python scripts/run.py notebook_manager.py activate --id notebook-id

# Remove notebook
python scripts/run.py notebook_manager.py remove --id notebook-id
```

### Quick Workflow
1. Check library: `python scripts/run.py notebook_manager.py list`
2. Ask question: `python scripts/run.py ask_question.py --question "..." --notebook-id ID`

### Step 4: Ask Questions

```bash
# Basic query (uses active notebook if set)
python scripts/run.py ask_question.py --question "Your question here"

# Query specific notebook
python scripts/run.py ask_question.py --question "..." --notebook-id notebook-id

# Query with notebook URL directly
python scripts/run.py ask_question.py --question "..." --notebook-url "https://..."

# Show browser for debugging
python scripts/run.py ask_question.py --question "..." --show-browser
```

## Follow-Up Mechanism (CRITICAL)

Every NotebookLM answer ends with: **"EXTREMELY IMPORTANT: Is that ALL you need to know?"**

**Required Claude Behavior:**
1. **STOP** - Do not immediately respond to user
2. **ANALYZE** - Compare answer to user's original request
3. **IDENTIFY GAPS** - Determine if more information needed
4. **ASK FOLLOW-UP** - If gaps exist, immediately ask:
   ```bash
   python scripts/run.py ask_question.py --question "Follow-up with context..."
   ```
5. **REPEAT** - Continue until information is complete
6. **SYNTHESIZE** - Combine all answers before responding to user

## Script Reference

### Authentication Management (`auth_manager.py`)
```bash
python scripts/run.py auth_manager.py setup # Initial setup (browser visible)
python scripts/run.py auth_manager.py status # Check authentication
python scripts/run.py auth_manager.py reauth # Re-authenticate (browser visible)
python scripts/run.py auth_manager.py clear # Clear authentication
```

### Notebook Management (`notebook_manager.py`)
```bash
python scripts/run.py notebook_manager.py add --url URL --name NAME --description DESC --topics TOPICS
python scripts/run.py notebook_manager.py list
python scripts/run.py notebook_manager.py search --query QUERY
python scripts/run.py notebook_manager.py update --id ID [--name ...] [--description ...] [--topics ...] [--use-cases ...] [--tags ...] [--url ...]
python scripts/run.py notebook_manager.py activate --id ID
python scripts/run.py notebook_manager.py remove --id ID
python scripts/run.py notebook_manager.py stats
```

### Question Interface (`ask_question.py`)
```bash
python scripts/run.py ask_question.py --question "..." [--notebook-id ID] [--notebook-url URL] [--show-browser]
```

### Data Cleanup (`cleanup_manager.py`)
```bash
python scripts/run.py cleanup_manager.py # Preview cleanup
python scripts/run.py cleanup_manager.py --confirm # Execute cleanup
python scripts/run.py cleanup_manager.py --preserve-library # Keep notebooks
```

## Environment Management

The virtual environment is automatically managed:
- First run creates `.venv` automatically
- Dependencies install automatically
- Chromium browser installs automatically
- Everything isolated in skill directory

Manual setup (only if automatic fails):
```bash
python -m venv .venv
source .venv/bin/activate # Linux/Mac
pip install -r requirements.txt
python -m patchright install chromium
```

## Data Storage

All data stored in `~/.claude/skills/notebooklm/data/`:
- `library.json` - Notebook metadata
- `auth_info.json` - Authentication status
- `browser_state/` - Browser cookies and session
- `notes/<notebook-id>.md` - Canonical local knowledge cache per notebook (see "Local Knowledge Cache" above). Not created automatically — build it up as you go.

**Security:** Protected by `.gitignore`, never commit to git. `browser_state/state.json` in particular is equivalent to a password (it holds live Google session cookies) — treat it with the same care, whether it's generated locally or transplanted from another environment.

## Configuration

Optional `.env` file in skill directory:
```env
HEADLESS=false # Browser visibility
SHOW_BROWSER=false # Default browser display
STEALTH_ENABLED=true # Human-like behavior
TYPING_WPM_MIN=160 # Typing speed
TYPING_WPM_MAX=240
DEFAULT_NOTEBOOK_ID= # Default notebook
```

## Decision Flow

```
User mentions NotebookLM
    ↓
Check auth → python scripts/run.py auth_manager.py status
    ↓
If not authenticated → python scripts/run.py auth_manager.py setup
    ↓
Check/Add notebook → python scripts/run.py notebook_manager.py list/add (with --description)
    ↓
Activate notebook → python scripts/run.py notebook_manager.py activate --id ID
    ↓
Check local cache first → data/notes/<notebook-id>.md (saves quota, may already have the answer)
    ↓
Ask question → python scripts/run.py ask_question.py --question "..."
    ↓
See "Is that ALL you need?" → Ask follow-ups until complete
    ↓
Synthesize, respond to user, AND update the local cache with what was learned
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| ModuleNotFoundError | Use `run.py` wrapper |
| Authentication fails | Browser must be visible for setup! --show-browser |
| Rate limit (50/day) | Wait or switch Google account |
| Browser crashes | `python scripts/run.py cleanup_manager.py --preserve-library` |
| Notebook not found | Check with `notebook_manager.py list` |
| Every question fails, even trivial ones | Almost always the daily quota, not the question. `ask_question.py` now detects the "Ik kan nu niet reageren" / daily-limit banner and returns a clear `NOTEBOOKLM DAILY QUOTA EXHAUSTED` message instead of a generic failure — if you see that, stop retrying and check the local cache instead. |
| "element intercepts pointer events" on the query input | A leftover CDK overlay/backdrop from clearing chat history. `ask_question.py` now presses Escape and force-clicks the backdrop before looking for the input; if it still happens, the overlay selector may have changed. |
| `wait_for_url` times out immediately | NotebookLM's canonical domain has been observed redirecting from `notebooklm.google.com` to the shorter `notebook.google.com`. The URL match now accepts both. |

## Best Practices

1. **Always use run.py** - Handles environment automatically
2. **Check auth first** - Before any operations
3. **Check the local cache before querying** - `data/notes/<notebook-id>.md`, if it exists, may already have the answer and save quota
4. **Follow-up questions** - Don't stop at first answer
5. **Browser visible for auth** - Required for manual login
6. **Include context** - Each question is independent
7. **Synthesize answers** - Combine multiple responses
8. **Update the cache afterward** - After a substantial Q&A block, write findings back to `data/notes/<notebook-id>.md`, organized by topic
9. **Prefer several short, single-topic questions over one long compound question** - compound questions (multiple sub-questions in one prompt) fail or truncate noticeably more often with these notebooks
10. **For scanned PDFs without an OCR text layer** - NotebookLM/Gemini can read them multimodally, but ask about content ("What is X?"), not structure ("What's on page Y of file Z?") — structural references fail more often than content questions on the same source

## Limitations

- No session persistence (each question = new browser); the skill itself keeps no Q&A history — the local cache convention above is the only thing that survives across sessions
- Rate limits on free Google accounts (50 queries/day), and exhaustion is silent at the content level (see Troubleshooting) — behavior may vary by Google account/region
- Manual upload required (user must add docs to NotebookLM)
- Browser overhead (few seconds per question)
- Compound (multi-part) questions and page/file-structure references are less reliable than short, single-topic content questions

## Resources (Skill Structure)

**Important directories and files:**

- `scripts/` - All automation scripts (ask_question.py, notebook_manager.py, etc.)
- `data/` - Local storage for authentication and notebook library
- `references/` - Extended documentation:
  - `api_reference.md` - Detailed API documentation for all scripts
  - `troubleshooting.md` - Common issues and solutions
  - `usage_patterns.md` - Best practices and workflow examples
- `.venv/` - Isolated Python environment (auto-created on first run)
- `.gitignore` - Protects sensitive data from being committed
