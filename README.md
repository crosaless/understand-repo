# understand-repo

A Claude Code + Codex skill that analyzes any repository and generates a self-contained HTML study guide — designed with a **LiquidGlass aesthetic** (light/dark, glassmorphism, iOS-inspired) and built for both newcomers and contributors.

Drop it on an unfamiliar codebase and get a one-page visual onboarding doc: overview, architecture diagram, data-flow walkthrough, annotated code excerpts, extension points, and a minimal runnable example — all in a single HTML file that works offline.

![Study guide preview](examples/sample-output/study-guide-preview.png)

---

## Quick start

> **Prerequisites:** [Claude Code](https://docs.anthropic.com/en/docs/claude-code) or [Codex CLI](https://github.com/openai/codex) installed and authenticated.

### Step 1 — Install the skill

**Claude Code (macOS / Linux):**
```bash
git clone https://github.com/crosaless/understand-repo.git ~/.claude/skills/understand-repo
```

**Claude Code (Windows — PowerShell):**
```powershell
git clone https://github.com/crosaless/understand-repo.git "$env:USERPROFILE\.claude\skills\understand-repo"
```

**Codex (macOS / Linux):**
```bash
git clone https://github.com/crosaless/understand-repo.git ~/.codex/skills/understand-repo
```

**Codex (Windows — PowerShell):**
```powershell
git clone https://github.com/crosaless/understand-repo.git "$env:USERPROFILE\.codex\skills\understand-repo"
```

You only need to do this once. The skill is now available in every project.

---

### Step 2 — Navigate to the repo you want to analyze

Open a terminal and go to the project you want to understand:

```bash
cd /path/to/the-repo-you-want-to-analyze
```

Then start Claude Code or Codex inside that directory:

```bash
claude   # Claude Code
codex    # Codex
```

> The agent must be started **inside the target repo**, not inside `understand-repo` itself.

---

### Step 3 — Invoke the skill

Type any of these in the agent prompt:

```
analyze this repo and generate a study guide
```
```
explain this codebase
```
```
onboard me to this project
```

Or invoke it explicitly:

**Claude Code:**
```
/skill understand-repo
```

**Codex:**
```
$understand-repo
```

The agent will read the codebase, analyze it, and write the output files.

---

### Step 4 — Open the study guide

The skill generates a `docs/` folder inside the **target repo** with these files:

```
docs/
├── study_guide.html      ← open this in your browser
├── architecture.mmd      ← Mermaid source of the diagram
├── summary.md            ← markdown version (for PRs, chats, READMEs)
└── index.json            ← structured metadata for tooling
```

**To open the HTML file:**

**macOS:**
```bash
open docs/study_guide.html
```

**Linux:**
```bash
xdg-open docs/study_guide.html
```

**Windows (PowerShell or Command Prompt):**
```powershell
start docs\study_guide.html
```

**Windows (Explorer):** Navigate to the `docs/` folder and double-click `study_guide.html`.

> The file is 100% self-contained — no server required, no build step, no dependencies to install. Just open it.

---

## What the study guide looks like

The generated page has 8 sections:

| Section | Content |
|---|---|
| Hero | Project name, type badge, commit hash, generation date |
| What & Why | What the project does and why it exists |
| Architecture | Interactive Mermaid diagram |
| How it works | Step-by-step data-flow trace, module by module |
| Key Files | Table of central files with a one-sentence role each |
| Code Deep Dive | Annotated excerpts from the most important files |
| Extension Points | Where and how the repo can be extended or customized |
| Try it now | Minimal runnable example + local setup commands |

### UX features

- **Light / dark toggle** — reads `prefers-color-scheme` on first load, persists in `localStorage`
- **LiquidGlass aesthetic** — iOS-inspired glassmorphism, specular highlights, ambient color orbs
- **Lucide icons** — inline SVG icons, fully offline (no CDN required)
- **Color picker** — customize the accent color directly in the page header
- **Language toggle** — switch between English and Spanish (EN / ES)
- **Collapsible sections** — every card can be collapsed or expanded
- **Sidebar sub-navigation** — Key Files and Code Deep Dive expand per-file links
- **Scrollspy** — sidebar and breadcrumb track your reading position in real time
- **Progress bar** — shows how far through the guide you are
- **Copy-to-clipboard** — every code block gets a copy button on hover
- **Full-text search** — filters sections in real time as you type

### What requires internet

| Feature | Needs internet? |
|---|---|
| All icons | No — embedded inline |
| All text, layout, interactions | No |
| Architecture diagram (Mermaid) | Yes — loads from `cdn.jsdelivr.net` |
| Syntax highlighting (highlight.js) | Yes — loads from `cdnjs.cloudflare.com` |

Without internet, the diagram shows a fallback message and code blocks appear without color — everything else works fine.

---

## How it works internally

When you invoke the skill the agent follows these steps:

1. **Read `SKILL.md`** — loads the instruction set that defines every analysis step and output format.
2. **Scan the file tree** — maps the directory structure and reads identity files (`README`, `package.json`, `pyproject.toml`, `go.mod`, etc.) to classify the project type.
3. **Compute a Documentation Score** — evaluates README depth and comment density. This determines whether Extension Points will use structural heuristics + LLM inference (well-documented repos) or LLM-only with `(inferred)` labels (sparse docs).
4. **Centrality analysis** — identifies the 3–5 files with the highest import fan-in / fan-out and picks narrative fragments from them for the Code Deep Dive section.
5. **Fill the template** — reads `templates/study_guide_template.html` and replaces all `{{PLACEHOLDER}}` tokens with the generated content.
6. **Write output files** — saves `study_guide.html`, `architecture.mmd`, `summary.md`, and `index.json` to `docs/`.

---

## Extension Points strategy

The skill adapts automatically based on how well-documented the target repo is:

- **Well-documented repo** — combines structural heuristics (abstract classes, plugin files, `@public` markers), doc analysis (README "Contributing"/"Extending" sections), and LLM inference.
- **Sparse docs** — LLM inference only, constrained to structurally obvious seams. All inferred points are marked `(inferred)` in the output.

---

## Using with other agents (Cursor, Aider, Hermes, custom)

Copy `SKILL.md` into your agent's instruction set or system prompt. The template lives in `templates/study_guide_template.html` — point your agent at it.

---

## Customization

### Change the output directory

Ask the skill at invocation time:

```
generate the study guide into wiki/ instead of docs/
```

### Change the theme

Edit `templates/study_guide_template.html`. The CSS custom properties at the top of the `<style>` block control everything:

```css
[data-theme="dark"] {
  --accent:    #f4f4f8;   /* primary accent color */
  --glass-bg:  rgba(255,255,255,0.07);
  --orb-1:     rgba(80, 80, 200, 0.10);
  /* ... */
}
```

A full palette change is a few variable edits. The built-in color picker in the page header also lets you set a custom accent per-session without touching the template.

### Add a new section

1. Add a new `{{MY_SECTION}}` placeholder to `templates/study_guide_template.html`.
2. Add a matching step in `SKILL.md` that tells the agent what to generate for that placeholder.
3. Next time the skill runs, the new section appears automatically.

---

## Placeholders reference

| Placeholder | Content |
|---|---|
| `{{PROJECT_NAME}}` | Repository name |
| `{{PROJECT_TYPE}}` | CLI / Web App / Library / etc. |
| `{{GENERATED_AT}}` | ISO 8601 date |
| `{{COMMIT_HASH}}` | Short git hash or `—` |
| `{{OUTPUT_LANGUAGE}}` | `en` or `es` |
| `{{WHAT_AND_WHY}}` | HTML — overview + purpose |
| `{{MERMAID_DIAGRAM}}` | Raw Mermaid source |
| `{{HOW_IT_WORKS}}` | HTML — ordered data-flow steps |
| `{{KEY_FILES_ROWS}}` | HTML — `<tr>` rows for the files table |
| `{{CODE_DEEP_DIVE_BLOCKS}}` | HTML — `.code-module` blocks |
| `{{EXTENSION_POINTS_LIST}}` | HTML — `.ext-point` blocks |
| `{{MINIMAL_EXAMPLE}}` | Plain text snippet |
| `{{LOCAL_SETUP}}` | Plain text install + run commands |

---

## Edge cases

- **Monorepos** — generates one guide per workspace under `docs/<package>/`.
- **No README** — infers purpose from package metadata; marks as `(inferred)`.
- **Large repos (>500 files)** — delegates exploration to a subagent/explorer when the host agent supports it.
- **Empty / scaffolding repos** — produces a minimal guide with next-step suggestions.

---

## Examples

See [`examples/`](examples/) for a sample output generated against this repo.

---

## Contributing

Issues and PRs welcome. Particularly useful contributions:

- Better project-type detection heuristics.
- Additional language support for Code Deep Dive fragments.
- Adapters for other agent frameworks.

---

## License

MIT — see [LICENSE](LICENSE).
