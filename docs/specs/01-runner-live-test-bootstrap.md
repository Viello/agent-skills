# 01 — Runner Live-Test Bootstrap

This spec seeds the agent-skills driver with a small starter ticket queue so the
end-to-end Ticket Runner workflow (worker → gatekeeper → commit) can be exercised
against the real `https://github.com/Viello/agent-skills` upstream without waiting
for organic work to appear; the two demo tickets describe real catalog hygiene
gaps observed on the live clone at chart-time — eight skills missing Codex
interface metadata (`agents/openai.yaml`) and a `README.md` "Skill Directory
Anatomy" section that does not document the `agents/` interface files — and
together exercise both the file-add and in-place-modify code paths the driver
must support; the queue lives at `docs/tickets/01-runner-live-test-bootstrap/`
inside the driver, and because the bootstrap script hard-codes
`--project-dir D:\Repos\agent-skills`, the Ticket Runner source itself never
reads the driver's queue (reconciling the prior cluster-layout discipline that
excluded `docs/tickets/` from the driver), so this scaffold is the only
non-runtime file the driver carries that overlaps the runner's read paths in
name only, not in interpretation.