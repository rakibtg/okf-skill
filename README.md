# okf-skill

This is an Agent Skill that teaches AI coding agents, including Claude Code, OpenCode, and Codex, how to read and write Open Knowledge Format (OKF) bundles. OKF is a spec from Google. It stores knowledge as plain markdown files with YAML frontmatter, so an AI agent can actually make use of it. It works with any agent that supports the [Agent Skills](https://github.com/vercel-labs/skills) format, and the full spec is here: [OKF spec](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md).

This project is a practical implementation of the pattern Andrej Karpathy described in his ["LLM Wiki" gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): instead of an agent re-deriving context from raw files every time, it builds and maintains a structured, interlinked knowledge base that keeps compounding. OKF is the standard for that structure, and this skill is the tooling that makes an agent fluent in it out of the box.

## The problem this solves

Most project docs are written for humans. So an agent either rereads whole files every time it needs context, or skips the docs entirely and re-derives everything from source code. Neither scales well.

OKF fixes this by giving knowledge a shape an agent can navigate step by step. It gets an `index.md` it can skim before deciding what to open, `type`-tagged frontmatter it can filter on, and a `log.md` it can check for recent history, instead of one big wall of prose it has to read in full every time.

This skill is the missing piece. Without it, an agent has to guess at OKF's rules, like which frontmatter goes where or how `index.md` and `log.md` should be structured, or you have to explain the spec to it every session. With it, the agent already knows the rules, and it has scripts to handle the mechanical parts correctly instead of hand-rolling YAML parsing.

Worth noting: this only helps for whatever you actually document in OKF. It is not a magic indexer for your whole codebase. It is a standard, and now a workflow, for the knowledge layer you choose to maintain alongside your code: data catalogs, API docs, playbooks, system context.

## Install

<details>
<summary><strong>Using skills.sh (recommended)</strong></summary>

Install with [`npx skills`](https://github.com/vercel-labs/skills):

```bash
npx skills add rakibtg/okf-skill --skill okf-skill
```

This detects your agent (Claude Code, OpenCode, Codex, and others) and copies the skill into the right directory automatically.

</details>

<details>
<summary><strong>Manual installation</strong></summary>

Copy the folder into whichever path your agent reads from. Most agents check project-local first, then global.

```bash
# Claude Code (project)
mkdir -p .claude/skills && cp -r okf-skill .claude/skills/okf-skill

# Claude Code (global, all projects)
mkdir -p ~/.claude/skills && cp -r okf-skill ~/.claude/skills/okf-skill

# OpenCode (project)
mkdir -p .opencode/skills && cp -r okf-skill .opencode/skills/okf-skill
# OpenCode also auto-discovers .claude/skills and .agents/skills, so the
# Claude Code install above already works here too.

# Codex, or any other Agent-Skills-compatible agent
mkdir -p .agents/skills && cp -r okf-skill .agents/skills/okf-skill
```

No skills support at all? Just point the agent at the folder and say "read SKILL.md and follow it." Everything it needs is self-contained: instructions, scripts, spec, templates.

</details>

## What's inside

```
  SKILL.md              the skill itself
  scripts/               dependency-free Python 3 (stdlib only)
    _okf_common.py         shared frontmatter/link parsing helpers
    init_bundle.py         scaffold a new bundle (root index.md + log.md)
    new_concept.py         create a correctly-frontmattered concept doc
    gen_index.py           regenerate index.md from concept frontmatter
    add_log_entry.py       append a dated entry to the nearest log.md
    validate.py            check §9 conformance + soft warnings + links
  references/
    SPEC.md                 vendored full OKF v0.1 spec from Google
    cheatsheet.md           one-page quick reference
  templates/
    concept.md.tmpl
    index.md.tmpl
    log.md.tmpl
```

## Quick start

There are two ways to use this, and both end up in the same place.

**Option A: just ask your agent, in plain English.**

Once installed, you don't need to remember any of the commands above. Your agent reads `SKILL.md`, picks the right script, and runs it for you. For example:

```
"Set up an OKF knowledge bundle in docs/knowledge for our BigQuery tables"
"Add a concept for the orders table, it's a BigQuery table, one row per order"
"Regenerate the index and check if the bundle is conformant"
```

The scripts exist so the agent (or you, in CI, or by hand) never has to guess at the correct frontmatter shape or hand-write YAML. Plain Python 3 stdlib, every one of them. No `pip install`, no network calls, no version drift between your machine, your teammate's, and CI.

## Examples that trigger this skill

An agent with this skill installed will reach for it on requests like:

- "Document this API endpoint as a knowledge bundle"
- "Add a concept for the users table"
- "Scaffold a knowledge base for our data catalog"
- "Generate an index.md for the tables directory"
- "Check if our docs are OKF conformant"
- "Log this change in the knowledge base"
- "Convert our existing wiki pages into an agent-readable format"
- "Keep this documentation agent-friendly"
- "Build a repo of docs for RAG retrieval"

It won't fire on unrelated requests (like "fix this rate limit bug") unless that work touches something already documented in a bundle, in which case the agent may offer to log the change afterward.

**Option B: run the scripts yourself.**

```bash
python3 scripts/init_bundle.py my-knowledge

python3 scripts/new_concept.py my-knowledge tables/orders \
  --type "BigQuery Table" --title "Orders" \
  --description "One row per completed customer order." \
  --resource "https://console.cloud.google.com/bigquery?p=acme&d=sales&t=orders" \
  --tags sales,orders

python3 scripts/gen_index.py my-knowledge

python3 scripts/add_log_entry.py my-knowledge --kind Creation \
  --text "Added the orders table."

python3 scripts/validate.py my-knowledge --check-links
```

## Deterministic scripts, fewer tokens, automatically

Every time an agent adds a concept, regenerates an index, or checks conformance, it doesn't need to figure out how from scratch. It runs one of the scripts with a fixed set of flags and gets the same correct result every time.

That determinism matters for token usage in two concrete ways, and the skill handles both of them for you:

- **No planning overhead.** The agent doesn't spend tokens reasoning about how to format frontmatter, how to shape an index entry, or whether a log heading is dated correctly. It calls a script and moves on.
- **No re-verification overhead.** Because `new_concept.py`, `gen_index.py`, and `validate.py` always produce the same correct shape, the agent doesn't need to re-read its own output to check it worked, or re-derive the rules to double-check itself. It can trust the result and continue.

You never have to ask for this. It's just how the scripts behave, so the savings happen automatically, every time the skill is used.

## Why the workflow stays reliable as your bundle grows

- **Progressive disclosure.** The agent only keeps `SKILL.md`'s roughly 150 lines in context to know the workflow exists. It opens `references/SPEC.md` or `cheatsheet.md` only when it needs an exact rule, so context stays cheap even across a long session adding dozens of concepts.
- **Deterministic conformance checking.** `validate.py` implements the spec's three hard rules (§9) as code, so whether a bundle is conformant is never left to the model eyeballing YAML, which is exactly the kind of check that quietly drifts wrong across a long conversation if it's done by hand each time.
- **No drift.** `gen_index.py` rebuilds `index.md` from the concepts themselves instead of hand-edited prose, so it can never fall out of sync with what it's listing. It also gets the one genuinely fiddly part of the spec right automatically: `okf_version` is the only frontmatter a generated `index.md` should ever carry, and only at the bundle root.

## License

Skill code and docs (`SKILL.md`, `scripts/`, `templates/`, `README.md`, `references/cheatsheet.md`): Apache-2.0.

`references/SPEC.md` is a vendored copy of Google's OKF v0.1 spec from [`GoogleCloudPlatform/knowledge-catalog`](https://github.com/GoogleCloudPlatform/knowledge-catalog). Check that repo directly for its license terms before redistributing that file on its own.
