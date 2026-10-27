# T102 — Document `agents/openai.yaml` in the Catalog's Skill Directory Anatomy
**Type:** wayfinder:task (AFK)
Status: completed
Completed: 2026-10-27T14:51:44Z
Spec: docs/specs/01-runner-live-test-bootstrap.md
Blocked by: T101
Reasoning: medium

### Question

Update the agent-skills README's "Skill Directory Anatomy" section to document
`agents/openai.yaml` as part of the standard catalog structure, then update the
"Contributing" guidance to mention that new skills should ship the interface
file by default.

### Why this matters

The catalog ships an `agents/openai.yaml` next to most `SKILL.md` files
(45 of 45 once T101 lands), but the README — the canonical onboarding
document for both humans and agents discovering the catalog — does not
describe that file anywhere. A new contributor copying a skill template
will copy `SKILL.md`, `scripts/`, and `references/` (because the Anatomy
section lists them), but won't know to ship `agents/openai.yaml` unless
they go hunting in an existing skill. The result is the same gap that
left eight catalog skills without the file at chart time.

This ticket closes that documentation gap so future contributions do not
regress the uniform interface coverage that T101 brings the catalog to.

### Requirements

- Modify `D:\Repos\agent-skills\README.md` in place:
  - Extend the directory-tree listing under "Skill Directory Anatomy" to
    include the `agents/` interface alongside `SKILL.md`, `scripts/`, and
    `references/`. The current listing is:
    ```text
    <skill-name>/
    ├── SKILL.md            # Instruction manual with YAML frontmatter (name, description)
    ├── scripts/            # (Optional) Deterministic automation or validation scripts
    └── references/         # (Optional) Deep architectural references, schemas, or templates
    ```
    Update to:
    ```text
    <skill-name>/
    ├── SKILL.md            # Instruction manual with YAML frontmatter (name, description)
    ├── scripts/            # (Optional) Deterministic automation or validation scripts
    ├── references/         # (Optional) Deep architectural references, schemas, or templates
    └── agents/             # (Optional) Harness-specific interface metadata
        └── openai.yaml     # Codex app display name, short description, optional policy
    ```
  - In the "Contributing" section, add a bullet to the existing list
    stating that new skills must include `agents/openai.yaml` unless the
    skill is intentionally in-agent-only with documented rationale.
- Do NOT add new top-level sections; the change is anatomy + contributing
  only.
- Do NOT touch the "Canonical Skills Index" table; the count and ordering
  remain governed by the catalog itself, not this ticket.
- Do NOT touch the "Installation & Synchronization" section; that block
  documents Ticket Runner behavior, not per-skill structure.
- Do NOT execute this ticket in this commit; T017 only seeds the queue.

### Acceptance Criteria

- `D:\Repos\agent-skills\README.md` "Skill Directory Anatomy" code block
  includes the new `agents/openai.yaml` line, indented as a child of the
  `agents/` directory in the listing.
- The "Contributing" section gains a bullet stating the
  `agents/openai.yaml` requirement with the documented-exception carve-out.
- The README's existing "45 canonical skills" claim is preserved
  (T101's catalog-coverage change does not affect the count).
- The diff against the previous README contains only the anatomy-line edit
  and the new contributing bullet; no other section changes.

### Smoke Scenarios

**Scenario: Catalog anatomy now reflects the Codex interface file**

- Setup: After execution, `D:\Repos\agent-skills\README.md` is the
  modified file.
- Why: Confirm the documentation matches the catalog's actual shape.
- Steps:
  1. From `D:\Repos\agent-skills`, run `Get-Content README.md | Select
     -String 'openai.yaml'` and confirm both the anatomy line and the
     contributing bullet appear.
  2. Open `README.md` and visually verify the anatomy tree shows the new
     `agents/openai.yaml` child entry with the right indentation.
  3. Run `git diff README.md` and confirm the diff touches only the
     anatomy block and the contributing list — no other section changes.
- Expected: All three steps succeed.

### Gotchas

- This ticket BLOCKS on T101 because T101 establishes uniform coverage in the
  data; documenting the file before T101 lands would risk documenting a
  half-coverage catalog. Update if any; the workflow expectation is
  T101 → T102.
- The README's "Skill Directory Anatomy" code-block uses tree-drawing
  characters (`├──`, `└──`). The new `agents/` entry must use the same
  tree-drawing characters, not ASCII pipe characters (`|--`, `` `-- ``).
  Match the existing style exactly.
- The `agents/openai.yaml` description in the anatomy block should be
  brief (one line, ≤80 chars) — same brevity as the `scripts/` and
  `references/` lines. Do not editorialize about the schema.
- The contributing bullet must use the same markdown style as the
  existing three bullets (single hyphen, short imperative). Read the
  existing bullets before authoring the new one.
- This ticket modifies an existing file in the live clone; the Gatekeeper
  commit lands against `D:\Repos\agent-skills`'s `agent/ticket-runner`
  branch. The hook on `D:\Projects` does NOT govern the clone.