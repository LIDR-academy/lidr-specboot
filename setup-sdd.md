# Spec-Driven Development Setup Instructions for LIDR Specboot

**Purpose**: These instructions tell any coding agent how to install and configure this repository's OpenSpec-aligned workflow into any target project automatically.

Repository source:
- `https://github.com/LIDR-academy/lidr-specboot`

Raw setup file URL:
- `https://raw.githubusercontent.com/LIDR-academy/lidr-specboot/main/setup-sdd.md`

---

## What the Coding Agent Must Do

Execute these steps in order. Perform the setup directly; do not ask the user to run commands unless a manual approval is required.

### 1) Preflight Validation

Verify all of the following in the current working directory (target project root):

1. It is a git repository (`git rev-parse --is-inside-work-tree`)
2. User has write permissions
3. `node` is available and version is `>= 20.19.0`
4. `npm` is available

If any check fails, stop and report exactly what is missing.

### 2) Detect Existing Specboot Installation Channel

This setup must work whether `specboot` was installed from npm or provided by a Claude Code plugin.

Detection and compatibility rules:

1. If the project already contains any of these paths, treat Specboot as already applied and skip re-import:
   - `ai-specs/`
   - `docs/base-standards.md`
   - `CLAUDE.md` or `AGENTS.md` symlinked to `docs/base-standards.md`
2. If the user says Specboot was installed via plugin, do **not** require a global npm install of `@lidr/lidr-specboot`.
3. If no Specboot artifacts exist yet, use the automated bootstrap command in step 5.

### 3) Install OpenSpec (if needed)

If `openspec` is not available, install it globally:

```bash
npm install -g @fission-ai/openspec@latest
```

Re-verify with:

```bash
openspec --version
```

### 4) Initialize OpenSpec (if needed)

If the project does not have OpenSpec initialized yet, run:

```bash
openspec init
```

If it is already initialized, keep existing artifacts and continue.

### 4) Import LIDR Specboot Into the Project

Copy this repository's baseline files into the target project **without overwriting existing files**.
Keep the source `README.md` as onboarding instructions inside `ai-specs/` instead of replacing the project's root README.

Recommended automation sequence:

```bash
tmp_dir="$(mktemp -d)"
git clone --depth 1 https://github.com/LIDR-academy/lidr-specboot.git "$tmp_dir/lidr-specboot"
cp -rn "$tmp_dir/lidr-specboot/"* .
mkdir -p ai-specs
cp -n "$tmp_dir/lidr-specboot/README.md" "ai-specs/specboot-instructions.md"
rm -rf "$tmp_dir"
```

Important:
- Use `cp -rn` exactly to preserve existing project files (especially existing root `README.md`)
- Store template onboarding instructions at `ai-specs/specboot-instructions.md` for future setup reference
- Keep hidden directories such as `.claude/` and `.cursor/` when present in source
- Remove temporary clone after copy (`rm -rf "$tmp_dir"`)

### 6) Ensure OpenSpec Config Includes This Technical Context

Update the project's OpenSpec config file (`config.yml` or `openspec/config.yaml`, whichever exists) so context includes:

- `docs/base-standards.md` as single source of truth
- `docs/backend-standards.md`
- `docs/frontend-standards.md`
- `docs/documentation-standards.md`
- `docs/api-spec.yml`
- `docs/data-model.md`
- `ai-specs/agents/backend-developer.md` for backend implementation guidance
- `ai-specs/agents/frontend-developer.md` for frontend implementation guidance
- `ai-specs/skills/` as reusable workflow guidance

Do not delete unrelated existing project context; merge safely.

If this project was set up through a plugin and no config file exists yet:
1. Run `openspec init`
2. Re-open `openspec/config.yaml`
3. Merge the required context entries listed above

### 7) Create a First Draft of Technical Context (When Applicable)

If the target project has missing or generic technical documentation in `docs/`, create a first draft before finishing setup.

Scope for first draft (adapt to repository reality):
- `docs/base-standards.md`
- `docs/backend-standards.md` (if backend exists)
- `docs/frontend-standards.md` (if frontend exists)
- `docs/documentation-standards.md`
- `docs/api-spec.yml` (if API exists)
- `docs/data-model.md` (if persistence/domain model exists)

Research requirements:
1. Perform deep repository research before drafting:
   - Detect stack, frameworks, architecture, testing tools, linting/formatting, and deployment conventions
   - Inspect source folders, package manifests, lockfiles, CI workflows, tests, and existing docs
   - Cross-check findings across multiple files before writing conclusions
2. Preserve existing project decisions; do not invent architecture that is not present.
3. Keep all content in English.

Prompt example (use with your coding agent to guarantee deep research):

```text
Create a first draft of this project's technical context in docs/ using the same structure and file set from LIDR Specboot.

Hard requirements:
- Perform deep repository research before writing anything.
- Analyze at least: package manifests, lockfiles, source tree, tests, CI/workflows, lint/format configs, and current documentation.
- Infer real stack, architecture, coding conventions, test strategy, API shape, and domain model from evidence in the repository.
- Keep existing structure and file names in docs/; do not change the template structure.
- If a section is missing evidence or is ambiguous, ask concise clarification questions first (short-answer format), then continue.
- Do not use placeholders like "TBD" when evidence exists in the codebase.
- Keep all technical artifacts in English.

Deliverables:
1) Updated docs/base-standards.md
2) Updated docs/backend-standards.md and/or docs/frontend-standards.md where applicable
3) Updated docs/documentation-standards.md
4) Updated docs/api-spec.yml and docs/data-model.md where applicable
5) A short evidence summary listing which files were used to infer each major decision
```

Missing section policy (mandatory):
- When a required section is missing or unclear, ask short clarification questions and wait for answers before finalizing that section.
- Use short-answer questions (one line each), for example:
  - "Backend framework? (Express/Fastify/Nest/Other)"
  - "Primary database? (PostgreSQL/MySQL/MongoDB/Other)"
  - "Frontend framework? (React/Vue/Angular/Other)"
  - "Testing stack? (Jest/Vitest/Cypress/Playwright/Other)"

### 8) Verify Symlink Integrity and Key Files

Verify that the imported structure is usable:

1. Root agent config files exist:
   - `AGENTS.md`
   - `CLAUDE.md`
   - `codex.md`
   - `GEMINI.md`
2. Core standards exist:
   - `docs/base-standards.md`
3. Skills and agents exist:
   - `ai-specs/skills/`
   - `ai-specs/agents/`
4. If `.claude` / `.cursor` symlinks exist, ensure they are not broken.

### 9) Completion Output (Required)

When done, report:

1. OpenSpec status (installed + initialized)
2. Files imported (high-level summary)
3. Which config file was updated and what sections were added/merged
4. Verification results
5. Any warnings (for example, files skipped because they already existed)

---

## Post-Installation Quick Usage

After installation, suggest this workflow:

```bash
/enrich-us <ticket-or-idea>   # optional
/ff <ticket-id>
/apply <ticket-id>
/verify <ticket-id>
/archive <ticket-id>
/commit
```

Also remind the user to customize `docs/` for their real stack/domain before generating production changes.

---

## Troubleshooting Rules

If setup fails:

1. Show the exact failing command
2. Explain root cause in one sentence
3. Propose the smallest safe fix
4. Continue automatically after applying the fix when possible

Do not silently skip failed steps.
