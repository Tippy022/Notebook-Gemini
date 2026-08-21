<div align="center">

# Gemini Notebook â NotebookLM Skill for Claude Code / Cowork

**Let Claude chat directly with Google NotebookLM for source-grounded, citation-backed answers from Gemini**

</div>

---

## What this is

A Claude skill that queries your Google NotebookLM notebooks through browser automation and returns Gemini's source-grounded answers directly into your conversation â no manual copy-paste between NotebookLM and your editor.

Originally built for local Claude Code by [PleasePrompto](https://github.com/PleasePrompto/notebooklm-skill); this fork adds fixes and a mandatory local-cache workflow so it also works from **Claude in Cowork** (cloud sessions), where NotebookLM's lack of session persistence and daily quota limits are otherwise a much bigger problem.

## Installation

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/sydovenema-lgtm/Notebook-Gemini.git gemini-notebook
```

Then open Claude Code (or Cowork) and say: **"What are my skills?"**

## First-time setup

See the **"ð First-Time Setup"** section at the top of `SKILL.md` for the full walkthrough â checking auth status, logging in (visible-browser requirement and how to handle that in a headless/cloud session), and validating the login actually works before adding notebooks.

## The local knowledge cache (important)

NotebookLM has no session persistence and a daily quota that runs out fast. This skill maintains a local Markdown cache per notebook at `data/notes/<notebook-id>.md`, and `SKILL.md` makes checking/updating it a **mandatory first step** before ever querying NotebookLM â not optional advice. This is what makes the skill usable across sessions and environments instead of re-asking the same questions (and burning quota) every time.

## What's different from the original

- Fixed a domain-redirect bug (`notebooklm.google.com` â `notebook.google.com`)
- Fixed a leftover-overlay bug that blocked the query input after clearing chat history
- Daily-quota exhaustion is now detected explicitly (NotebookLM answers "I can't respond right now" to every question when quota is out, indistinguishable from a real failure otherwise)
- Added an `update` subcommand to `notebook_manager.py` for editing existing notebook metadata
- Notebook IDs are sanitized (`/` â `-`) so names containing a slash don't create unwanted subfolders
- Mandatory local-cache-check workflow, enforced both in `SKILL.md` and with a runtime reminder printed by `ask_question.py` itself

## Security note

`data/` (your notebook library, cache, and browser auth state) is deliberately excluded from this repo via `.gitignore` and was never committed. `data/browser_state/state.json` in particular holds live Google session cookies; treat it like a password, never commit it, never share it outside a trusted channel.

## License

See `LICENSE`.
