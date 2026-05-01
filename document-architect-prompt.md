# Doc Architect — Beast Pass Documentation Analysis & Reusable Prompt

This file captures (1) an analysis of how the Beast Pass repository organizes its documentation, (2) the transferable patterns behind that organization, and (3) a stack-agnostic prompt that can generate a similar documentation system for any other project.

---

## 1. Document responsibility analysis

The Beast Pass docs system has a clear **layered information architecture** with each file owning a specific topic and audience.

### Layer A — Entry points (repo root)

| File            | Topic owned                                                                                                                                                | Audience           | Distinctive trait                                                      |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ | ---------------------------------------------------------------------- |
| `README.md`     | Onboarding, prerequisites, getting started, CI/CD, releases, **entity lifecycle checklists** (add/remove/modify), monorepo management, directory structure | Human contributors | Long-form, table-heavy, has explicit checklists with `[ ]` boxes       |
| `SUMMARY.md`    | High-signal **agent orientation**: stack, projects, codegen pipeline, commands, entry points, gotchas                                                      | AI agents          | "Decision aids" + "Where to dive deeper" sections; pointers, not prose |
| `AGENTS.md`     | **Agent rules & coding guidelines** (delegation rules, backend patterns, "don't do X" list, diagnostics workflow)                                          | AI agents          | Persistent-context anchor at top; rules-first                          |
| `CLAUDE.md`     | Just `@AGENTS.md`                                                                                                                                          | Claude Code        | Aliases for tooling that auto-loads `CLAUDE.md`                        |
| `REFERENCES.md` | Flat index of `@`-prefixed paths to every key doc                                                                                                          | Tooling            | Pure reference list                                                    |
| `CHANGELOG.md`  | Release notes                                                                                                                                              | Everyone           | Conventional                                                           |

### Layer B — Cross-cutting concerns (`docs/`)

| File                            | Topic owned                                                                                          | Distinctive trait                                                                                                                        |
| ------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `docs/authentication.md`        | **Who** the caller is — Clerk, JWT, M2M, JIT provisioning, env vars                                  | Cross-refs authorization.md; "Failure modes & quick triage" section linking to DIAG-IDs                                                  |
| `docs/authorization.md`         | **What** caller can do — PBAC policies, scopes (self/assigned/org), bypass controls, frontend gating | Mirror structure to authentication.md; "Generated vs custom boundaries"                                                                  |
| `docs/domain.md`                | Non-coding **business rules** (entity meaning, real-world processes, workflows)                      | Explicitly says "NOT for code guidelines" — pure domain knowledge                                                                        |
| `docs/diagnostics.md`           | Troubleshooting playbook                                                                             | Searchable IDs (`[DIAG-XXX]`) + hashtags (`#auth #401`); standardized entry template (Symptoms / Diagnosis / Resolution / Related Files) |
| `docs/integrations/<vendor>.md` | Per third-party integration deep dive                                                                | ASCII data-flow + sync triggers + data mapping tables                                                                                    |
| `docs/specs/<feature>.md`       | Feature specs with requirements + sequence diagrams                                                  | PlantUML diagrams, `[ ]` requirement checkboxes                                                                                          |

### Layer C — Per-app docs

| File                    | Topic owned                                                                            |
| ----------------------- | -------------------------------------------------------------------------------------- |
| `apps/<app>/README.md`  | Build, run, test, project structure, deployment URLs                                   |
| `apps/<app>/SUMMARY.md` | Agent-focused high-signal orientation (only on large/complex apps like portal)         |
| `apps/<app>/AGENTS.md`  | App-specific agent rules (only where needed; e.g., `infrastructure/aws-cdk/AGENTS.md`) |
| `apps/services/API.md`  | Endpoint reference (special — separates API surface from app README)                   |

### Layer D — Per-package docs

| File                       | Topic owned                                                                         |
| -------------------------- | ----------------------------------------------------------------------------------- |
| `packages/<pkg>/README.md` | Scope (in-scope / **out-of-scope**), public API, conventions, regeneration commands |

### Layer E — Tools, data, and generation

| File                         | Topic owned                                                                     |
| ---------------------------- | ------------------------------------------------------------------------------- |
| `tools/generators/README.md` | Codegen pipeline with ASCII data-flow diagram, generator-by-generator reference |
| `data/*.yaml`, `data/*.dbml` | Source-of-truth config (not docs but referenced everywhere as canonical)        |

### Layer F — Infrastructure, operations, and narrow references

| File                               | Topic owned                                                                                   |
| ---------------------------------- | --------------------------------------------------------------------------------------------- |
| `infrastructure/aws-cdk/README.md` | Deployment, environments, secrets, AWS architecture, database tunneling, troubleshooting      |
| `infrastructure/aws-cdk/AGENTS.md` | Infra-specific agent orientation: stack responsibilities, extension rules, safety constraints |
| `assets/**/README.md`              | Narrow asset placement references; useful only when a workflow needs local static assets      |

### Quality observations from the current repo

| Category                     | Files                                                                                                                         | Why it matters for another project                                                                                                                                              |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Central routing docs         | `AGENTS.md`, `SUMMARY.md`, `README.md`, `docs/diagnostics.md`, `tools/generators/README.md`, `apps/portal/SUMMARY.md`         | These docs should be treated as the navigation spine. Other docs should link to them rather than duplicate their content.                                                       |
| High-value concept docs      | `docs/authentication.md`, `docs/authorization.md`, `docs/domain.md`, `docs/integrations/ticketing-platform.md`                | They each own one mental model and finish with key implementation files. This is the strongest reusable pattern.                                                                |
| Potentially stale docs       | `apps/services/API.md`, `REFERENCES.md`, some runtime/env-var claims split between `README.md`, `SUMMARY.md`, and portal docs | A doc system needs a repair/audit mode, not only a generation mode. Prompts must verify concrete claims against the repo before preserving them.                                |
| Generated/scaffold-like docs | `CHANGELOG.md`, `packages/client-utils/README.md`, `packages/shared/README.md`, generated-looking infra `AGENTS.md`           | Generated or scaffold docs should be identified before rewriting; either preserve them as generated artifacts or replace low-signal scaffolds with real package ownership docs. |
| Duplicate narrow docs        | `assets/credentials/README.md`, `packages/backend-utils/src/domains/accreditation/modules/credentials/assets/README.md`       | Duplicate tiny references are acceptable only when they serve different physical locations. Otherwise consolidate and cross-link.                                               |

---

## 2. Transferable approaches (the "how", not the content)

These are the patterns that carry to any project, regardless of stack:

1. **Two-track docs at root** — one for humans (`README.md`), one for AI agents (`SUMMARY.md` + `AGENTS.md`). Different density, different goals, same source of truth.
2. **Audience declared up front** — every doc opens with "who this is for" and "what it is NOT for" (see `domain.md`).
3. **Ownership by topic, not by file size** — split when a topic deserves its own page (auth ≠ authz; diagnostics ≠ domain).
4. **Cross-references at the top** — "for X, see Y" links pair complementary docs.
5. **"Key files" footer** — every conceptual doc ends with a list of canonical implementation files (with paths and sometimes line numbers).
6. **Tables over prose** for: tech stack rows, env→URL maps, branches→envs, comparison matrices, file→purpose maps.
7. **Lifecycle checklists with `[ ]`** for repetitive multi-step procedures (add entity, remove entity, modify entity).
8. **Searchable IDs + tags** in diagnostics (`[DIAG-XXX]` + `#tag`) so LLMs and humans can grep deterministically.
9. **Generated vs custom boundary** explicitly called out in any doc that touches codegen.
10. **Source-of-truth declarations** — name the canonical file ("schema.dbml is the primary source of truth").
11. **Failure-modes section** in every flow doc, pointing to diagnostic IDs.
12. **High-signal framing for AI** — "use this instead of reading the whole repo," concrete pointers, no narrative filler.
13. **Diagrams where flow matters** — ASCII for architecture, PlantUML for sequences.
14. **Quickstart command blocks** in fenced code with comments explaining each line.
15. **Out-of-scope sections** in shared-unit docs — prevents scope creep and tells readers where NOT to look.
16. **Persistent-context anchor** at top of agent-facing docs ("never summarize, never omit").
17. **Quality status is explicit** — identify central, stale, generated/scaffolded, duplicated, and low-signal docs instead of pretending every doc has equal authority.
18. **Root docs route; deep docs decide** — root docs should answer where to go next, while cross-cutting docs should answer how a subsystem actually works.
19. **Review/repair mode is first-class** — a mature repo often has existing docs with drift; the process must audit and minimally repair instead of regenerating everything.
20. **Target-native paths over copied paths** — preserve the layered responsibilities, but use the target repository's real structure. Do not introduce folder names from Beast Pass or from another stack unless they already exist in the target repo.

---

## 3. Prompt explanation

The prompt below is **idempotent** — it works for both first-time documentation generation and subsequent re-runs that maintain existing docs. It auto-detects which mode applies and behaves accordingly:

**Mode A — Generate (first run)**: when the target repo has no documentation system yet (or only a bare `README.md`). The AI surveys the project, plans the file layout, and writes the full doc system from scratch.

**Mode B — Review & repair (re-run)**: when the doc system already exists. The AI audits every existing doc against the architecture, flags inconsistencies (drifted paths, broken cross-refs, stale commands, topic overlap between files) and over-detail (narrative essays in agent docs, duplicated content, content that belongs in code rather than docs), proposes targeted edits, and applies them only after approval. It never rewrites a doc that is already conformant.

Both modes share the same three-step rhythm:

1. **Discovery & plan** — survey the repo, produce a written plan of what to create/update/skip, and stop for approval.
2. **Execution** — apply the plan: write new files in Mode A, edit minimally and preserve voice in Mode B.
3. **Self-audit** — verify the writing patterns are present, the cross-references resolve, and no Beast Pass terminology leaked in.

The prompt is stack-agnostic: it should work for any language, framework, or repository shape. It explicitly tells the AI not to copy Beast Pass terminology, folders, commands, or tooling — only the documentation responsibilities and writing patterns.

It enforces the AI-agent-first patterns (SUMMARY.md, AGENTS.md, key-files footers, DIAG-IDs) because those are the most distinctive and most often missing in standard doc setups. Beast Pass-specific paths appear only in the analysis above; the reusable prompt below uses placeholders and target-native paths.

---

## 4. The prompt (copy this block to use in another project)

```text
# Documentation Architect — Project Doc System Generator & Maintainer

You are a documentation architect. Analyze a software project and either create a documentation system or repair the existing one. Preserve the documentation responsibilities and writing patterns below, but use the target project's real language, framework, tooling, and directory structure. Do not copy Beast Pass folder names, framework names, commands, or terminology unless they actually exist in the target project.

## Mode detection (run this first, before anything else)

Inspect the repo for the doc system's signature files:

- `SUMMARY.md` at repo root
- `AGENTS.md` at repo root
- `REFERENCES.md` at repo root
- `docs/` directory with at least one of `authentication.md`, `authorization.md`, `domain.md`, `diagnostics.md`

Decide:

- **Mode A — Generate**: zero or one of those exist (e.g., only a stock `README.md`). Build the doc system from scratch.
- **Mode B — Review & repair**: two or more exist. Audit and fix the existing system; do not regenerate.

State which mode you selected and why in one sentence before continuing. If the signal is mixed (e.g., `SUMMARY.md` exists but is empty or unrelated), default to Mode B and treat the empty/unrelated files as items to repair.

## Discovery first (both modes)

Before writing or editing anything, perform a discovery pass on the target project:

1. **Repository shape** — top-level directories, monorepo vs single project, languages, frameworks, runtimes, package/build tools.
2. **Runnable units** — identify each application, service, job, CLI, worker, frontend, backend, or other executable surface using the repo's own structure.
3. **Shared units** — identify reusable code, internal SDKs, generated clients, common utilities, or equivalent shared areas using the repo's own structure.
4. **Cross-cutting systems** — authentication, authorization, persistence, messaging, integrations, code generation, testing, deployment, observability.
5. **Source-of-truth files** — schemas, migrations, API specs, IDL files, generated-file inputs, deployment config, policy config, or equivalent canonical files.
6. **Existing docs** — inventory every README/MD/RST/ADOC file and mark its current topic, audience, and quality status: `central`, `ok`, `stale-risk`, `duplicate`, `generated`, `scaffold`, or `placeholder`.

Use file listings, dependency manifests, CI files, infrastructure/deployment files, and code search for SDK imports, env-var names, route definitions, commands, and generated-file markers. **Do not start writing or editing docs until discovery is complete.**

## Required output structure

Use this file structure as an information architecture, not as a literal folder template. Keep root and `docs/` names where possible. For runnable units, shared units, tools, and infrastructure, use the target repo's existing directories. Do not create a folder only because it appears in this prompt.

### Layer A — Repo root (entry points)

- **README.md** — Human onboarding: what the project is, prerequisites, setup, run/test/build, CI/deployment, release/versioning, directory map, key workflows, useful links. Include `[ ]` checklists for repeated procedures with more than three steps. Only include verified commands and URLs; otherwise mark them as `<confirm>` and list them as follow-up questions.
- **SUMMARY.md** — Agent-focused orientation: stack, project map, quick rules, source-of-truth files, entry points, commands, gotchas, and where to dive deeper. It must be scannable in under 60 seconds. Pointers, not prose.
- **AGENTS.md** — Mandatory agent rules: first docs to read, safe command usage, generated-file rules, testing expectations, area-specific coding constraints, app-running safety, and diagnostics workflow. Open with a persistent-context anchor warning.
- **CLAUDE.md** — One line: `@AGENTS.md`. Skip or adapt only if the target tooling uses a different agent alias mechanism.
- **REFERENCES.md** — Flat list of every doc file as `@<path>`. No prose.
- **CHANGELOG.md** — Only if not already present; standard Keep-a-Changelog format.

### Layer B — Cross-cutting concerns (`docs/`)

For each cross-cutting concern you discovered, produce a dedicated file. Always include at minimum:

- **docs/authentication.md** — Who the caller is: identity providers, sessions/tokens, local vs deployed behavior, provisioning, required configuration, failure modes, key files.
- **docs/authorization.md** — What the caller can do: roles, permissions, scopes, enforcement layers, frontend/backend gates, bypasses, generated/custom boundaries, key files. Cross-link to authentication.md at top.
- **docs/domain.md** — Business rules / domain knowledge. State explicitly at top: "THIS DOCUMENT IS NOT FOR CODE GUIDELINES." Cover entities' real-world meaning, lifecycle, key constraints, relationships. Use bold for entity names, tables for comparisons, "Key rules:" bullet lists.
- **docs/diagnostics.md** — Troubleshooting playbook. Mandatory format: instructions for "How to Search" (by ID, by tag, by error message), table of contents (ID | Title | Tags), then entries each with `[DIAG-XXX]` heading, `**Tags:**`, `**Symptoms:**`, `**Diagnosis Steps:**`, `**Resolution:**` or `**Common Causes:**` table, `**Related Files:**`. Tag format: `#lowercase-hyphenated`. End with a "How to Add New Entries" stub.
- **docs/integrations/<integration>.md** — One file per important third-party or external system integration: purpose, data flow, triggers, configuration, mappings, failure handling, key files.
- **docs/specs/<feature>.md** — One file per significant feature/system spec: requirements, flow diagrams where helpful, state model if any, access model, related files.

Skip a file only if the concern genuinely doesn't exist in the project.

### Layer C — Runnable-unit docs

For each runnable unit, in its existing folder:

- **README.md** — Overview, deployment URLs table, getting started (build/run/test/lint), project structure (tree block), entry points, configuration, environment variables.
- **SUMMARY.md** — Only for large/complex runnable units. Agent-focused mirror of root SUMMARY.md but scoped to that unit.
- **AGENTS.md** — Only when the app has rules distinct from root AGENTS.md (typical for infrastructure/IaC).
- **API.md** — Only for API-exposing runnable units. Endpoint reference.

### Layer D — Shared-unit docs

For each shared unit, in its existing folder:

- **README.md** — Sections: scope (in-scope bullet list), **out-of-scope** bullet list, public API or conventions, regeneration/build commands if generated, transaction/lifecycle patterns, external clients if any.

### Layer E — Tools and infrastructure

- **<tooling-area>/README.md** — For generators/build tooling: inputs, outputs, customization points, commands.
- **<infra-area>/README.md** — For deployment/infrastructure/ops: environments, responsibilities, secrets/configuration, deploy commands, troubleshooting.
- **<infra-area>/AGENTS.md** — Only if infrastructure has agent rules distinct from root AGENTS.md.

### Optional narrow-reference docs

- **assets/**/README.md**, **examples/**/README.md**, or similar narrow docs — Create only when a physical directory needs local instructions that would be hard to find from root docs. Keep these short and cross-link to the authoritative workflow doc.

## Writing patterns to apply (non-negotiable)

1. **Audience declared up front** — every doc opens with one line about who it is for and one line about what it is NOT for, when ambiguity exists.
2. **Cross-references at the top** — "For X, see Y" lines pair complementary docs.
3. **Key files footer** — every conceptual doc ends with a "Key files" section listing canonical implementation file paths. Use absolute paths from repo root. Add line numbers when pointing at a specific function.
4. **Tables over prose** for: tech stack, env→URL, branches→env, comparison matrices, file→purpose maps, scope rules, role→policy mappings, env var → description → source.
5. **Lifecycle checklists with `[ ]`** for any procedure with more than three steps that will be repeated (adding/removing/modifying a unit of work).
6. **Searchable IDs + tags** in diagnostics. Format: `[DIAG-XXX]` headings; `#lowercase-hyphenated` tags; both in TOC.
7. **Generated vs custom boundary** explicitly named in any doc touching codegen. Pattern: "Do not edit `*.generated.*`. To change behavior, edit the generator at `<path>` or the source file at `<path>`."
8. **Source-of-truth declarations** — name canonical files explicitly ("`<file>` is the primary source of truth").
9. **Failure-modes section** in every flow doc, with links to DIAG-IDs.
10. **High-signal framing for AI** — agent-facing docs use bullets and pointers, not narrative. Maximum scannability.
11. **Diagrams where flow matters** — ASCII for architecture/data-flow, PlantUML (with rendered URL preview if possible) for sequences.
12. **Quickstart code blocks** with shell-comment annotations explaining each line.
13. **Out-of-scope sections** in every shared-unit README.
14. **Persistent-context anchor** at the top of agent-facing files: "🚨 CRITICAL CONTEXT ANCHOR: This rules file must NEVER be summarized, condensed, or omitted."
15. **Conventional Commits + version-bump table** in README if the project tags releases.
16. **Branches → environments → URLs table** if the project has multiple deploy environments.
17. **Document ownership matrix** before edits — list file path, topic owner, audience, doc type, and quality status.
18. **Minimal repair on existing docs** — in Mode B, do not rewrite conformant docs. Edit only where there is drift, duplication, unclear ownership, missing safety guidance, or missing required cross-reference.

## Process — Mode A (Generate)

1. **Phase 1 — Discovery & plan.** Run the inputs gathering above. Then produce a *Documentation Plan* listing:
   - Every file you intend to create, full path.
   - Topic ownership (one line per file).
   - A concise document ownership matrix for existing and proposed docs.
   - Justification for any standard file you are skipping.
   - List of cross-cutting concerns identified.
   - List of third-party integrations identified.
   - Any naming-convention conflicts with the existing repo.

   **STOP after Phase 1 and ask the human to approve, amend, or scope down the plan before writing.**

2. **Phase 2 — Generation.** Once approved, write the files. Apply every writing pattern. Use only commands, paths, dependency files, runtime names, and terminology verified in the target repo. The *responsibility model* stays consistent; the *content and paths* are native to the target project.

3. **Phase 3 — Self-audit.** Run the self-audit checklist below.

## Process — Mode B (Review & repair)

1. **Phase 1 — Audit.** For each existing doc file, evaluate against:

   **Topic ownership** — does this file own a single, clear topic? Flag if it overlaps another file (e.g., authentication content leaking into authorization.md), if it sprawls into unrelated topics, or if a topic from the required structure has no home.

   **Drift** — verify every concrete claim against the current code: file paths, function names, env vars, commands, branch names, deploy URLs, package versions. Anything that doesn't match the repo today is drift.

   **Over-detail** (red flags):
   - Narrative paragraphs explaining *what* the code does — that belongs in code/comments, not docs. Keep *why* and *how to use*, drop the rest.
   - Step-by-step internal mechanics that change with refactors. Replace with a one-line summary plus a link to the canonical source file.
   - Duplicated content already covered in another doc. Replace with a cross-reference.
   - Long inline code samples that aren't load-bearing examples. Trim or remove.
   - Auto-generated content that should live in generated artifacts (API endpoint dumps, full env var lists from config) — point at the source instead.
   - Any section longer than ~30 lines without a heading break — it has lost focus.

   **Inconsistency** (red flags):
   - Same concept named differently across files (e.g., "module" in one doc, "package" in another for the same thing).
   - Conflicting commands or version requirements between README.md and SUMMARY.md.
   - Cross-references that resolve to missing files or sections.
   - Tables of contents out of sync with actual content.
   - REFERENCES.md missing files that exist, or pointing at files that don't.

   **Quality status** — classify each doc as `central`, `ok`, `stale-risk`, `duplicate`, `generated`, `scaffold`, or `placeholder`. Generated release files and intentionally tiny local README files may be left alone if they are serving their purpose.

   **Pattern conformance** — check against every item in the "Writing patterns to apply" section above. Note which patterns are missing per file.

   **Completeness** — list any required-output-structure files that are missing entirely.

2. **Phase 2 — Repair plan.** Produce a *Repair Plan* organized by file. For each file, list:
   - **Status:** `OK` (no changes), `Edit` (specific changes), `Restructure` (topic ownership wrong, content needs to move), `Delete` (redundant, content moved elsewhere), `Create` (missing).
   - **Findings:** bullet list of issues found, each tagged `[drift]`, `[over-detail]`, `[inconsistency]`, `[missing-pattern]`, or `[completeness]`.
   - **Quality status:** one of `central`, `ok`, `stale-risk`, `duplicate`, `generated`, `scaffold`, or `placeholder`.
   - **Proposed action:** specific, minimal edits. For deletions or content moves, name the destination.

   **STOP after Phase 2 and ask the human to approve, amend, or scope down the repair plan before editing.** Default to *fewer*, *smaller* edits — only restructure when topic ownership is genuinely wrong.

3. **Phase 3 — Apply.** Execute the approved plan. Edit minimally:
   - Preserve the original author's voice and section order where possible.
   - Do not reformat working tables or rewrite working sentences for stylistic preference.
   - When trimming over-detail, keep the *why* and the pointer to the canonical file; drop the *what*.
   - When fixing drift, change only the inaccurate line, not the surrounding paragraph.
   - When restructuring, move content with cut-and-paste fidelity before rewording.

4. **Phase 4 — Self-audit.** Run the self-audit checklist below.

## Self-audit checklist (both modes)

After writing or editing, verify:

- Every conceptual doc has a "Key files" footer (where applicable).
- Every flow doc cross-references its diagnostic entries.
- Every codegen-touching doc names the generated-vs-custom boundary.
- SUMMARY.md is scannable in under 60 seconds (target: ≤ 250 lines).
- AGENTS.md opens with the persistent-context anchor.
- No file copies Beast Pass terminology by accident — all examples reflect the target project.
- REFERENCES.md is in sync with files actually present on disk.
- Every cross-reference resolves to a real file/section.
- No file exceeds the project-appropriate length for its audience (agent docs short; human onboarding can be longer).
- Each file's topic ownership is clear, and duplicated content is either removed or intentionally justified.
- Required docs were skipped only with an explicit reason tied to the target repo's actual architecture.
- (Mode B only) Every finding from the Repair Plan has been addressed or explicitly deferred with a reason.

Report any gaps explicitly; do not silently leave them.

## Anti-goals (do not do these)

- Do not invent features, integrations, or env vars that don't exist in the codebase. Verify by grep before writing.
- Do not copy code samples that won't run. If you can't verify, mark as `<example>` placeholder and flag it.
- Do not write narrative essays. Bullets, tables, code blocks, lists.
- Do not duplicate content across files. Cross-reference instead.
- Do not omit "out-of-scope" or "what this is NOT for" — those are load-bearing.
- Do not skip the discovery phase. Writing without grounding produces hallucinated docs.
- (Mode B) Do not regenerate files that are already conformant. "It's not how I would have written it" is not a reason to edit.
- (Mode B) Do not bulk-rewrite to apply stylistic preferences. The bar for editing existing prose is a concrete finding (drift, over-detail, inconsistency, missing pattern, completeness gap).
- (Mode B) Do not silently delete content. If content is removed, name where it went or why it was redundant.

## When you finish

End with a one-paragraph summary of what was created vs. updated vs. skipped (Mode A) or what was edited vs. restructured vs. deleted vs. left as-is (Mode B), plus a list of follow-up questions the human should answer to fill in any unverifiable gaps (e.g., "I couldn't determine the production URL — please confirm").
```
