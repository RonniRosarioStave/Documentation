---
description: Register a newly added product folder into the Stave Apps docs hub (sidebar + home index)
argument-hint: <folder-name> (e.g. stave-notify) — optional if obvious from recent changes
---

You are processing a new product into the **Stave Apps documentation hub** (a Docsify v4 site). The user has added a product folder and wants it registered consistently. The product to process is: **$ARGUMENTS** (if empty, detect the new/untracked product folder via `git status` and the file listing; if more than one candidate exists, ask which one).

Follow this checklist exactly, in order. Do not skip steps. Keep changes minimal and match the existing style of each file.

## 1. Normalize the product folder
- The product must live at `<folder>/README.md`. Docsify serves `<folder>/` as `<folder>/README.md`, so the doc file MUST be named `README.md`.
- If the markdown doc inside the folder has a different name, `Move-Item` it to `README.md`.
- If the user instead dropped a loose `*.md` at the repo root, create the folder and move the file into it as `README.md`. Pick a clean lowercase-hyphenated folder name (confirm with the user if ambiguous).
- Ensure `<folder>/assets/` exists with a `.gitkeep` (for images / exported SVGs / `.mmd` sources). Create them if missing.
- Do NOT edit the product doc's content — only its location/name.

## 2. Derive display name + description
- Read the product `README.md`. Take the **display name** from its H1 (strip scope/backticks, e.g. "Stave Data Tools (`x_stave_dt`) — Technical Documentation" → "Stave Data Tools").
- Draft a **one-line description** from the doc's intro blockquote / first paragraph. Keep it concise and consistent with the existing Data Tools entry.

## 3. Register in the sidebar — `_sidebar.md`
- Add `  - [<Display Name>](<folder>/)` under the `**Products**` group.
- Keep existing entries; place the new one in a sensible order (alphabetical is fine). Don't duplicate if it already exists.

## 4. Register on the home page — root `README.md`
- Add `- **[<Display Name>](<folder>/)** — <one-line description>.` under the `## Products` heading, above the `<!-- Add new products ... -->` comment.
- Don't duplicate if already present.

## 5. Verify
- If a `docsify-cli serve` process is already running, just confirm the new route loads; otherwise start `npx docsify-cli serve .` (it picks a free port — read the port from its output).
- Fetch and confirm HTTP 200 for `/<folder>/README.md`, `/_sidebar.md`, and `/README.md` (use `Invoke-WebRequest -UseBasicParsing`).
- Report the local URL so the user can eyeball the rendered sidebar nav.

## 6. Report
Summarize in a short table: folder normalized (y/n), display name, description used, lines added to `_sidebar.md` and `README.md`, and the local URL. Flag anything that needed a judgment call (e.g. folder rename, ambiguous name) so the user can correct it.

Reminders:
- The per-product section nav (1, 2, 3…) is auto-generated from the doc's `##`/`###` headings via `subMaxLevel` — never hand-maintain a TOC.
- Mermaid diagrams render from inline ` ```mermaid ` blocks; no separate registration needed.
