# Learning Map Prompt (canonical)

> Single source of truth for what a learning map contains.
> The `/learning-map` skill reads this file; you can also paste it into any LLM
> after replacing `{{TOPIC}}`.

---

I want to understand **{{TOPIC}}**.

Assume AI can perform almost all operational tasks for me. Therefore, do not begin with installation, commands, syntax, APIs, menus, or step-by-step tutorials.

Your goal is not to teach me how to use it.

Your goal is to help me build an accurate **Mental Model** and **World Model**, so I understand:

- What it is
- Why it exists
- When I should think of using it
- How it changes thinking, decision-making, and work
- Where it fits within the larger ecosystem

Generate a well-structured, maintainable `README.md` with exactly the sections below, in this order.

## 1. Essence

- What is it fundamentally?
- Why does it exist?
- What problem does it solve?
- What did people do before it?
- What new possibilities does it create?

Focus on **Why**, not How. Open with a one-sentence definition a smart outsider would understand.

## 2. World Model

Describe:

- **Objects** — the things that exist in this world
- **Actors** — who or what acts on those objects
- **State changes** — how objects move between states
- **Workflows** — the recurring sequences of state changes
- **Systems** — the larger structures those workflows form
- **Value** — where value is created, captured, or destroyed

Include at least one System Map and one Lifecycle diagram.

## 3. Concept Map

Organize the domain into **2–3 conceptual layers** (e.g. Foundations → Mechanisms → Practice).

For each concept include: **Entity**, **Action**, **Purpose**, **Relationships**.
Use one table per layer, then one Concept Graph showing cross-layer relationships.

## 4. Decision Map

Instead of features, answer:

**Situation → Decision → Reason → Expected Outcome**

Give 8–12 realistic decisions as a table. Include at least one "don't use it" decision. Add a Decision Tree diagram for the central choice (usually: "should I reach for this at all, and in which form?").

## 5. Search Space Expansion

Help me discover what I don't yet know to ask:

- Beginner questions rarely asked
- Questions experts ask
- Questions worth exploring next
- Questions that define mastery

Expand the search space rather than merely answering it. 5–8 questions per group; for each, one line on *why the question matters* — do not answer it fully.

## 6. Ecosystem

Explain: **Upstream**, **Downstream**, **Alternatives**, **Complements**.
Include a Dependency Graph. For each alternative, state the trade-off that would make you choose it instead.

## 7. Transferable Principles

Separate:

- First principles
- Transferable methodologies
- Implementation details
- Stable knowledge (true in 10 years)
- Changing knowledge (likely different in 2 years)

Make clear which of these I should invest memory in and which I can delegate to AI.

## 8. Minimum Mental Model

If I could remember only **10–20 concepts**, which matter most?
Render them as a Mind Map, then a numbered list with one sentence each.

## 9. Common Misconceptions

For each (5–8 items): **Common misunderstanding**, **Why it happens**, **Better mental model**.

## 10. Summary

Complete:

> What I truly gain is not ________, but ________.

Follow with 3–5 sentences that compress the whole map.

---

## Output Requirements

- Assume AI performs the operational work. Focus on Mental Models, World Models, Decision Models, Search Space, and Concept Relationships — **not** commands or procedures. No code blocks except Mermaid.
- Use Mermaid whenever it clarifies. Prefer: Mind Maps, Concept Graphs, System Maps, Dependency Graphs, Decision Trees, Lifecycle Diagrams (`stateDiagram-v2`).
- Be specific to {{TOPIC}}. Every sentence should fail the test "could this be pasted into a map about a different topic unchanged?"
- Maintainable: stable headings, tables over prose where structure exists, no dated claims without a year.

### Document header

Start the file with:

```markdown
# {{TOPIC}} — Learning Map

> One-sentence definition.

**Generated:** YYYY-MM-DD · **Lens:** mental model, not manual

## Contents
(linked list of the 10 sections)
```

### Mermaid safety rules (so diagrams render on GitHub)

- Wrap any node label containing spaces or punctuation in double quotes: `A["Label (detail)"]`. Never put a raw `"` inside a label; use `'` instead.
- Node IDs are short alphanumerics (`A`, `ctx1`) — never reuse a keyword (`end`, `graph`, `class`) as an ID.
- `mindmap`: one root, indentation defines hierarchy, keep node text free of `()[]{}` characters, no quotes.
- `stateDiagram-v2`: state names without spaces; use `state "Long name" as S1` for readable names.
- Keep each diagram under ~25 nodes; split rather than cram.
