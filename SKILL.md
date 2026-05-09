---
name: agent-activation-smoke-test
description: "Structured pre-flight checklist for verifying a new Paperclip agent or company skill before it goes live. Use this skill whenever you are asked to smoke-test a newly hired or activated agent, run a first-heartbeat verification, confirm an agent's API access is healthy, or sign off on a company skill deployment before it is assigned to production agents. Trigger on any QA issue that mentions \"activation\", \"smoke test\", \"pre-flight\", \"first heartbeat\", \"go-live check\", or \"skill deployment QA\". Also trigger when a newly-hired agent's readiness must be confirmed before the board or manager assigns it real work."
---

# Agent Activation Smoke Test

You are running a structured pre-flight check. Your job is to verify that the subject (a new agent or a new
company skill) is safe to put into production. Work through the relevant checklist below, record a verdict for
each check, and emit a structured verdict comment at the end.

Never skip a check without explaining why it does not apply. A skipped check is not a pass.

---

## Part A — New Agent Activation

Run this part when a newly hired or recently configured agent must be verified before the board assigns it
real issues.

### A1. Identity check

Call `GET /api/agents/me` **as the subject agent** (or read the agent record as yourself via
`GET /api/companies/{companyId}/agents/{agentId}`). Verify:

- `id` matches the expected agent ID
- `companyId` matches the company
- `role` is the role the agent was hired for
- No unexpected `status` flags (e.g. paused, suspended)

**Pass condition:** all four fields match the hire spec. Any mismatch is a FAIL.

### A2. Inbox access

Call `GET /api/agents/{agentId}/skills` and `GET /api/agents/me/inbox-lite` (or the equivalent on the
subject agent's behalf). Verify:

- The endpoint returns HTTP 200
- The response is a valid JSON object (not an error body)
- The `assignments` array (or equivalent field) is present — it may be empty on day 1; that is fine

**Pass condition:** 200 with valid inbox schema. A 401/403/500 is a FAIL.

### A3. Checkout smoke

Assign a low-priority test issue (create one if none exists) to the subject agent. Then from the subject
agent's context, run:

```
POST /api/issues/{testIssueId}/checkout
{ "agentId": "{subjectAgentId}", "expectedStatuses": ["todo"] }
```

Verify:

- Returns 200 (not 409 or 422)
- The issue `status` is now `in_progress` and `assigneeAgentId` matches the subject

Clean up after: release the issue (`POST /api/issues/{testIssueId}/release`) and set it back to `todo` or
`cancelled`.

**Pass condition:** clean 200 checkout + 200 release. A 409 or failure to release is a CONDITIONAL PASS —
document it and create a follow-up issue for the locking bug before go-live.

### A4. Skill load check

If the subject agent has assigned skills, for each skill:

1. Confirm the skill name appears in the company skill library (`GET /api/companies/{companyId}/skills`)
2. Confirm the agent skill list (`GET /api/agents/{agentId}/skills`) includes it
3. In a test heartbeat or local invocation, invoke the skill by name and confirm it loads without error

**Pass condition:** every assigned skill resolves and loads. An unresolvable skill is a FAIL — the agent will
silently lack guidance on those workflows.

### A5. First heartbeat output review

Read the agent's most recent run log (via the Paperclip runs API or run output). Look for:

- Unexpected 401/403 errors (env var / API key problem)
- Missing required env vars (search for "undefined", "missing", "not set" in the log)
- Tool permission denials (search for "denied", "not allowed", "permission")
- Unhandled exceptions or stack traces

If no heartbeat has run yet, trigger one (assign and immediately release a test issue) and review the
resulting run log.

**Pass condition:** no unexpected errors. Known-benign warnings (e.g., "inbox empty on first run") are fine;
document them but do not count them as failures.

---

## Part B — Skill Deployment (No New Agent)

Run this part when a company skill is being deployed and must pass QA before being assigned to production
agents.

### B1. Skill resolution

Confirm the skill is installed and readable:

```
GET /api/companies/{companyId}/skills
GET /api/companies/{companyId}/skills/{skillId}/files?path=SKILL.md
```

Verify:

- The skill appears in the company library with the expected name/slug
- `SKILL.md` is readable and non-empty
- The description field (YAML frontmatter) is present and non-trivial

**Pass condition:** skill is installed, readable, and has a description. Missing SKILL.md is a FAIL.

### B2. Step coverage walk-through

Pick a representative issue from the company issue tracker (a real past issue or a synthetic scenario).
Walk through the skill's documented steps against that issue scenario. For each step ask:

- Is the instruction unambiguous? (Would two different agents interpret it the same way?)
- Does it reference any tool, endpoint, or resource that might not exist?
- Is there a gap — something the skill expects the agent to know but does not state?

Document any ambiguous step with a short description of the ambiguity.

**Pass condition:** zero ambiguous steps → PASS. One or two minor ambiguities → CONDITIONAL PASS with
follow-up issues. Fundamental gaps in the core workflow → FAIL.

### B3. Existing-skill conflict check

Read the descriptions and key instructions of all currently assigned skills for the target agent(s).
Check whether any step in the new skill contradicts or overrides a step in an existing skill.

Specific conflict patterns to watch for:

- Two skills that give contradictory rules for the same trigger phrase
- A new skill that silently supersedes a step in an existing skill without acknowledging it
- A new skill that duplicates an existing skill's core workflow (use-case overlap rather than a true conflict)

**Pass condition:** no contradictions found → PASS. Overlap without conflict → CONDITIONAL PASS, document
and link the related skill. Outright contradiction → FAIL, must resolve before assignment.

---

## Verdict

After completing all applicable checks, emit a structured verdict comment on the activation issue:

```
## Smoke Test Verdict: [PASS | CONDITIONAL PASS | FAIL]

**Subject:** [agent name/ID or skill name]
**Checks run:** [list each check ID and pass/fail/skip]

### Summary
[1–3 sentences describing overall health]

### Conditions (if CONDITIONAL PASS)
[Bullet list of open items, each with a linked follow-up issue if created]

### Blockers (if FAIL)
[Bullet list of what must be fixed before go-live, with owner/action]

**Recommendation:** [Approve go-live | Block until conditions resolved | Block pending fix]
```

A **PASS** or **CONDITIONAL PASS** (with all follow-up issues filed and linked) clears the agent or skill
for production assignment. A **FAIL** blocks go-live — update the activation issue to `blocked` with the
first blocker as `blockedByIssueIds` and notify the manager.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
