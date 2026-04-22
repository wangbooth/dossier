# dossier

> A [Claude Code](https://claude.com/claude-code) skill that lets Claude delegate citation-heavy research to [Google NotebookLM](https://notebooklm.google.com/), while keeping the corpus out of Claude's context window.

## Why

When you ask Claude to "research topic X", three things normally go wrong:

1. **Tokens burn fast.** Pasting 20 PDFs into the conversation blows through context and the bill.
2. **Answers hallucinate.** Plausible-sounding but unsourced claims get mixed in with real facts.
3. **Work doesn't compound.** Next week, same topic, same sources — you start from scratch.

`dossier` splits the job:

| Role | Who | What they do |
|------|-----|--------------|
| 📚 Corpus + retrieval | **NotebookLM** | Holds your sources forever. Answers questions with `[1][2]` citations that map back to source text, never extrapolating. |
| 🧠 Orchestration | **Claude Code** | Curates sources, decides when to delegate to the dossier, writes code, runs experiments, synthesizes. |
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
- Codebase exploration → Claude Code's native tools are stronger
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

**Claude Code (personal install):**

```bash
git clone https://github.com/<you>/dossier ~/.claude/skills/dossier
```

Or, if you already cloned elsewhere, symlink it:

```bash
ln -sfn /path/to/dossier ~/.claude/skills/dossier
```

### 3. Optional: Jina Reader fallback key

Some sites (PubMed most notably) aggressively block NotebookLM's backend scraper. The skill falls back to [Jina Reader](https://jina.ai) to fetch them through a LLM-friendly proxy. Free tier works for light use; for heavy use get a key from <https://jina.ai> and export it:

```bash
export JINA_API_KEY="jina_xxxxxxxx"
```

## Quick start

In a Claude Code session, just describe your research:

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

The skill is pure prompt engineering — no code. It's a markdown file that teaches Claude Code:

- When to trigger (description field)
- How to run the preflight (install check, auth check, PATH fallback)
- Which of the three routes to propose and how to branch
- The seven working rules (shortlist confirmation, citation preservation, "outside source" layering, etc.)
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
