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
# Prepare project documentation

Analyze the service or application repository I name at the end and generate or verify its documentation set. Do NOT write application code or modify source files. Only create, update, or review documentation files.

## Stack-agnostic principle

This prompt is intentionally stack-agnostic. Do not assume the programming language, framework, runtime, package manager, build tool, test runner, linter, deployment platform, message transport, database, or repository shape. Discover them from the target repository and document what is actually used.

When this prompt mentions API-shape concepts (constructors, methods, overloads, parameters, fields, props, schema fields, return shapes, etc.), treat them as a representative list of API-shape detail across stacks. Apply the rule to whatever the equivalent concept is in the target stack: Go exports, Rust traits/impls, Python signatures, Elixir module functions, TypeScript types, React component props, etc.

## Document set

The doc set falls into three categories. Use these labels everywhere in this prompt.

- **Always-required.** Must exist in every repo: `README.md`, `AGENTS.md`, `CLAUDE.md`.
- **Required when the concern exists.** Must exist when the repo has the concern; omit and update the README documentation index when it does not:
  - `docs/architecture.md` when the repo has a runtime boundary, internal modules, or deployment ownership. (Skip for trivial single-file utilities.)
  - `docs/authentication.md` when the repo has authentication or authorization logic.
- **Always-emitted with documented absence.** Must exist in every repo, even when the concern is empty. State the absence and cite what was inspected:
  - `docs/domain.md`, `docs/interactions.md`, `docs/testing.md`, `docs/gotchas.md`.

The full ordered set, used in every file list and procedure in this prompt: `README.md`, `AGENTS.md`, `CLAUDE.md`, `docs/architecture.md`, `docs/domain.md`, `docs/authentication.md`, `docs/interactions.md`, `docs/testing.md`, `docs/gotchas.md`.

## Mandatory source-of-truth scan

This scan runs once, before any mode is announced. The Fresh, Verification, and Review procedures all begin from its results.

Inspect the repository's actual sources of truth:

1. Existing docs: `README.md`, `AGENTS.md`, `CLAUDE.md`, and any existing `docs/*.md`
2. Language and tooling indicators: dependency manifests, lockfiles, workspace files, build files, task-runner files, tool-version files, formatter/linter configs, generated-code configs, and editor configs
3. Runtime entrypoints: executable targets, server/app bootstrap files, CLI entrypoints, container definitions, process manager files, local-development scripts, and documented start commands
4. Commands: scripts/tasks in manifests, task runners, make/task files, CI workflows, container commands, test-runner configs, and existing docs
5. Testing: unit, integration, and end-to-end test projects/directories/configs; shared fakes, fixtures, helpers, test containers, browser drivers, seeded data, or other test infrastructure
6. Linting and static checks: linter, formatter, type checker, analyzer, security scanner, style checker, and CI quality gates
7. Dependencies: first-party/shared packages, third-party packages, private registries, external services, local services, credentials, environment variables, databases, queues, storage, browsers, emulators, containers, and system tools required to run or test the repo
8. Architecture and interactions: internal modules/packages/projects, persistence, API or UI boundaries, background jobs, scheduled work, event publishers/consumers, webhooks, runtime HTTP calls, command-line integrations, and other external systems
9. Business workflows and domain behavior: lifecycle/state machines, approval flows, setup/order-of-operations, capacity or quota rules, ownership/scoping rules, generated/system-managed data, idempotency/retry/concurrency behavior, and stable domain invariants documented in code, tests, schemas, migrations, or existing docs.
10. Local app lifecycle: ports, startup order, background/foreground process behavior, required local services, login/auth setup, browser/API inspection requirements, cleanup/stop commands, and any commands or scripts that start multiple runtimes together.
11. Deployment and infrastructure: infrastructure-as-code, manifests, deployment configs, environment-specific settings, resource naming patterns, and runtime hosting configuration

Do not write a section until you have inspected the files that establish the facts in that section. If a fact cannot be determined, say that and name what you inspected. Never invent a command, dependency, or framework.

## Mode detection

After the source-of-truth scan, choose one mode:

- **Fresh run** - fewer than two of the always-required docs (`README.md`, `AGENTS.md`, `CLAUDE.md`) exist, or the user explicitly asks to regenerate, rewrite, or replace the docs. Generate the full applicable set per the per-doc rules below.
- **Verification run** - at least two of the always-required docs exist and the user asks to generate, fix, update, apply, or regenerate. Do NOT rewrite correct docs from scratch. Audit each existing doc against this prompt and fix only the violations. Preserve correct content verbatim.
- **Review run** - the user asks to review, audit, check, or score the docs without asking to fix, apply, or regenerate. Read-only: inspect the repo and doc set, produce a structured findings report, and make no file changes.

### Mode disambiguation

- When the user says "analyze and generate a documentation set" or similar generation-flavoured wording and the target docs already exist, treat this as a Verification run. "Generate" means "produce the correct end state", not "rewrite from scratch".
- Treat the run as Fresh only when the user explicitly asks to regenerate, rewrite, or replace the docs, or when the always-required docs are essentially absent per the threshold above.
- When the user asks to "fix", "apply", "update", or "regenerate", pick Verification or Fresh per the rules above and proceed.
- When the user asks to "review", "audit", "check", or "score" without "and fix" or "and apply", stay in Review run.

### Mode announcement

Your first user-facing message, before any edits, must (1) state which mode you picked and why, (2) list the applicable target docs that currently exist, and (3) for a Fresh or Verification run, name the files you plan to create or change and the category of change. Do not begin edits until this summary is in the conversation. If the user disagrees with the mode, switch.

## Fresh run procedure

1. Confirm the source-of-truth scan is complete.
2. Announce Fresh mode and list which always-required, required-when-applicable, and always-emitted-with-absence files you will create.
3. Generate each file per the per-doc rules below, in the canonical order.
4. Skip required-when-applicable files whose concern is absent, and update the `README.md` documentation index to reflect the absence.
5. For always-emitted-with-absence files where the concern is empty, write the file with a single short section that states the absence and cites what was inspected.
6. Run the Postflight checklist across the full generated set.

## Verification run procedure

1. List which applicable target files exist, using the canonical order: `README.md`, `AGENTS.md`, `CLAUDE.md`, `docs/architecture.md`, `docs/domain.md`, `docs/authentication.md` if the repo has authentication or authorization logic, `docs/interactions.md`, `docs/testing.md`, and `docs/gotchas.md`.
2. For each existing file, check it against the Core principles and its own per-doc rules. Flag:
   - Content that belongs in a different file.
   - Duplicated facts across files; keep the fact in the owning file and replace other copies with links.
   - Banned drift-prone specifics: file counts, line counts, API-shape element counts, exact dependency versions, full route/event/mock/entity inventories, valid-value lists that belong next to code or validation.
   - File enumerations where a directory link plus purpose would be enough.
   - API-detail tables (property, method, constructor, parameter, schema-field, component-prop, or stack-equivalent) that belong in source-adjacent API docs, typed schemas, comments, validators, generated docs, or examples.
   - Missing required sections for that file.
   - For `AGENTS.md`, a missing, altered, condensed, or non-leading CRITICAL CONTEXT ANCHOR block.
   - Stale facts that contradict the current code, configs, scripts, CI, or deployment files.
   - Claims, commands, file paths, or directory links that are not grounded in the repo or no longer exist.
   - Sections, lines, tables, diagrams, or whole files that describe concerns the repo no longer has, that are over-detailed for what this prompt requires, or that were written speculatively before this prompt existed. Remove them; do not soften or rephrase to preserve them. Per Core principle 13, prefer cutting over keeping.
3. Apply fixes in place. Do not regenerate a correct file. If a file is fundamentally misshaped, note that and rewrite only that file. Removal of stale or over-detailed content counts as a fix; record what was removed in your mode-announcement summary.
4. If a required applicable file is missing, generate it per the Fresh-run rules and the per-doc rules below. If a required-when-applicable file exists but the repo does not contain that concern, remove it and update indexes or cross-links. If an always-emitted-with-absence file exists but the concern is empty, leave the file in place and update it to state the absence with citations.
5. If a finding cannot be verified within this run, leave the existing content unchanged and list the item in a closing "Unverified during this run" section in your reply, with the reason and what you inspected.
6. Run the Postflight checklist across the full applicable set.

## Review run procedure

1. The Mandatory source-of-truth scan has already been performed. Do not edit anything.
2. List which applicable target files exist, using the canonical order.
3. For each existing file, run the Preflight checklist, Core principles, and per-doc rules against its contents. For concrete claims, verify them against the repo: search for symbols named in docs, inspect dependency manifests and lockfiles, check scripts and CI workflows for commands, confirm directory and file paths resolve, confirm cross-repo references are prose rather than broken relative links, and verify universal claims by inspecting every relevant matching file. If `AGENTS.md` exists, verify that its first block is the exact CRITICAL CONTEXT ANCHOR text from the AGENTS.md section below, with no summarising, rewording, or preceding prose.
4. Produce a single markdown findings report:

   ```markdown
   # Documentation review - <service name>

   ## Verdict
   <PASS | PASS-with-minors | NEEDS-WORK> - <one-sentence summary>

   ## File coverage
   <bullet list of applicable files: present / missing / unexpected>

   ## Findings

   ### <path/to/file.md>
   - **Blocker** - line <N>: <quoted offending text>
     - Rule: <Core principle or file-section name>
     - Why: <one sentence grounded in the repo files you inspected>
     - Suggested fix: <one sentence>
   - **Major** - ...
   - **Minor** - ...

   ## Missing or extra files
   <missing applicable docs or unexpected conditional docs>

   ## Unverifiable items
   <anything you could not check, with the reason>
   ```

5. Severity definitions:
   - **Blocker** - violates a Core principle, misses or alters the required AGENTS.md CRITICAL CONTEXT ANCHOR block, documents a command that is not supported by repo evidence, asserts an unverified universal claim, contains a broken cross-repo or ignored-file link, or states a load-bearing fact contradicted by current code/config. "Load-bearing" means the fact appears in a doc index, command list, dependency setup section, environment variable table, or any other place a reader will rely on to act.
   - **Major** - content belongs in a different file, required section is missing, drift-prone specifics are included, enumerations should be directory links, API-detail tables belong in source-adjacent docs, a stale path or resource name appears in non-load-bearing prose, or a practical command section is incomplete.
   - **Minor** - wording, repetition within a file, inconsistent capitalisation, or non-load-bearing style issues.
6. The Review run does not modify files, commit, or create missing docs. If a finding cannot be fully inspected in one pass, list it under Unverifiable items instead of guessing.
7. If the user follows up with "apply" or "fix these", switch to Verification run and address findings in place.

## Core principles

1. No duplication across files. Every fact lives in exactly one `.md`. Other documents link to it. If two files would repeat the same table, one is wrong.
2. Navigation pointers may repeat across files when they only direct the reader to the owning document and do not restate the underlying facts.
3. API-shape details belong next to the source of truth, not in markdown overview docs. Do not put property tables, method signatures, constructor signatures, field lists, parameter tables, component prop tables, schema-field inventories, return-shape tables, or stack-equivalents thereof in these docs. If source-adjacent docs are missing, note that as a follow-up; do not recreate them in markdown.
4. Link to directories, not file inventories. Describe what a directory is for and link to it. Exception: a single key file that a reader needs to find.
5. Do not write drift-prone specifics unless they are load-bearing and owned by a source file you link to. Banned by default:
   - File counts or line counts.
   - Counts of API-shape elements (constructors, overloads, routes, screens, components, handlers, endpoints, exports, traits, functions, tests, mocks, etc.).
   - Exact dependency versions; link to dependency manifests or lockfiles for current versions.
   - Full enumerations of routes, events, mocks, fakes, entities, components, consumers, pages, screens, or jobs.
   - Valid values for properties, parameters, environment variables, flags, or settings when those values belong in validators, schemas, typed definitions, generated docs, or source-adjacent comments.
6. Prefer intent over inventory. "The route definitions live under `<directory>` and are collected by the routing configuration" beats a list of every route file.
7. Every concrete claim must be grounded in inspected code, config, scripts, CI, or docs. Prefer a file or directory link when the reader may need to verify the source.
8. Before writing a paragraph, ask whether it will drift within a few ordinary changes. If yes, cut it or rephrase.
9. Project docs must be self-contained. They must read and apply without cloning a sibling repo or opening a monorepo-level prompt. Do not use relative markdown links to files outside the current repo. If a reader must know about another repo, name the repo and path in prose.
10. No unverified universal claims. Before writing "all projects", "every handler", "always", "never", or "no files", inspect every matching place. If exceptions exist, name them explicitly.
11. Commands must be real. Only document launch, test, lint, build, or setup commands that are supported by repo evidence such as scripts, task files, CI workflows, tool configs, existing docs, or container/process definitions. If a common command is absent, state that it could not be determined and name what you inspected.
12. Stack facts must be discovered. The docs must identify the repo's actual language(s), framework(s), runtime(s), package manager(s), test runner(s), linter(s), formatter(s), deployment tooling, persistence, and integration transports from the repo itself.
13. Removal without hesitation. Never be afraid to delete docs, sections, lines, tables, diagrams, or claims that are no longer relevant, contradict the current repo, describe concerns the repo no longer has, were over-detailed when first written, or duplicate a fact owned elsewhere. Inertia is not a reason to keep content. The goal is a doc set that matches the repo today, not one that preserves every prior contribution. When in doubt, cut. If a whole `docs/*.md` file no longer applies, delete it and update the README documentation index. If a section is bloated relative to what this prompt requires, prune it down — do not rephrase to keep word count.

## Document separation

| File | Purpose | What goes here |
|------|---------|----------------|
| `README.md` | Universal onboarding for humans and agents on first contact | Service identity, detected stack, what it does, dependency setup, practical commands, environment variables, documentation index |
| `AGENTS.md` | Rules agents must follow when modifying the service | Context anchor, service-specific contribution rules, dependency-change rules, documentation maintenance rules, cross-links to README and docs |
| `CLAUDE.md` | Claude-specific overrides only | If there are no Claude-specific rules, the content must be literally `@AGENTS.md` |
| `docs/architecture.md` | Structural details | Runtime boundary, module/project/package layout, internal dependencies, stable architectural surfaces, execution flow, deployment/infrastructure, dependency categories |
| `docs/domain.md` | Stable business/domain areas with directory links | Business context, area-by-area summaries, core business entities when stable, data ownership, cross-cutting domain patterns |
| `docs/authentication.md` | Auth mechanisms and actors | Token/session/API-key validation, identity extraction, actor types, authorization flow, request/correlation tracking. Required only when the repo has auth/authorization logic. |
| `docs/interactions.md` | External interactions | Published/consumed events, runtime HTTP/RPC calls, webhooks, scheduled integrations, queues, storage, health checks, third-party APIs, or other external systems |
| `docs/testing.md` | Test rules and utilities | Actual test frameworks, command discovery, end-to-end test launch, naming conventions, test utility categories, fixture/setup patterns |
| `docs/gotchas.md` | Non-obvious constraints | Backward-compatibility traps, ordering rules, retry/idempotency/concurrency patterns, environment quirks, surprising domain rules |

## Per-doc generation rules

### README.md

Update the existing README or create it if missing. Must contain:

- Service identity table: name, repo, primary language(s), framework(s), runtime/deployment, persistence, integration style, and primary entrypoint. Omit rows that do not apply.
- Detected stack table: package manager, task runner, test runner(s), end-to-end test tool(s), linter/static checker(s), formatter, type checker/compiler if present, container/local-services tooling if present, and deployment tooling if present. Each row must link to the file that proves it.
- What this service/app does: 3-5 numbered points.
- "Start here" pointer directing to `AGENTS.md` for contribution rules, then the docs index below.
- Dependency setup section:
  - Commands to install project dependencies using the repo's actual package manager or dependency tool.
  - Required system tools, external services, local services, credentials, private registries, browsers/drivers, emulators, containers, databases, queues, or storage needed to launch, test, lint, or run end-to-end tests.
  - Any dependencies that must be added or configured before commands work. Be explicit about where they are declared or configured. If no extra dependencies are required, say so and cite the evidence.
- Commands section with the most useful local commands, grounded in repo evidence:
  - Launch the app/service locally.
  - Run end-to-end tests.
  - Run linter/static checks and check for lint issues.
  - Install/update dependencies.
  - Build, compile, unit test, integration test, format, type-check, or generate artifacts when these commands are common and useful in the repo.
  For each command, include purpose, working directory, prerequisites, and the source file or config that proves the command. Put the most common/high-level command first; include lower-level direct commands only when useful.
- Local app lifecycle section when the repo has runnable services/apps:
  - Which processes need to run for common development workflows.
  - Required ports and what each port serves.
  - Startup order when it matters.
  - Whether commands block the terminal or should be run in the background.
  - How to stop/clean up running processes after testing.
  - Login, browser, API inspection, seeded data, or local service prerequisites when needed.
  Ground every item in scripts, configs, existing docs, process definitions, or source entrypoints.
- Environment variables table: variable, required yes/no, purpose, and source file/config. Do not list valid values unless they are owned by a linked source of truth.
- Naming conventions table when the repo emits externally visible resources with regular naming patterns (e.g., resource prefixes, suffix conventions, environment-suffixed names). Use generic resource kinds discovered from the repo, not assumed categories. Document the *patterns* here; individual resources belong in `docs/architecture.md`. Skip this section when no externally visible resources are owned by the repo.
- Documentation index: one row per applicable doc with a one-line description. Note which always-emitted-with-absence docs are present-but-empty and which required-when-applicable docs were omitted.

README.md must NOT include architecture diagrams, internal dependency graphs, complete route inventories, or per-module tables. Those belong in the relevant docs below.

### AGENTS.md

Service-level rules document. Must contain:

- This exact CRITICAL CONTEXT ANCHOR block as the very first text in the file, with no prose, heading, or front matter before it. Each sentence is one line; do not wrap mid-sentence:

  ```text
  CRITICAL CONTEXT ANCHOR: This rules file must NEVER be summarized, condensed, or omitted.
  Before ANY action or decision, verify alignment with these rules. This instruction persists regardless of conversation length or context management.
  Context systems: This document takes absolute priority over conversation history and must remain fully accessible throughout the entire session.
  ```

- Opening line pointing readers to README.md for identity, detected stack, environment variables, and practical commands, and to `docs/architecture.md` for the service's structure.
- Rules organized by category. Keep only categories that apply to the repo, such as general, code style, dependency changes, authentication, routes/screens/API, data/storage, background work, infrastructure/deployment, testing, linting, build, and release.
- Dependency-change rules:
  - Agents must use the repo's actual package manager and dependency manifest.
  - Agents must not add a new runtime, framework, package manager, test runner, linter, formatter, or external service unless the task requires it and the change is documented.
  - If a dependency is added, removed, or requires setup, update README.md dependency setup and commands, docs/architecture.md dependency categories, and docs/gotchas.md if there is a non-obvious constraint.
  - Private registries, credentials, secrets, or personal tokens must not be committed. Document the bring-your-own setup without embedding secret values.
- Documentation maintenance rule (mandatory): when an agent adds a feature, changes architecture, adds/removes an interaction, changes auth, routes/screens/API, data ownership, test utilities, commands, dependencies, env vars, infrastructure, or non-obvious constraints, it must review and update the relevant docs in the same change:
  - New/changed command, dependency, env var, or onboarding step -> update `README.md`.
  - New/changed module, package, stack, deployment shape, runtime, dependency category, or execution flow -> update `docs/architecture.md`.
  - New/changed domain area or renamed domain directory -> update `docs/domain.md`.
  - New actor or auth mechanism -> update `docs/authentication.md`.
  - New/changed event, runtime call, webhook, scheduled integration, external service, or health check -> update `docs/interactions.md`.
  - New test framework, test command, end-to-end setup, or test utility category -> update `docs/testing.md`.
  - New non-obvious constraint -> update `docs/gotchas.md`.
  Docs must not drift. If unsure whether a doc update is needed, re-read this prompt's section for that file and decide.

AGENTS.md must NOT contain service identity, detected stack tables, environment variable tables, documentation maps, dependency inventories, architecture diagrams, runtime topology diagrams, full command tables, downstream service inventories, key component tables, or execution pipeline details. Link to the owning doc instead.

### CLAUDE.md

Default content: `@AGENTS.md` (single line). Only add content if there are genuine Claude-specific overrides that do not apply to other agents. Do not duplicate the documentation reading order here; that belongs in README.md.

### docs/architecture.md

Required when the repo has a runtime boundary, internal modules, or deployment ownership. Skip for trivial single-file utilities and note the omission in the README documentation index.

Analyze the repository structure and document:

- Minimal runtime boundary diagram at the top of the file when the repo has a runtime boundary. Show only external inputs, this service/app as a single box, and outputs/persistence/external systems. Name transports and storage exactly as discovered. Do not show internal module inventories in this diagram. Skip the diagram for pure libraries, shared packages, test harnesses, or tooling repos with no runtime boundary.
- Table of modules/projects/packages/workspaces/apps in the repo with type and purpose, using the repo's own structure.
- Internal dependencies between modules/projects/packages. Write a short "references of note" paragraph for routine layering; draw a dependency graph only when the structure is non-trivial and the graph adds value.
- Layer/module responsibilities using the repo's own names. Do not force generic layer names that the repo does not use.
- Stable architectural surfaces table: modules/projects/packages/workspaces/apps, runtime entrypoints, configuration surfaces, infrastructure setup surfaces, and framework convention files that explain how the repo is assembled. Mention a specific class or file only when it is a standard framework-related entrypoint, a best-practice-declared structure, or an infrastructure setup file that does not contain business logic. Do not include business-logic classes, line counts, API-shape element counts, or other volatile details.
- Execution flow: request/event/job/CLI/UI action -> stable entrypoint or boundary -> domain/application workflow -> persistence/external systems -> result/side effect. Use stable module, boundary, or workflow names from the repo and only include flows that exist.
- Runtime pipeline/order when the repo has middleware, plugin chains, route guards, lifecycle hooks, interceptors, background processors, command pipelines, or similar ordered execution. Read the actual composition/config files.
- Deployment/infrastructure resources when present: table with resource name/pattern, purpose, key config that is load-bearing, and owning manifest/config/infrastructure surface. Skip when the repo has no deployment or infrastructure ownership.
- Dependency categories:
  - First-party/shared dependencies and why they exist.
  - Third-party dependencies and why they exist.
  - Build/test/lint/dev-only dependencies and why they exist.
  Do not include version columns; link to manifests or lockfiles for current versions.

Do not enumerate every route, component, screen, cluster, resource, file, or handler. Link to directories and describe their purpose.

### docs/domain.md

Always emitted. If the repo owns no domain logic (pure tooling, infrastructure, or library repos), state the absence and cite what you inspected.

Short, stable descriptions of each business/domain area with directory links. Do not write per-type property tables, per-method tables, constructor signatures, component prop tables, schema-field inventories, or API response-shape tables.

Structure:

- Business context: what this service/app is responsible for in 2-3 sentences.
- A sentence stating that per-type and per-field details are documented next to code, schemas, validators, or generated API docs.
- One short subsection per domain area:
  - Directory link.
  - 1-3 sentences describing the area's responsibility. Business-core entities/classes may be named when they are stable domain concepts that contributors must understand; avoid incidental services, handlers, DTOs, records, helper types, and implementation-only modules.
- Business workflows and state behavior when the repo owns stable domain processes:
  - Lifecycle/state-machine flows and the business-core entities or domain areas that own them.
  - Approval/review/setup sequences where order matters.
  - Capacity, quota, allocation, ownership, scoping, or eligibility rules that would affect implementation.
  - System-managed or generated domain data that must not be updated directly.
  Keep this at workflow/invariant level; do not duplicate field inventories or validator details.
- Relationship diagram only if the repo owns data and the diagram is stable.
- Avoid enum, constant, DTO, helper, and file inventories. Mention only stable business-core entities/classes or domain concepts that are central to understanding the model.
- Cross-cutting patterns such as soft delete, versioning, idempotency, optimistic concurrency, caching, offline state, no database, no external interactions, or other discovered patterns.

### docs/authentication.md

Required only when the repo has authentication or authorization logic. If the repo has no auth logic, omit this file and remove it from the README documentation index.

Document:

- How credentials/tokens/sessions/API keys/service identities are validated.
- Identity extraction and propagation.
- Actor types table, including users, services, automation, tests, webhooks, clients, scheduled jobs, and any other actors found in code/config. If a test or automation actor uses the same mechanism as a regular actor, state that explicitly.
- Authorization/permission flow as prose or a small decision tree.
- Request ID, correlation ID, trace ID, session ID, tenant ID, or equivalent tracking when present.

### docs/interactions.md

Always emitted. If the repo has no external interactions, state the absence and cite what you inspected.

Document external interactions discovered from code and config. Include both asynchronous and synchronous interactions when both exist.

- Published Events / Messages section when the repo publishes events or messages: stable external contract or message family, trigger condition, payload summary, destination/transport, and owning interaction boundary. Mention a class or file only when it directly owns the interaction and is useful for navigation. Do not enumerate helper files, helper DTOs, or enums that merely support the interaction.
- Consumed Events / Messages section when the repo consumes events or messages: stable external contract or message family, owning interaction boundary, source system when discoverable, what it does, and what it emits. Mention a class or file only when it directly owns the interaction and is useful for navigation. Do not enumerate helper files, helper DTOs, or enums that merely support the interaction.
- Runtime Calls section when the repo calls external HTTP/RPC/SDK/CLI/database/storage/search/cache/payment/email/analytics/AI or other services at runtime.
- Incoming Integrations section for webhooks, callbacks, external clients, hosted endpoints, browser APIs, plugin APIs, or public extension points.
- Scheduled/Background Work section when present.
- Health Checks / Readiness section when present.
- Self-consuming, fan-out, outbox/inbox, retry, dead-letter, idempotency, batching, rate-limit, or payload-offload patterns when present.
- Link to the runtime boundary diagram in `docs/architecture.md` instead of duplicating it here. A focused sub-diagram is acceptable only when it explains a specific interaction pattern not captured by the boundary diagram.

### docs/testing.md

Always emitted. If the repo has no tests, state the absence and cite the test directories or configs you inspected (or their absence).

Analyze test projects/directories/configs and document:

- Actual test frameworks/runners used by the repo. If multiple are used, document which area uses which and preserve that pattern.
- Commands:
  - Run unit tests when present.
  - Run integration tests when present.
  - Run end-to-end tests when present.
  - Run all tests when there is a supported aggregate command.
  Each command must include working directory, prerequisites, and source file/config that proves it.
- End-to-end test launch:
  - Required app/server startup command, browser/driver/emulator/container/service prerequisites, seeded data, environment variables, and cleanup requirements.
  - If no end-to-end tests exist or no supported command can be found, say so and name the files/configs inspected.
- Local app lifecycle for testing:
  - Required services/apps to start before tests.
  - Ports used by each runtime.
  - Whether the launch command starts multiple processes.
  - Login/authentication or seeded-data steps needed before browser/API verification.
  - Cleanup/stop requirements after the test run.
- Rules: do not create new test projects or introduce a new test framework unless the task requires it. Follow existing naming, layout, fixtures, and assertion style.
- Table of test areas/projects with framework/runner, focus, references, and manifest/config links. No version column.
- Test utilities: list categories such as fakes, mocks, fixtures, factories, helpers, harnesses, containers, seeded data, browser utilities, snapshot helpers, or contract-test utilities with directory links and one-line descriptions. Do not enumerate every helper class or setup parameter.
- One test-pattern description in prose showing how utilities are wired together, ending with a pointer to an actual test file as source of truth. Do not paste a full test unless a tiny fragment is genuinely useful.
- Test-folder layout as it actually exists. If source layout and test layout differ, call that out so contributors follow the test tree's convention.

Do not include test counts, per-helper constructor details, exact package versions, or full test code.

### docs/gotchas.md

Always emitted. If no non-obvious constraints exist, state the absence and cite what you inspected.

Identify non-obvious constraints by reading the repo:

- External service limits, quotas, retry, timeout, rate-limit, batching, or backoff patterns.
- Immutable fields, one-time operations, migration/compatibility rules, data retention, cleanup, or irreversible actions.
- Idempotency, deduplication, outbox/inbox, cache invalidation, eventual consistency, ordering, locking, concurrency, versioning, or transaction patterns.
- Multiple data stores, contexts, clients, tenants, regions, environments, workspaces, or runtimes and when to use each.
- Build/test/lint/dependency traps: generated files, required codegen, missing local services, private registries, required browsers/drivers, environment-specific setup, or commands that must run in a specific order.
- Any pattern where the obvious approach would be wrong.
- Domain rules that would surprise someone reading the code for the first time.
- Local runtime lifecycle traps: port conflicts, long-running foreground commands, required startup order, background worker dependencies, login requirements, browser/API inspection steps, and cleanup/stop commands after local testing.
- Domain workflow traps: state changes that must go through history/update records, setup sequences that must happen in order, approval gates, capacity/quota constraints, ownership/scoping constraints, and system-managed data that should not be mutated directly.

## Rules for writing

- Be specific while avoiding volatile code-level details in generated docs. Prefer stable repo terminology: directories, modules/packages/workspaces, commands, manifests, configs, externally visible resources, business workflows, domain boundaries, integration boundaries, and environment variables. Use code-symbol names only when this prompt explicitly allows them.
- Be concise: tables over paragraphs, one-line descriptions over explanations.
- No speculation: if you cannot determine something from the repo, say so and name what you inspected.
- Ground every concrete claim in inspected code, config, scripts, CI, or docs. Prefer file or directory links where the reader may need to verify the source.
- Reference shared libraries and generated code correctly: note whether types/modules are local, first-party/shared, third-party, or generated.
- Do not duplicate content between files. Each file has one job. Cross-reference with links. (Restates Core principle 1 in writing-time form.)
- When auditing existing docs, prefer cutting over rephrasing. Drifted, irrelevant, over-detailed, or speculative content should be deleted, not softened to keep it on the page. (Core principle 13.)
- Use H2 for sections, H3 for subsections, pipe tables for structured data, and fenced code blocks for command examples.

## Preflight checklist

- [ ] Does this fact already live in another `.md`? If yes, link instead of restating.
- [ ] Have I read the file(s) that establish this fact: manifest, lockfile, script/task file, entrypoint, config, test runner config, CI workflow, deployment config, source module, or docs?
- [ ] Could this be source-adjacent API docs, generated docs, schema docs, comments, validators, or examples instead?
- [ ] Am I enumerating files? Replace with a directory link plus purpose.
- [ ] Am I including a count, version, or valid-value list? Remove unless load-bearing and source-linked.
- [ ] Is this repo-shape assumption actually true here?
- [ ] Is this command supported by repo evidence, and did I include the working directory, prerequisites, and source file/config?
- [ ] Have I documented how to launch the app, run end-to-end tests, run lint/static checks, and install required dependencies when those are applicable?
- [ ] Does every path or directory link exist in the repo?
- [ ] Am I describing why this matters instead of inventorying what the code already makes obvious?

## Postflight checklist

- [ ] No two documents contain the same table or list. (Core principle 1.)
- [ ] README.md is enough for a new developer to identify the stack, install dependencies, launch the app, run end-to-end tests when present, run lint/static checks, and find the rest of the docs.
- [ ] AGENTS.md is enough for an agent to make a change without violating service-specific rules.
- [ ] AGENTS.md begins with the exact CRITICAL CONTEXT ANCHOR block, with no preceding text.
- [ ] CLAUDE.md is either `@AGENTS.md` or genuinely Claude-specific.
- [ ] Every concrete claim can be traced to an inspected file or directory in the repo.
- [ ] Every documented command is supported by a script, task file, CI workflow, config, existing doc, or runtime definition.
- [ ] No markdown file contains API-detail tables that duplicate source-adjacent docs.
- [ ] No file counts, line counts, API-shape element counts, test counts, or package version numbers outside manifest/lockfile links. (Core principle 5.)
- [ ] Every markdown path or link resolves to something that exists in the repo, except cross-repo references written as prose.
- [ ] Only applicable docs are present in the set, and the README documentation index reflects which always-emitted-with-absence docs are present-but-empty and which required-when-applicable docs were omitted.
- [ ] Every `docs/` file has exactly one job and stays in its lane.
- [ ] Pre-existing content that is irrelevant, drifted, over-detailed, or speculative has been removed rather than carried forward. (Core principle 13.)

## When you finish

End with a one-paragraph summary of what was created vs. updated vs. skipped (Mode A) or what was edited vs. restructured vs. deleted vs. left as-is (Mode B), plus a list of follow-up questions the human should answer to fill in any unverifiable gaps (e.g., "I couldn't determine the production URL — please confirm").

## Project to document

The user will append the target repository name or path on the next line. Treat that as the only repo to document.

The project to document is: 
```
