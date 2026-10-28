# AGENTS.md

Agent Skills Catalog. A distribution repo of agent skills: each skill is a directory of Markdown
and YAML. There is no application code, no build step, and no test suite of its own.

## Repository shape

- `README.md` — the canonical onboarding document: Overview, Installation & Synchronization,
  Canonical Skills Index, Skill Directory Anatomy, Contributing.
- `<skill-name>/` — one directory per skill, 45 of them, each listed in the README's
  Canonical Skills Index table.
- `docs/specs/`, `docs/tickets/` — ephemeral planning scaffolding for the active work queue.
- `.agent/`, `.agents/`, `observations/` — runtime state; never commit, never hand-edit.

## Skill directory contract

```text
<skill-name>/
├── SKILL.md            # Instruction manual with YAML frontmatter (name, description)
├── scripts/            # (Optional) Deterministic automation or validation scripts
├── references/         # (Optional) Deep architectural references, schemas, or templates
└── agents/             # (Optional) Harness-specific interface metadata
    └── openai.yaml     # Codex app display name, short description, optional policy
```

- `SKILL.md` frontmatter carries `name` and `description`; keep both valid YAML.
- Keep instructions actionable, deterministic, and test-first.

## Interface metadata (`agents/openai.yaml`)

- Every skill ships `agents/openai.yaml`. Author it with `interface.display_name` and
  `interface.short_description`; derive the description from the skill's frontmatter and keep it
  under 80 characters, second-person imperative, and distinct from the README table prose.
- Skills whose `SKILL.md` frontmatter declares `disable-model-invocation: true` also carry
  `policy.allow_implicit_invocation: false`.
- Match the established schema exactly: key order `interface:` then optional `policy:`,
  two-space indentation, double-quoted values. Do not introduce `version:`, an `agents:` array, or
  any other extension.
- Never modify an existing `agents/openai.yaml` as a side effect of unrelated work. Treat existing
  files as the format reference and leave them byte-identical.

## Verification

- `python -c "import glob,yaml; fs=glob.glob('*/agents/openai.yaml'); [yaml.safe_load(open(f,encoding='utf-8')) for f in fs]; print('validated %d openai.yaml files' % len(fs))"`
  run from the repository root is the whole verification contract. It must exit 0 and its count
  must match the number of interface files the change intends to produce.
- Verify catalog coverage with a recursive count of `openai.yaml` files, not by trusting a summary.

## Discipline

- Work on branch `agent/ticket-runner`. Never `git push`; a pre-push guardrail blocks it.
- Only the Gatekeeper commits, and only after independent verification passes and the human
  approves the Evidence Card. Never commit directly, and never author a verification log.
- One commit per ticket, combining the change, the relocated ticket file (`Status: completed`), and
  any gotchas logged in that session.
- Commit titles follow `<type>(<scope>): <Title>` with no ticket number, followed by a blank line
  and hyphen-bulleted imperative changes without trailing periods.
- Record work-in-progress rules and recurring failure modes in `docs/tickets/gotchas.md`.
- Keep the README canonical count claim true: T101 brings the catalog to uniform interface
  coverage; T102 documents it so future contributions do not regress.
