# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A personal cheat-sheet / quick-reference collection. Three subject areas:

- `ArchLinux/` — 17 guides on Arch Linux administration (pacman, systemd, networking, etc.) + `_index.md`
- `DotNet/` — guides on the .NET CLI, IConfiguration, custom templates, and WPF hosting
- `OpcUa/` — OPC UA protocol overview and .NET (OPC Foundation SDK) usage + `_index.md`

`README.md` at the root is the canonical table of contents and links to every file.

## No build, no tests, no CI

This repo is plain markdown. There is no static site generator (no Hugo/MkDocs/Jekyll/Docusaurus config), no `package.json`, no `Makefile`, no `.github/workflows`. Files are read directly on GitHub or in an editor. Do not invent build/test/lint commands — there are none.

## Authoring conventions

The dominant style (used by all `ArchLinux/*.md`, all `OpcUa/*.md`, and three of the four `DotNet/*.md` files) is:

1. **No frontmatter.** Files start directly with `# Title`.
2. **One short intro paragraph** under the H1 stating the scope of the guide.
3. **H2 sections** for major topics, **H3** for subsections.
4. **Descriptive prose before each code block** — one or two sentences explaining what the commands do and when to use them. This is the most important convention: do not dump bare commands.
5. **Fenced code blocks with a language tag** (` ```bash `, ` ```csharp `, ` ```ini `, etc.). Inline shell snippets use single backticks.
6. **Tables** for reference/comparison data (commands, flags, types, paths).
7. **Blockquote callouts** for warnings: `> **Warning:** …`.
8. **Cross-links** between files use relative markdown links (e.g. `[Overview](overview.md)`); the README uses paths like `[Topic](ArchLinux/file.md)`.

A good example to mirror: `ArchLinux/pacman.md`.

## Known inconsistencies (don't perpetuate)

- **`DotNet/dotnet-cmd.md` is a 13-line stub** of bare shell commands with no headings, no fences, no prose. It does not match the rest of the repo. If editing it, bring it into the dominant style.
- **`DotNet/` has no `_index.md`** while `ArchLinux/` and `OpcUa/` do. If adding one, mirror the table format used in `OpcUa/_index.md`.
- **File naming differs by directory.** `ArchLinux/` and `OpcUa/` use `kebab-case.md`; `DotNet/` uses a mix (`dotnet-cmd.md`, `IConfiguration.md`, `Templates.md`, `WPF-AppHost.md`). When adding new files, match the existing convention of the directory rather than introducing a third style.

## When adding content

- **New file in an existing directory:** follow the dominant style above, then add a row to `README.md` (and to the directory's `_index.md` if one exists).
- **New subject directory:** create `_index.md` inside it with a topics table, and add a new H2 section to `README.md` linking the files.
- **Editing existing content:** preserve the surrounding style — prose-first, then code block. Don't rewrite a guide's tone or restructure headings without reason.
