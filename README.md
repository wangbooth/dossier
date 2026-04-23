# dossier

[English](README.md) · [中文](README.zh-CN.md)

> A portable [Skills](https://agentskills.io/)-format skill — works with [Claude Code](https://claude.com/claude-code), Codex, ChatGPT, and other Skills-compatible agents. It lets your AI agent delegate citation-heavy research to [Google NotebookLM](https://notebooklm.google.com/), while keeping the corpus out of the agent's context window.

## Why

When you ask Claude to "research topic X", three things normally go wrong:

1. **Tokens burn fast.** Pasting 20 PDFs into the conversation blows through context and the bill.
2. **Answers hallucinate.** Plausible-sounding but unsourced claims get mixed in with real facts.
3. **Work doesn't compound.** Next week, same topic, same sources — you start from scratch.

`dossier` splits the job:

| Role | Who | What they do |
|------|-----|--------------|
| 📚 Corpus + retrieval | **NotebookLM** | Holds your sources forever. Answers questions with `[1][2]` citations that map back to source text, never extrapolating. |
| 🧠 Orchestration | **AI agent** (Claude Code / Codex / ChatGPT / …) | Curates sources, decides when to delegate to the dossier, writes code, runs experiments, synthesizes. |
| 🎯 Decisions | **You** | Confirm the shortlist of sources. Judge the final conclusion. |

The result is a reusable, citation-backed corpus per research topic, where repeat questions cost cents instead of dollars and every factual claim traces back to source text.

## When to use

- Literature reviews (multiple papers, cross-paper comparisons)
- Due diligence on companies, funds, sectors
- Regulatory / legal / compliance review across document sets
- Competitive intelligence
- Long-lived personal knowledge base (Obsidian / Readwise / meeting archives)
- Any task where **citations are required** and **the same corpus will be queried repeatedly**

## When NOT to use

- One-off web lookups → use Claude directly
- Codebase exploration → your agent's native file tools (Read / Grep / etc.) are stronger
- Real-time data → NotebookLM is static
- Tiny corpora (< 5k tokens) → just paste into the prompt
- When latency matters more than citation quality → NotebookLM is ~3× slower than direct chat

## How it works — the three routes

Every new dossier starts with one question: how are the sources going to get in?

| Route | Best for | Who picks sources |
|-------|----------|-------------------|
| **A · NotebookLM auto-research** | Quick overview, exploration | NotebookLM's own research agent (via web search) |
| **B · Claude-curated** ⭐ | Authoritative sources needed, long-term reuse | Claude proposes a shortlist, **you confirm**, Claude imports |
| **C · User-specified** | You already know what you trust | You give the list, Claude just imports |

The skill's working rule: **never pick silently — present the tradeoff, let the user choose.**

## Install

### 1. Get the underlying tool

`dossier` builds on top of [teng-lin/notebooklm-py](https://github.com/teng-lin/notebooklm-py) (Python NotebookLM CLI). The skill itself will prompt you to run these on first use, but you can do it now:

```bash
pip install "notebooklm-py[browser]"
python3 -m playwright install chromium
python3 -m notebooklm login      # one-time Google OAuth via browser
python3 -m notebooklm skill install  # installs the low-level `notebooklm` skill
```

> **PATH note (macOS):** `pip install` often puts the `notebooklm` binary in `~/Library/Python/<ver>/bin/` which isn't on `$PATH`. The skill handles this automatically by falling back to `python3 -m notebooklm` — you don't have to fix PATH.

### 2. Install this skill

`dossier` is in the [Skills format](https://agentskills.io/specification) — a single markdown file with YAML frontmatter that any Skills-compatible agent can load. Install path depends on your agent:

| Agent | Install |
|-------|---------|
| **Claude Code** | `git clone https://github.com/wangbooth/dossier ~/.claude/skills/dossier` |
| **Codex CLI** | `git clone https://github.com/wangbooth/dossier ~/.codex/skills/dossier` |
| **Multi-agent (Open Skills)** | `npx skills add wangbooth/dossier` |
| **ChatGPT / others** | Check your agent's docs for its skill directory, then `git clone` the repo into it |

If you already have the repo cloned somewhere, symlink instead of re-cloning:

```bash
ln -sfn /path/to/dossier <your-agent-skill-dir>/dossier
```

> **Codex `skill-installer` note:** Codex's built-in installer currently has trouble with "root-as-skill" repositories like this one (`SKILL.md` sits at the repo root, not inside a subdirectory), because its `git sparse-checkout` path doesn't handle `--path .` gracefully. The plain `git clone` command above bypasses the installer and works reliably — just point it at `~/.codex/skills/dossier` and restart Codex.

### 3. Jina Reader key — no setup needed upfront

Some sites (PubMed most notably) aggressively block NotebookLM's backend scraper. When that happens the skill falls back to [Jina Reader](https://jina.ai), an LLM-friendly fetch proxy.

**You don't need to do anything now.** The first time a fetch actually needs Jina, Claude will ask you to paste a key (free from <https://jina.ai>, no credit card) and save it to `~/.config/dossier/config.json` with `chmod 600`. From then on it's automatic — no `export`, no shell config, no key typed in a terminal.

Advanced users who want to preconfigure can either set `$JINA_API_KEY` (env var always wins) or write the config file directly:

```json
// ~/.config/dossier/config.json
{ "jina_api_key": "jina_xxxxxxxx" }
```

## Quick start

In your agent session (Claude Code, Codex, etc.), just describe your research:

> "I want to research whether creatine is safe for the kidneys long-term."

Claude will detect the intent, load the skill, and walk you through the three-route choice. Typical flow for Route B (recommended):

1. Claude scans your existing notebooks — if a relevant dossier exists, it reuses it.
2. Otherwise Claude proposes a shortlist of authoritative sources (you say go / edit).
3. Claude imports sources, verifies they landed correctly (catches anti-bot failures), uses Jina Reader where needed.
4. You ask questions. Answers come back with `[1][2]` citations mapped to specific text spans in specific sources.
5. Next week, same topic, same dossier — skip steps 1–3.

## Example

See [examples/creatine-research.md](examples/creatine-research.md) for a real 8-source dossier built in ~15 minutes, with three live Q&A demonstrating the depth of cited answers.

## Under the hood

The skill is pure prompt engineering — no code. It's a markdown file in the [Skills format](https://agentskills.io/specification) that teaches any compatible agent:

- When to trigger (description field)
- How to run the preflight (install check, auth check, PATH fallback)
- Which of the three routes to propose and how to branch
- The nine working rules (shortlist confirmation, citation resolution via `--json`, "outside source" layering, conversation clearing between subtopics, etc.)
- How to structure the output (`## Dossier says / ## What I did / ## Conclusion / ## Not covered`)

All commands route through the `notebooklm` CLI (from notebooklm-py).

## Why notebooklm-py and not another client

We compared three active NotebookLM clients before picking one. Full decision record in [research/client-comparison.md](research/client-comparison.md). TL;DR:

- **teng-lin/notebooklm-py** (11k★, Python) — chosen. Only client with a real `create notebook` command, best agent-integration story, PyPI-stable releases.
- icebear0828/notebooklm-client (181★, TypeScript) — missing `create`, npm package lags the main branch.
- PleasePrompto/notebooklm-mcp (2k★, TypeScript MCP) — chat-only, can't manage sources or dossiers; 4 months stale.

## Known limitations

- **Unofficial Google API.** NotebookLM has no public API. `notebooklm-py` uses reverse-engineered endpoints that may break when Google changes their backend.
- **Rate limits are real.** PubMed specifically aggressively anti-bots cloud scrapers. The skill uses Jina Reader as a fallback.
- **Session token decays.** Re-run `notebooklm login` if auth errors start appearing (typically weeks-to-months).
- **Source cap.** Free NotebookLM tier = 50 sources per notebook, Pro = 300. Plan dossier scope.
- **Generation commands are slow.** `notebooklm generate audio`/`video` take 5–10 min each.

## Contributing

Bug reports and PRs welcome. The skill is one markdown file ([SKILL.md](SKILL.md)) — improvements typically come from running real dossiers and finding cases where the rules miss.

## License

MIT — see [LICENSE](LICENSE).
