# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code skill** (`deep-research`) that provides an autonomous, multi-phase research pipeline. It generates citation-backed research reports with source credibility scoring, triangulation, and verification. The skill is invoked via Claude Code when users request comprehensive analysis (e.g., "deep research on X").

Installed globally at `~/.claude/skills/deep-research/`.

## Key Files

- **SKILL.md** - The skill definition (Claude Code reads this when the skill triggers). Contains the decision tree, workflow, phase execution rules, and anti-hallucination protocol. This is the most important file.
- **reference/methodology.md** - Detailed 8-phase methodology, loaded on-demand during execution
- **templates/report_template.md** - Report structure template, used in Phase 8 (Package)
- **scripts/** - Python validation and utility scripts (stdlib-only, no external deps)

## Validation & Testing

```bash
# Validate a generated report (8 automated checks)
python scripts/validate_report.py --report path/to/report.md

# Verify citations aren't hallucinated
python scripts/verify_citations.py --report path/to/report.md

# Verify HTML report matches markdown source
python scripts/verify_html.py --html path/to/report.html --md path/to/report.md

# Convert markdown report to HTML
python scripts/md_to_html.py path/to/report.md
```

Test fixtures live in `tests/fixtures/` with `valid_report.md` and `invalid_report.md` examples.

There is no unit test framework -- validation is done via the Python scripts above.

## Architecture

The research pipeline has 4 modes with increasing depth:

| Mode | Phases | Use Case |
|------|--------|----------|
| Quick | 1,3,8 | Exploration, broad overview |
| Standard | 1-5,8 | Most research (default) |
| Deep | 1-8 | Important decisions |
| UltraDeep | 1-8+ iterations | Critical analysis, 50K+ word reports |

**Phase flow**: Scope -> Plan -> Retrieve (parallel) -> Triangulate -> Outline Refinement -> Synthesize -> Critique -> Refine -> Package

**Progressive File Assembly**: UltraDeep reports use auto-continuation via recursive agent spawning to exceed the 32K output token limit, chaining multiple agents that each write sections to disk.

## Scripts Overview

All scripts are stdlib-only Python (no pip install needed):

- `research_engine.py` - Core orchestration with ResearchState/ResearchPhase/ResearchMode enums and JSON state persistence
- `validate_report.py` - 8-check quality gate (citations, word count, prose ratio, etc.)
- `citation_manager.py` - Citation tracking, bibliography generation
- `source_evaluator.py` - Credibility scoring (0-100 scale)
- `verify_citations.py` - Anti-hallucination citation verification
- `md_to_html.py` - Markdown to McKinsey-style HTML conversion
- `verify_html.py` - HTML/MD consistency check

## Key Design Principles

1. **Autonomy-first**: The skill proceeds without asking for clarification unless the query is incomprehensible or contradictory. Default to standard mode and infer context.
2. **Anti-hallucination**: Every factual claim must cite a source immediately `[N]`. Distinguish FACTS (sourced) from SYNTHESIS (analysis). Never fabricate citations.
3. **Parallel execution**: Phase 3 (Retrieve) must launch 5-10+ concurrent searches in a single message, plus 3-5 parallel agents for deep dives.
4. **Prompt caching**: SKILL.md has a static block at the top (>1024 tokens) optimized for caching. Dynamic content goes after. Don't restructure this.
5. **Progressive disclosure**: Reference external files (methodology.md, report_template.md) on-demand rather than inlining everything into SKILL.md.
6. **Prose-first**: Reports must be >=80% prose, not bullet lists. Anti-fatigue enforcement prevents quality degradation in long reports.

## When Modifying SKILL.md

- Keep the static context block at the top intact (it's optimized for prompt caching)
- Critical instructions go at the START or END of sections, not buried in the middle ("loss in the middle" avoidance)
- The decision tree at the top executes first -- changes there affect all research runs
- Phase references point to `./reference/methodology.md#phase-N` -- keep anchors in sync
