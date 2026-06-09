---
layout: project
title: "One-Research — a terminal UI for following AI research"
summary: "A keyboard-driven terminal app that pulls arXiv, HuggingFace daily papers, OpenReview, and research blogs into one feed, and reads the papers inline, LaTeX and all, without ever opening a browser."
status: "Active · open source (AGPL-3.0)"
permalink: /projects/one-research/
---

Status: Active · open source (AGPL-3.0). Working and self-hosted. [Source on GitHub](https://github.com/VictoryChianumba/one-research).

![One-Research terminal interface](/assets/images/one-research-thumbnail.png)

One-Research is a research-feed reader that lives entirely in the terminal. It aggregates arXiv (the full ~155-category taxonomy is browsable), HuggingFace daily papers, OpenReview, CORE, and arbitrary RSS/Atom feeds into a single keyboard-driven interface, then lets you triage what shows up — Inbox, Queued, Deep Read, Archived — with single keystrokes that persist across sessions. The thing I most wanted from it was to stop bouncing between twenty browser tabs and a PDF reader: One-Research opens papers *in place*, rendering LaTeX, math, tables, and figures into a vim-style reading pane with bookmarks, highlights, and per-paper notes.

It's built in Rust with no async runtime — plain OS threads and blocking I/O throughout. The cache and the workflow-state store are siblings that both load from disk at startup, so the UI paints instantly from the last session while network fetches run on a background thread and stream in over channels. Beyond the feed, it grows the features I actually use day to day: a fuzzy, relevance-ranked search with field scoping (`au:vaswani year:2017 attention`), an arXiv subject browser, AI-assisted source discovery, Semantic Scholar citation enrichment, an in-app chat pane (Claude or GPT) scoped to the selected paper, and runtime themes.

## What it does

- **Aggregates** arXiv, HuggingFace, OpenReview, CORE, and any RSS/Atom feed into one de-duplicated, date-sorted feed with persistent per-item workflow states.
- **Reads papers inline** — a terminal reader that renders LaTeX prose, math symbols, tables, and figures, with vim navigation, bookmarks, highlights, marks, and a command mode.
- **Searches well** — fuzzy, typo-tolerant, relevance-ranked (a title hit outranks an abstract hit) with field prefixes and year ranges, run off the UI thread.
- **Browses the firehose** — navigate arXiv's full taxonomy and promote any subject into your daily feed with one key, with scroll-tail pagination so the list deepens as you read.
- **Layers on tooling** — AI source discovery, citation enrichment, a GitHub repo viewer, per-paper notes, and an LLM chat pane scoped to the current item.

## Scope

~34k lines of Rust in the main binary (~45k across the workspace), single-developer. Unix-only (crossterm + Unix paths; untested on Windows). The terminal reader is factored into a sibling crate consumed as a path dependency.

## Links

- [Source: https://github.com/VictoryChianumba/one-research](https://github.com/VictoryChianumba/one-research)
