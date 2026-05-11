# Git & GitHub Bible

Cookbook-first reference for every Git and GitHub workflow I run. Safety rules → mental model → decision matrices → 21 scenario recipes → worktrees → multi-machine → `gh` CLI.

- **Live (styled HTML):** https://jayvee6.github.io/git-bible/
- **Source (markdown):** [`Git & GitHub Bible.md`](./Git%20%26%20GitHub%20Bible.md)

Built as a single self-contained HTML artifact in the spirit of Thariq Shihipar's [_The Unreasonable Effectiveness of HTML_](https://thariqs.github.io/html-effectiveness/). Theme: **sj-design** deep-space.

## Features

- Sticky table of contents with live filter — type to narrow scenarios + command rows; press `/` to focus, `Esc` to clear
- Copy-to-clipboard on every command block
- 21 scenarios collapsed by default — expand individually or all at once
- Glossary terms hover for inline definitions (HEAD, rebase, reflog, fast-forward, etc.)
- Self-contained: no JS frameworks, no font CDNs, ~82 KB

## Stack

- HTML + CSS variables + vanilla JS, single file
- sj-design **deep-space** theme tokens (cyan accent `#32D2F2`, radial gradient, two-layer CSS starfield, glass cards with inset highlights)
- Apple system font stack (`-apple-system`, SF Pro, SF Mono)

## Sections

1. Safety Rules — the never/always table
2. Mental Model — three trees + concept reference
3. Daily Golden Path — the 90% workflow
4. Decision Matrices — merge vs rebase, reset modes, force-push tree
5. Command Reference — grouped by purpose (12 sub-tables)
6. Scenario Cookbook — 21 "I'm in situation X → run Y" recipes
7. Worktree Workflows
8. Multi-Machine Workflows (Mac ↔ Win)
9. PR Workflow with `gh` CLI
10. Co-Authored Commits with Claude
11. Glossary & Authoritative Sources

## License

Personal reference, MIT-spirited. Steal what's useful.
