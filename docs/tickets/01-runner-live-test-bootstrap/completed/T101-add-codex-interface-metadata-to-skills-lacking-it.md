# T101 — Add Codex Interface Metadata to Catalog Skills Currently Missing It
**Type:** wayfinder:task (AFK)
Status: completed
Completed: 2026-10-27T14:47:17Z
Spec: docs/specs/01-runner-live-test-bootstrap.md
Blocked by: none
Reasoning: medium

### Question

Add an `agents/openai.yaml` interface-metadata file to each of the eight
catalog skills currently missing it, so the catalog surfaces uniformly in the
OpenAI Codex app skill picker.

### Why this matters

`agents/openai.yaml` is the surface the OpenAI Codex app reads to register a
skill in its picker (display name, short description, optional `policy`
block for skills that should not be auto-invoked). Of the 45 skills in the
catalog at chart time, eight are missing the file:

- `clean-architecture`
- `create-verification-skill`
- `live-qa`
- `maintain-verification-skill`
- `security-review`
- `smoke-fail`
- `software-architecture`
- `status`

That gap means the Codex app either omits these skills from its picker or
falls back to ad-hoc heuristics on the frontmatter description, neither of
which the catalog promises. The README's "Skill Directory Anatomy" section
already calls out `scripts/` and `references/` as optional siblings of
`SKILL.md`; the `agents/` interface is the third and is silently inconsistent
across the eight skills above.

### Requirements

- For each of the eight skills listed above, create
  `<skill>/agents/openai.yaml` with at minimum:
  ```yaml
  interface:
    display_name: "<Title-cased skill name>"
    short_description: "<≤80-char second-person imperative derived from the SKILL.md frontmatter description>"
  ```
- Use the existing `agents/openai.yaml` files as the format reference — match
  key order (`interface:` then optional `policy:`), string-quoting style
  (double-quoted values), and indentation (two spaces). Do not introduce a
  new schema.
- For skills whose `SKILL.md` frontmatter already carries
  `disable-model-invocation: true`, also add a matching
  `policy.allow_implicit_invocation: false` block (see the existing
  `grill-me/agents/openai.yaml` and `handoff/agents/openai.yaml` for the
  pattern). Audit each skill's frontmatter before authoring.
- Keep the `short_description` distinct from the README table description:
  the Codex short description is user-prompted into the picker UI and should
  be an imperative ("Review a diff on standards and spec") rather than the
  catalog's descriptive prose ("Review the changes since a fixed point …
  along two axes: Standards … and Spec …").
- Do NOT modify any existing `agents/openai.yaml` files. Eight new files
  only.
- Do NOT touch `SKILL.md`, `references/`, or `scripts/` for any of the
  eight skills; this ticket is interface-metadata only.
- Do NOT execute this ticket in this commit; T017 only seeds the queue.
  Execution is the user's decision in a subsequent session.

### Acceptance Criteria

- All eight skills have a new `agents/openai.yaml` file matching the
  existing schema (parseable as YAML, key order matches the reference
  cluster).
- Each new `short_description` is ≤80 characters, imperative, and distinct
  from the corresponding README table description.
- Skills with `disable-model-invocation: true` in their `SKILL.md`
  frontmatter also carry `policy.allow_implicit_invocation: false`.
- All 45 catalog skills now have an `agents/openai.yaml` file (verified
  via `(Get-ChildItem -Recurse -Filter openai.yaml).Count` returning 45).
- Existing `agents/openai.yaml` files are unchanged (`git diff --stat
  <repo>` shows only additions, no modifications to existing
  `agents/openai.yaml` files).

### Smoke Scenarios

**Scenario: All catalog skills uniformly ship the Codex interface metadata**

- Setup: After execution, `D:\Repos\agent-skills` is the Target Project and
  the eight skills are present with new `agents/openai.yaml` files.
- Why: Confirm the catalog reaches uniform interface coverage end to end.
- Steps:
  1. From `D:\Repos\agent-skills`, run a PowerShell one-liner counting
     `agents/openai.yaml` files (e.g.
     `(Get-ChildItem -Recurse -Filter openai.yaml).Count`); confirm the
     result is `45`.
  2. Pick any one of the eight touched skills and run
     `Get-Content <skill>/agents/openai.yaml`; confirm the YAML parses
     (no unquoted special chars, key order matches the reference).
  3. Run `git diff --stat <repo>` and confirm the diff contains exactly
     eight new file paths and zero modifications to existing
     `agents/openai.yaml`.
- Expected: All three steps succeed.

### Gotchas

- The eight skills were identified by walking the catalog at chart time
  (`b72cdf8` HEAD); re-verify the list inside this ticket's session
  before authoring, in case the catalog state has shifted.
- `live-qa` already declares `disable-model-invocation: true` in its
  frontmatter — that skill should also receive a `policy` block. Other
  skills in the eight may or may not; check each individually.
- Do not introduce a `version:` field, `agents:` array, or any other
  schema extension — the existing 37 files use only `interface:` and
  optional `policy:`. Stay within the established schema.
- The chart-time observation was made before this ticket was authored; if
  the upstream has since added the metadata to any of the eight, drop
  that skill from the work list rather than overwriting.
- The `agents/openai.yaml` file is read by the OpenAI Codex app, not by
  Ticket Runner itself; this is purely a catalog-completeness fix, not a
  Runner-feature change.