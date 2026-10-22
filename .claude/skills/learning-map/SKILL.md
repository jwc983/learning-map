---
name: learning-map
description: Generate an AI-era learning map (mental model + world model README.md) for any tool, technology, subject, methodology, or domain. Use when the user runs /learning-map <topic> or asks to "map", "build a mental model of", or "understand" a topic rather than learn how to operate it.
argument-hint: <topic> [--lang <language>] [--audience <who>]
---

# Learning Map Generator

Turn any topic into `maps/<slug>/README.md`: a map of *why* and *where*, not a tutorial of *how*.

## Inputs

- `$ARGUMENTS` — the topic, free text (e.g. `Kubernetes`, `double-entry bookkeeping`, `Jobs to be Done`).
- Optional `--lang <language>`: write the map in that language (headings included). Default: the language the user wrote the topic in.
- Optional `--audience <who>`: tune examples to that reader (e.g. `product manager`). Default: smart generalist.

If `$ARGUMENTS` is empty, ask for the topic and stop.

## Steps

1. **Load the spec.** Read `prompts/learning-map.md` (repo root). It is the single source of truth for sections, depth, and Mermaid rules. Substitute `{{TOPIC}}` with the topic.
2. **Resolve the topic.** If it is ambiguous (e.g. `Spark`, `Rust`, `Mercury`), pick the most likely meaning from context and state the interpretation in the definition line; ask only if there is no reasonable default.
3. **Check freshness.** If the topic changes fast (AI tools, frameworks, regulations) and web search is available, run 1–2 searches to confirm its current shape before writing. Put dated claims under *Changing knowledge* with a year.
4. **Pick the slug.** Lowercase kebab-case of the topic (`Model Context Protocol` → `model-context-protocol`). If `maps/<slug>/README.md` already exists, read it and ask whether to overwrite or write `README.v2.md`.
5. **Write** `maps/<slug>/README.md` following the spec exactly: document header, all 10 sections in order, required diagrams.
6. **Self-review before finishing.** Re-read the file and fix:
   - Any section missing or out of order, or a section that drifts into commands, syntax, or setup steps.
   - Generic sentences that would fit any topic.
   - Mermaid that violates the safety rules in the spec (unquoted labels with punctuation, reserved IDs, special characters in `mindmap` nodes).
7. **Update the index.** Add or update one row in `maps/README.md`: `| [Topic](slug/README.md) | one-sentence definition | YYYY-MM-DD |`, keeping rows sorted alphabetically.
8. **Report** in 2–3 lines: the file path, the "What I truly gain is not ___, but ___" sentence, and one question from *Questions that define mastery* worth exploring next.
