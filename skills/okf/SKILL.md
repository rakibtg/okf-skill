---
name: okf-skill
description: 'Author, validate, and maintain Open Knowledge Format (OKF) for AI-agent-readable knowledge (concepts, index.md, log.md, citations, cross-links). BEFORE starting any non-trivial coding task, check whether the repo has an OKF bundle (look for index.md/log.md or a concept.md-style file with `type` frontmatter) and read the relevant concept docs first — this is often faster and more accurate than re-deriving system context from source code alone. AFTER finishing a task that changes something a bundle documents (new feature, bug fix, schema/API/config change), add a log.md entry and update or create the relevant concept doc before considering the task done — skip this only for trivial changes or if the user declines. Also trigger on explicit requests: OKF, "knowledge bundle", "knowledge catalog", scaffolding a knowledge base, adding a concept doc, generating index.md, validating conformance, or converting existing docs/data catalogs into agent-readable markdown. Always use scripts/ (init_bundle.py, new_concept.py, gen_index.py, add_log_entry.py, validate.py) instead of hand-rolling frontmatter parsing or logging by hand.'
license: Apache-2.0
compatibility: general
metadata:
  spec: "OKF v0.1 (Google, GoogleCloudPlatform/knowledge-catalog)"
  spec-source: "https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md"
  category: documentation, knowledge-management
---

# OKF Skill — Open Knowledge Format authoring & validation

Open Knowledge Format (OKF) is Google's spec for representing knowledge — metadata,
context, curated insight — as a directory of plain markdown files with YAML
frontmatter. No schema registry, no bespoke tooling: `cat` it to read it, `git clone`
it to ship it. Full spec: `references/SPEC.md`. One-page cheat sheet:
`references/cheatsheet.md` — read that first for any quick question; open the full
spec only when you need an exact conformance rule or an edge case.

This skill turns that spec into a repeatable workflow: scaffold a bundle, add
concepts correctly, keep `index.md`/`log.md` current, and validate conformance —
using the scripts in `scripts/` instead of re-deriving frontmatter/link parsing by
hand each time.

## Core vocabulary (memorize, don't re-derive)

- **Bundle** — the whole directory tree. **Concept** — one `.md` file = one unit of
  knowledge. **Concept ID** — its path minus `.md` (`tables/users.md` → `tables/users`).
- **Reserved filenames** at any level: `index.md` (directory listing, §6) and
  `log.md` (change history, §7). Every other `.md` file is a concept.
- A concept has YAML frontmatter (`---`-delimited) with **exactly one required
  field: `type`** (a free-text string like `BigQuery Table`, `API Endpoint`,
  `Playbook`, `Metric` — not centrally registered). Recommended fields, in
  priority order: `title`, `description`, `resource` (canonical URI of the
  underlying asset, omit for abstract concepts), `tags` (list), `timestamp`
  (ISO 8601). Producers may add any other keys; consumers must not choke on them.
- Body is free-form markdown, but favor structure. Three conventional section
  headings: `# Schema` (columns/fields), `# Examples`, `# Citations` (§8, numbered
  external sources).
- Links are standard markdown links. `/tables/customers.md` = bundle-root-relative
  (recommended, stable under moves). `./other.md` = relative. Broken links are
  **tolerated by design** — they may just mean not-yet-written knowledge.
- `index.md` has **no frontmatter** (only exception: bundle-root `index.md` may
  carry `okf_version: "0.1"` — the only frontmatter permitted in an index file).
- `log.md` is a flat, newest-first, date-grouped list (`## YYYY-MM-DD` headings,
  ISO 8601 date form is mandatory) of prose entries, conventionally prefixed
  `**Update**`, `**Creation**`, `**Deprecation**`.
- **Conformance (§9)** — a bundle is conformant iff: (1) every non-reserved `.md`
  file has parseable YAML frontmatter, (2) that frontmatter has a non-empty
  `type`, (3) `index.md`/`log.md` follow §6/§7 where present. Everything else —
  missing optional fields, unknown `type` values, unknown extra keys, broken
  links, missing `index.md` — is soft guidance. Never treat those as errors.

## Workflow

Pick the branch that matches what the user asked for. All scripts are
dependency-free (Python 3 standard library only) — no `pip install` needed.

### 1. Starting a brand-new bundle

```bash
python3 scripts/init_bundle.py <path/to/bundle> [--okf-version 0.1]
```
Creates the root directory with a root `index.md` (frontmatter-only allowed
place: `okf_version`) and an empty `log.md`. Confirm the target path and, if the
user is documenting an existing system, ask what top-level groupings make sense
(e.g. `tables/`, `apis/`, `playbooks/`) before creating subdirectories — OKF's
directory layout is domain-defined, there is no fixed taxonomy (see §Non-goals).

### 2. Adding a concept

```bash
python3 scripts/new_concept.py <bundle_root> <concept/path/without-md> \
  --type "BigQuery Table" --title "Customer Orders" \
  --description "One row per completed customer order." \
  [--resource <uri>] [--tags sales,orders] [--no-timestamp]
```
This writes a correctly-frontmattered concept file from
`templates/concept.md.tmpl`, with `# Schema` / `# Examples` / `# Citations`
placeholder sections you should then fill in (or delete if not applicable —
none of them are required). Timestamp defaults to now (UTC, ISO 8601) unless
`--no-timestamp` is passed. If the user is transcribing an existing asset
(a real table, API, dashboard), prefer real column names/types and a real
`resource` URI over placeholders — do not invent schema details you were not
given; ask if unclear.

For a concept **not** bound to a physical resource (a playbook, a metric
definition, a process) just omit `--resource`.

### 3. Cross-linking

Always link with bundle-root-relative paths (`/tables/customers.md`) when
referencing another concept, per §5.1 — this survives the linking file being
moved. Only use relative links (`./sibling.md`) for concepts that will always
live side-by-side. Never invent a link target that doesn't exist to "complete"
a thought — write the plain-text claim instead, or create the missing concept.

### 4. Keeping `index.md` current

```bash
python3 scripts/gen_index.py <bundle_root> [--dir <subdir>] [--dry-run]
```
Regenerates `index.md` for the given directory (or every directory, recursively,
if `--dir` omitted) from the frontmatter of the concepts and subdirectories it
contains — pulling `title`/`description` per §6. Safe to re-run any time after
adding/removing concepts; it fully regenerates the auto section rather than
diffing, so if a human hand-edited extra prose into an `index.md`, warn the user
before overwriting (`--dry-run` first to show the diff).

### 5. Recording history

```bash
python3 scripts/add_log_entry.py <bundle_root> [--dir <subdir>] \
  --kind Update --text "Added freshness SLA to [orders](/tables/orders.md)."
```
Appends to the nearest `log.md` (creating it if absent) under today's
`## YYYY-MM-DD` heading, newest-first, `**Kind**: text` form per §7. Use this
after any meaningful add/edit/deprecate — don't let `log.md` go stale, but also
don't fabricate history for changes that didn't happen.

### 6. Validating a bundle

```bash
python3 scripts/validate.py <bundle_root> [--strict] [--check-links]
```
Checks the three hard conformance rules from §9 and reports them as PASS/FAIL.
By default also prints soft-guidance warnings (missing recommended fields,
non-ISO timestamps, unknown reserved-name misuse) — these are informational,
never a failure. `--strict` turns select warnings (missing `description`,
missing `title`) into failures for teams that want a stricter house style;
don't use `--strict` unless the user asks for it, since the spec explicitly
makes those optional. `--check-links` walks all markdown links and reports
which resolve within the bundle vs. which don't — broken links are reported,
never treated as conformance failures, per §5.3/§9.

Run this after any batch of changes, and always before telling the user a
bundle is "done" — surface the exact PASS/FAIL/warning lines, don't just say
"looks good."

### 7. Converting existing docs/catalogs into OKF

When migrating an existing data catalog, README set, or wiki into OKF: map
each existing entity to one concept file, preserve its real metadata into
frontmatter (don't invent `resource` URIs), and use `gen_index.py` /
`add_log_entry.py` at the end rather than hand-writing `index.md`/`log.md`.
Ask the user for the desired directory taxonomy up front if it isn't obvious
from the source material — OKF intentionally leaves this undefined (§Non-goals),
so guessing wrong means a rename/move pass later.

## Using this skill across coding agents

This directory follows the (Claude/OpenAI/OpenCode-shared) Agent Skills format:
a folder named after the skill containing `SKILL.md` with `name`+`description`
frontmatter, plus `scripts/`, `references/`, `templates/`. See `README.md` for
exact install paths per agent (Claude Code, OpenCode, Codex, generic). No
agent-specific code exists anywhere in this skill — the workflow above is the
same regardless of which agent is running it.

## Guardrails

- Never treat unknown `type` values, missing optional fields, or broken links
  as errors — the spec explicitly forbids rejecting a bundle for those (§9).
- Never put frontmatter in a non-root `index.md`, and never add anything to
  `log.md` except the entries themselves.
- Never fabricate `resource` URIs, schema columns, or citations. If the user
  hasn't given you the real facts, ask, or write the section as a placeholder
  and say so explicitly.
- Prefer running `validate.py` over eyeballing frontmatter — it implements the
  spec's exact conformance rules, including the root-`index.md` `okf_version`
  exception, which is easy to get subtly wrong by hand.
