---
name: understand-repo
description: Analyze any repository and generate a self-contained, LiquidGlass-styled HTML study guide plus a Mermaid architecture diagram, a compressed markdown summary, and a structured JSON index. Use when the user asks to "analyze this repo", "generate a study guide", "explain this codebase", or wants a visual onboarding doc for a project.
---

# Repo Study Guide Skill

Universal instruction set to analyze any repository and produce a high-quality, self-contained study guide with a modern LiquidGlass aesthetic. Compatible with Claude Code, Codex, and other code agents that can read this skill and the bundled template.

## When to use

Trigger this skill when the user:
- Opens an unfamiliar repo and wants a fast mental model.
- Asks for "study guide", "onboarding doc", "architecture overview", or "explain this codebase".
- Wants a visual artifact (HTML + diagram) instead of a chat-only explanation.

## Inputs

- **Target repo path** (default: current working directory).
- **Output directory** (default: `docs/` relative to the repo root; override if the user asks).

---

## Execution Steps

### 1. Exploration phase

Goal: build a complete mental model before writing any output.

- **Scan tree** — map root and source folders (`src/`, `lib/`, `app/`, `pkg/`, `cmd/`, etc.).
- **Identity files** — read in parallel: `README.md`, `package.json`, `pyproject.toml`, `requirements.txt`, `Cargo.toml`, `go.mod`, `pom.xml`, `Gemfile`.
- **Type detection** — classify as one of: CLI, Web App, Library/SDK, ML/Data project, Monorepo, Infra/IaC.
- **Large repos (>500 files)** — delegate bounded exploration to a subagent when available. Request: entry points, top 10 files by import-fan-in, and the dependency graph at module level.

### 2. Analysis phase

#### 2a. Core analysis

- **Entry point** — locate `main.*`, `index.*`, `app.*`, `cli.*`, `__main__.py`, or the `"main"`/`"bin"` field in `package.json`.
- **Data flow** — trace one realistic input from entry → core logic → output. Note each module touched.
- **External surface** — APIs exposed, CLI commands, env vars, config files.

#### 2b. Documentation Score

Compute a score (HIGH / LOW) to determine the Extension Points strategy:

- **HIGH** if two or more of the following are true:
  - README has a "Contributing", "Extending", "Plugins", or "Customization" section
  - Source files contain `@api`, `@public`, `// override`, or `// extend` comments
  - Files named `plugin.*`, `middleware.*`, `extension.*`, `adapter.*`, `hook.*` exist
  - A `CONTRIBUTING.md` or `docs/contributing.*` file is present
- **LOW** otherwise.

Store the score — it is used in step 2d.

#### 2c. Centrality analysis (for Code Deep Dive)

Identify the 3–5 files with the highest structural centrality:

- **Fan-in**: files that are imported by many others (high dependents count).
- **Fan-out**: files that import many others (high dependencies count).
- Prefer `rg` for import/require/use patterns; fall back to `grep` if unavailable.

For each central file, identify the fragment that best illustrates the data flow described in the "How it works" narrative — not the entire file, just the relevant excerpt (function, class, handler, or transformer).

#### 2d. Extension Points (conditional on Documentation Score)

**If Documentation Score = HIGH → Strategy C+D:**
1. Pattern heuristics: abstract classes, interfaces, hooks, plugin/middleware/adapter files, `@public` exports.
2. Doc analysis: README "Contributing"/"Extending"/"Customization" sections, `@api`/`@public` comments.
3. LLM inference: reason over the core modules to identify natural extension seams.
4. Merge results; de-duplicate by file/location.

**If Documentation Score = LOW → Strategy D with guardrails:**
1. LLM inference only: reason over core modules for structural extension seams.
2. **Every point must be marked `(inferred)`** — no exceptions.
3. Constrain to structurally obvious seams (entry points, factories, registries, config objects). Do not invent extension points that have no structural basis.
4. Maximum 3–4 points. Prefer fewer, higher-confidence points over many uncertain ones.

---

### 3. Artifact generation

#### 3a. Architecture diagram

- Create a Mermaid `graph TD` (component/module relationships) or `sequenceDiagram` (data flow).
- **≤15 nodes** — high-level only, no leaf files.
- Save to `<output_dir>/architecture.mmd`.

#### 3b. HTML study guide

Load `templates/study_guide_template.html` (sibling of this `SKILL.md`) and replace all placeholders:

| Placeholder | Content |
|---|---|
| `{{PROJECT_NAME}}` | Repository name |
| `{{PROJECT_TYPE}}` | CLI / Web App / Library / ML / Monorepo / Infra |
| `{{GENERATED_AT}}` | ISO 8601 date (YYYY-MM-DD) |
| `{{COMMIT_HASH}}` | `git rev-parse --short HEAD` output, or `—` if not a git repo |
| `{{WHAT_AND_WHY}}` | HTML: 2–3 `<p>` tags — what it does + why it exists / problem it solves |
| `{{MERMAID_DIAGRAM}}` | Raw contents of `architecture.mmd` (no fencing) |
| `{{HOW_IT_WORKS}}` | HTML: ordered `<ol>` tracing the data flow step by step. Each step names the module touched. |
| `{{KEY_FILES_ROWS}}` | HTML: `<tr><td><code>path/to/file</code></td><td>one-sentence role</td></tr>` per file |
| `{{CODE_DEEP_DIVE_BLOCKS}}` | HTML: one `.code-module` block per central file (see format below) |
| `{{EXTENSION_POINTS_LIST}}` | HTML: one `.ext-point` block per extension point (see format below) |
| `{{MINIMAL_EXAMPLE}}` | Plain text: smallest runnable snippet |
| `{{LOCAL_SETUP}}` | Plain text: install + run commands |

**`{{CODE_DEEP_DIVE_BLOCKS}}` format (one block per file):**

```html
<div class="code-module" id="code-module-SLUG">
  <div class="code-module-header">
    <i data-lucide="file-code-2" style="width:15px;height:15px;color:var(--accent)"></i>
    <h3>path/to/file.ext</h3>
  </div>
  <p class="module-role">One sentence: why this file matters and what makes it central.</p>
  <div class="code-wrap">
    <pre><code class="language-LANG">// relevant excerpt here</code></pre>
    <button class="copy-btn"><i data-lucide="copy" style="width:11px;height:11px"></i> Copy</button>
  </div>
</div>
```

Where `SLUG` is a kebab-case version of the file name and `LANG` is the detected language (e.g. `javascript`, `python`, `go`, `typescript`).

**`{{EXTENSION_POINTS_LIST}}` format (one block per point):**

```html
<div class="ext-point">
  <div class="ext-point-header">
    <i data-lucide="plug-2" style="width:15px;height:15px;color:var(--accent)"></i>
    <h3>Extension Point Name <span class="badge-inferred">(inferred)</span></h3>
  </div>
  <p>Where it is and how to use it. Include the file path and function/class name.</p>
</div>
```

Omit `<span class="badge-inferred">(inferred)</span>` for points derived from structural evidence (Strategy C+D). Always include it for Strategy D points.

Save to `<output_dir>/study_guide.html`.

#### 3c. Markdown summary

Generate `<output_dir>/summary.md` — a compressed version suitable for pasting into a PR, chat, or README:

```markdown
# {PROJECT_NAME} — Study Guide Summary

**Type**: {PROJECT_TYPE} | **Generated**: {DATE} | **Commit**: {HASH}

## What & Why
{1–2 sentences}

## Architecture
```{mermaid}
{diagram}
```

## Key Files
| File | Role |
|------|------|
{rows}

## Extension Points
{bulleted list, one line each, (inferred) where applicable}

## Try it now
```{lang}
{minimal example}
```
```

#### 3d. JSON index

Generate `<output_dir>/index.json` — structured metadata for tooling consumption:

```json
{
  "name": "...",
  "type": "...",
  "generated_at": "...",
  "commit_hash": "...",
  "entry_points": ["path/to/main.ext"],
  "core_modules": [
    { "path": "path/to/file.ext", "role": "one-sentence role" }
  ],
  "extension_points": [
    { "name": "...", "location": "path/to/file.ext", "inferred": false }
  ],
  "doc_score": "HIGH"
}
```

---

## Edge cases

- **Monorepos** — detect `workspaces` (`package.json`), `pnpm-workspace.yaml`, Turborepo, Nx, Cargo workspaces. Generate one study guide per workspace package under `docs/<package>/`, plus a root index.
- **No README** — infer purpose from package metadata, top-level comments, and folder names. Mark the "What & Why" section as `(inferred)`.
- **Empty / scaffolding only** — produce a minimal guide noting it is a scaffold; suggest next steps.
- **Unknown language** — fall back to file-extension stats and any build configs found.
- **Documentation Score cannot be computed** — treat as LOW.

---

## Quality standards

- **English only** — professional technical English in all output.
- **Visual-first** — diagrams, tables, and code blocks over prose.
- **Zero fluff** — every sentence must help a first-time reader.
- **Self-contained** — HTML must open offline (CDN scripts are acceptable; the template ships fallbacks).
- **Reproducible** — same repo state → same output. Mark inferences explicitly.

---

## Final checklist (must pass before reporting done)

- [ ] `study_guide.html` opens in a browser without console errors.
- [ ] `architecture.mmd` parses (balanced brackets, no empty graph).
- [ ] Zero `{{PLACEHOLDER}}` tokens remain in `study_guide.html`.
- [ ] All Lucide icons render (check DevTools — no missing icon warnings).
- [ ] Light/dark toggle works; system preference is respected on first load.
- [ ] Copy buttons appear on hover over every code block.
- [ ] Sidebar sub-items populate for Key Files and Code Deep Dive.
- [ ] `summary.md` exists and contains no unreplaced placeholders.
- [ ] `index.json` is valid JSON (pipe through `python -m json.tool` or equivalent to verify).
- [ ] Extension Points marked `(inferred)` only when Documentation Score = LOW or evidence is absent.

---

## Activation trigger

- Manual: `$understand-repo` (Codex), `/skill understand-repo` (Claude Code).
- Contextual: "Analyze this repo", "Generate a study guide", "Explain this codebase", "Onboard me to this project".
