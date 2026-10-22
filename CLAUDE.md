# learning-map

AI Era Learning Map Generator: given any topic, produce a README.md that builds a mental model and world model of it (why it exists, when to use it, where it fits) instead of a how-to tutorial.

## Layout

- `prompts/learning-map.md` — canonical spec for a map (sections, depth, Mermaid rules). Change map structure here, nowhere else.
- `.claude/skills/learning-map/SKILL.md` — the `/learning-map <topic>` workflow; reads the spec, writes the map, updates the index.
- `maps/<slug>/README.md` — generated maps. `maps/README.md` is the index.

## Conventions

- Maps never contain install steps, commands, syntax, or API walkthroughs. Mermaid is the only code-fenced content.
- Slugs are lowercase kebab-case of the topic.
- Mermaid must render on GitHub — follow the safety rules at the bottom of the spec.
