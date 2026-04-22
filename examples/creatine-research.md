# Example: Creatine research dossier

A real session recorded 2026-04-22. Used to dogfood this skill during development. Conversation was in Chinese; quoted outputs preserved as-is.

## What we built

**Dossier**: `dossier-creatine-research` (id `39220ea5-…`)
**Sources**: 8 (1 position-paper + 1 independent encyclopedia + 6 systematic-review/meta-analyses)
**Build time**: ~15 minutes wall clock (most of it spent on anti-bot workarounds)
**Cost to build**: negligible — curation was Claude orchestrating, indexing was NotebookLM's free tier

## Step 1 — Route selection

> **User:** 研究肌酸对人体的作用
>
> **Claude** (triggered dossier skill): You don't have an existing creatine dossier. Here are the three routes — which should I take?
>
> - **A.** Let NotebookLM auto-research the topic (5 min, zero token cost, opaque sources)
> - **B.** I curate authoritative sources (PubMed meta-analyses, ISSN position stand, Examine.com), you approve the shortlist (20 min, persistent)
> - **C.** You give me the source list explicitly
>
> I recommend **B** for this topic — creatine has high marketing noise and plenty of peer-reviewed meta-analyses worth getting right.
>
> **User:** B

## Step 2 — Shortlist

Claude proposed 8 sources spanning the core decision axes:

| # | Source | Axis covered |
|---|--------|--------------|
| 1 | Kreider et al. 2017, ISSN position stand (JISSN) | Mechanism, dosing, safety, eligible populations |
| 2 | Examine.com creatine page | Independent, question-organized encyclopedia |
| 3 | Zhang 2025 (PMID 41328071) | Strength gains meta-analysis |
| 4 | Sharifian 2025 (PMID 41062952) | Older adults + resistance training |
| 5 | Eckert 2025 (PMID 41189312) | Creatine for depression symptoms |
| 6 | Naeini 2025 (PMID 41199218) | Kidney function safety |
| 7 | Ashtary-Larky 2025 (PMID 41433021) | Dose-response |
| 8 | Kazeminasab 2025 (PMID 40944139) | Upper/lower body strength & power |

> **User:** 全要

## Step 3 — Import, and the realities of anti-bot

Direct `notebooklm source add <URL>` worked for:
- ✓ ISSN position stand (Springer/BMC)
- ✓ Examine.com
- ✓ Naeini 2025 (PubMed — succeeded, likely one-shot request before rate-limiting kicked in)
- ✓ Zhang 2025 (PubMed — succeeded)

**Parallel batch of 4 other PubMed URLs → all 4 returned `"Checking your browser - reCAPTCHA"`.** NotebookLM's backend scraper hit PubMed's Cloudflare challenge.

Retry serially with 3-second sleep between calls → still failed (3 of 4 reCAPTCHA).

**Workaround that actually worked**: fetch via [Jina Reader](https://r.jina.ai) (LLM-friendly proxy), save as Markdown, ingest as file:

```bash
curl -sL -H "Authorization: Bearer $JINA_API_KEY" \
  "https://r.jina.ai/https://pubmed.ncbi.nlm.nih.gov/41189312/" -o /tmp/eckert.md
notebooklm source add /tmp/eckert.md
```

For Kazeminasab 2025, even Jina hit reCAPTCHA on the PubMed URL. Fix: use the **DOI link** which redirects to the publisher (MDPI Nutrients — open access):

```bash
curl -sL -H "Authorization: Bearer $JINA_API_KEY" \
  "https://r.jina.ai/https://doi.org/10.3390/nu17172748" -o /tmp/kazeminasab.md
# result: 236 KB of real content — full text, not just abstract
```

Gotcha: `--title` flag is silently ignored for file-source imports. Follow up each file import with `notebooklm source rename <id> "<real title>"`.

## Step 4 — Final dossier

```
$ notebooklm source list
Total: 8 sources, all ready
1. Ashtary-Larky 2025 — Creatine dose-response in resistance training (abstract)
2. Creatine benefits, dosage, and side effects                  # Examine.com
3. Eckert 2025 — Creatine for depression symptoms (meta-analysis, abstract)
4. Effect of creatine supplementation on kidney function...     # Naeini 2025
5. Effects of creatine supplementation on muscle strength...    # Zhang 2025
6. Impact of creatine supplementation and exercise training in older adults  # Sharifian 2025
7. International Society of Sports Nutrition position stand...  # Kreider 2017
8. Kazeminasab 2025 — Creatine on upper/lower body strength and power (Nutrients full text)
```

## Step 5 — Sample Q&A

### Q1: 肾安全

> 长期服用肌酸对肾功能有影响吗？健康人群和已有肾病人群应该如何区分对待？

**Answer (abridged):** 不会对健康人群造成实质性损害 [1-3]。研究显示即使 5 年、30g/天也无肾功能障碍 [2,3]。血清肌酐会有轻微上升但源于肌酸代谢转化而非肾损伤 [1,4]。GFR 未受影响 [1,4]。已有肾病人群建议咨询主治医生 [3]。一项动物模型研究提示对肾囊肿疾病大鼠高剂量肌酸可能加剧恶化 [3]。

**Source contribution**: Naeini 2025（肾 meta-analysis）+ ISSN + Examine 三者综合。

### Q2: 女性 vs 男性

> 女性服用肌酸和男性效果是否有差异？

**Answer (abridged):** 男性在力量/爆发力提升上通常更显著 [1]。一项 69 RCT 的 meta-analysis 显示下肢力量、垂直弹跳、Wingate 无氧峰值功率上男性显著提升，女性幅度小且统计上不显著 [2-5]。上肢力量男女都显著提升但男性绝对增长略大 [6,7]。机制：雌激素下调肌酸转运蛋白表达 [8]；女性肌肉肌酸基线可能更高从而削弱补充空间 [9]；男性初始肌肉量、II 型快肌纤维比例、磷酸肌酸储备更契合肌酸作用机制 [8,9]。需要注意：针对纯女性群体的专项研究数量远少于男性。

**Source contribution**: 260 citation spans from 3 sources: Kazeminasab 2025 (211×), ISSN (39×), Examine (10×). The Kazeminasab full text (via DOI/Jina) contributed **5× more** than ISSN or Examine combined — strong evidence that full-text import beats abstract-only.

### Q3: 肌酸 × 咖啡因互作

> 肌酸和咖啡因同时服用会相互影响吗？

**Answer (abridged):**

**Part 1 — from the dossier:** 资料中没有直接给出明确结论。文献记录了多项咖啡因×肌酸联合试验，评估无氧爆发力、肌力、耐力、身体成分、疲劳感、认知 [1,2]，包含冲击期咖啡因摄入的专项研究 [3]。

**Part 2 — explicitly flagged as outside the sources:**
- 早年研究提示持续高剂量咖啡因可能干扰肌酸部分肌肉放松益处
- 近期应用研究认为合理剂量下同时服用有效安全（市售氮泵常同时含有两者）
- 肠胃不适风险存在
- 若有不适建议错开时间（咖啡因训前、肌酸训后）

**Why this Q&A is important**: NotebookLM self-layered "sourced vs. speculative" — the skill's rule #9 says to preserve this split in the final output, never to collapse them.

## Observations after the fact

1. **The anti-bot wall is the biggest operational hurdle.** PubMed URLs blocked even serially; Jina Reader + DOI publisher redirect unlocked everything.
2. **Full text >> abstract** for depth. Kazeminasab 2025 full-text (from MDPI via DOI/Jina) produced 5–10× more citations than abstract-only imports.
3. **`--json` on `ask` is non-negotiable.** Without it, `[1][2]` in the answer is unresolvable. With it, every citation maps to `source_id` + the exact `cited_text` span.
4. **NotebookLM's "outside source" layering is pure gold.** Q3 is the clearest example — it tells you exactly what the corpus covers and what it's speculating. Don't flatten this in the Claude-facing output.
5. **Cost**: The entire dossier build + 4 Q&A ran at negligible Claude token cost. All the heavy indexing lives in NotebookLM's free tier. Repeat questions later cost basically nothing.
