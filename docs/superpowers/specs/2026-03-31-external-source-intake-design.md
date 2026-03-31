# External Source Intake For Job And LinkedIn URLs — Design Spec

**Date:** 2026-03-31
**Status:** Approved

## Summary

Add prompt-native, read-only external source intake so any agent operating in this repo can fetch and use job descriptions or profile context from LinkedIn and job-board URLs instead of requiring pasted text first.

The capability stays at the prompt and workflow layer. This repo will not port the Python scraper/browser stack from `11-job-application-material-creation`. Instead, it will define a shared retrieval protocol that agents follow when they are given job, company, or LinkedIn URLs and when the runtime supports authenticated browsing.

## Goals

- Let `decode`, `prep`, `research`, and `linkedin` accept URLs as first-class inputs.
- Support authenticated, read-only retrieval for LinkedIn and gated job-board pages when the environment exposes browser or web access tied to the user's session.
- Normalize retrieved content into a consistent coaching input contract so downstream commands do not invent their own scraping behavior.
- Fail closed when a page is thin, blocked, or ambiguous instead of producing fabricated context from partial previews.
- Keep the implementation LLM-native and maintainable for a prompt-first repo.

## Non-Goals

- No application autofill, submission, saved-job import, or browser automation workflows.
- No board-specific Python port, scraper library port, or ATS API client in this repo.
- No attempt to guarantee retrieval in environments that do not expose authenticated browser or web tooling.
- No change to the output schemas of existing coaching commands.

## Why This Repo Needs A Different Shape

`11-job-application-material-creation` achieves high-fidelity extraction through code: URL resolvers, browser sessions, board-specific parsers, and fallback fetchers. That is the right abstraction there because the repo owns deterministic pipelines and browser automation.

This repo is different. Its core product is the prompt and command system. The right move here is to bring over the capability boundary, not the implementation stack:

- shared URL intake behavior
- shared source-quality rules
- shared fallback rules
- shared normalized context contract

That gives agents the same practical ability to work from LinkedIn and job-board sources without turning this repo into a second board-automation codebase.

## Design Overview

Introduce a global **External Source Intake** layer that runs before normal command logic whenever the user:

- supplies a job, company, or LinkedIn URL
- asks the agent to fetch a JD from LinkedIn or a job board
- asks for interviewer research from LinkedIn URLs
- references a role or company page without pasting the content

The intake layer classifies the source, resolves it to the best readable page, retrieves content through a strict ladder, normalizes the result, and then hands the normalized context into the target command.

## Shared Intake Contract

Every successful retrieval produces a compact normalized context object conceptually shaped like this:

```text
source_url
resolved_url
source_kind
access_mode
title
company
location
body_text
retrieval_confidence
blockers_or_gaps
```

### Field Definitions

- `source_url`: the original user-provided URL
- `resolved_url`: the highest-fidelity readable URL reached after resolution
- `source_kind`: one of `job_posting`, `company_page`, `interviewer_profile`, `candidate_profile`, or `unknown`
- `access_mode`: one of `public`, `authenticated_browser`, or `user_pasted`
- `title`: job title or profile headline/title when available
- `company`: employer or current company when available
- `location`: job or profile location when available
- `body_text`: the extracted text actually used by the coaching workflow
- `retrieval_confidence`: `high`, `medium`, or `low`
- `blockers_or_gaps`: explicit missing data, page blockers, or ambiguity notes

Downstream commands should treat this normalized context as the source of truth for fetched material.

## Source Classification

The intake layer classifies URLs into one of these buckets:

- LinkedIn job URL
- LinkedIn profile URL
- aggregator job URL
- direct ATS or job-board URL
- company careers page URL
- unknown URL

This classification exists to pick the right retrieval path, not to expose board-specific complexity inside each command.

## Resolution Rules

Before retrieval, the agent should resolve the URL to the best readable page available.

### Resolution Priorities

1. If the input is already a direct ATS or board posting, use it.
2. If the input is an aggregator or wrapper page, attempt to resolve to the underlying direct posting.
3. If the first page is a thin application shell, try the fuller same-site posting before failing.
4. If multiple resolved URLs refer to the same role, prefer the highest-fidelity source in this order:
   - direct ATS posting
   - resolved company-hosted posting
   - authenticated LinkedIn page
   - aggregator preview

The intake layer should never treat teaser text, preview cards, or application-shell fragments as a complete JD when a richer page is available or required.

## Retrieval Ladder

After classification and resolution, retrieval follows a strict ladder:

1. **Public fetch**
   Use a public page fetch when the page contains enough readable content.
2. **Authenticated browser fetch**
   Use the environment's authenticated browser or web session for LinkedIn or gated job-board pages when available.
3. **User-provided pasted text**
   Ask the user for pasted content only if the first two paths fail or still produce a thin or blocked source.

This ladder preserves the prompt-first design while still allowing authenticated LinkedIn and job-board reads when the runtime supports them.

## Authenticated Read-Only Retrieval

This design assumes authenticated retrieval is read-only and bounded:

- agents may open and read LinkedIn or job-board pages through an authenticated session
- agents may extract visible professional context from those pages
- agents may not edit profiles, click submit flows, import saved jobs, or automate application actions

If the environment does not expose authenticated browsing or the session is unavailable, the agent must say that directly and fall back to asking for pasted content. It must not imply that LinkedIn access exists when it does not.

## Failure Semantics

The new capability must be conservative by default.

### Hard Rules

- If a page is readable and complete enough, use it.
- If a page is only a thin shell, teaser, login wall, or partial preview, explicitly say the source is insufficient.
- If authenticated browsing is required but unavailable, say so directly.
- If retrieval yields ambiguous or partial role data, ask for the missing content instead of inferring.
- Never fabricate a JD or interviewer profile from fragments, memory, or generic knowledge.

### Confidence Rules

- `high`: direct readable posting or profile with sufficient body text
- `medium`: resolved source is usable but some metadata is thin or inferred from adjacent page structure
- `low`: partial retrieval with meaningful blockers; only use for limited context, never as a full JD substitute

## Command Integration

The intake layer is shared. Commands consume it rather than reimplementing it.

### `decode`

- Accept pasted JD text, LinkedIn job URLs, and supported job-board URLs.
- If a URL is provided, fetch and normalize the JD first, then run the existing decode workflow.
- Preserve the existing decode schema and confidence labeling; only the intake path changes.

### `prep [company]`

- Accept JD URLs and interviewer LinkedIn URLs directly.
- Fetch the JD and interviewer context before competency parsing or interviewer intelligence.
- Preserve the current prep schema and sourcing tiers.

### `research [company]`

- Allow company careers pages and job-board or company-hosted role pages to count as Tier 1 verified inputs.
- Use fetched company or job-page context as session evidence alongside normal research.

### `linkedin`

- Accept LinkedIn profile URLs directly when the environment supports browsing.
- Use the fetched profile text as the audit input instead of requiring pasted sections first.
- Apply the same read-only rule to interviewer-profile use inside `prep`.

### Global Routing Rule

If the user says things like:

- "look at this role"
- "pull the JD from here"
- "check this LinkedIn job"
- "research this interviewer"

the agent should route through External Source Intake before deciding whether the request becomes `decode`, `prep`, `research`, or `linkedin`.

## Prompt Packaging

This repo currently functions as a prompt-first project. The canonical change should land in `SKILL.md`, then be mirrored into provider-facing prompt files.

### Canonical Prompt Files

- `SKILL.md`: canonical skill source
- `CLAUDE.md`: Claude-facing mirror
- `AGENTS.md`: Codex-facing mirror to be added or regenerated during implementation

The design assumes provider-facing files stay behaviorally aligned so "any agent in this project" means the same external source rules regardless of entrypoint.

## Command Reference Files To Update

- `references/commands/decode.md`
- `references/commands/prep.md`
- `references/commands/research.md`
- `references/commands/linkedin.md`
- `references/commands/help.md`

These updates should remove the current assumption that LinkedIn cannot be browsed directly and replace it with capability-sensitive wording:

- use fetched URLs when available
- fall back to pasted content when browsing is unavailable or insufficient
- keep failure behavior explicit and fail-closed

## Verification Requirements

Implementation is only complete when all of the following are true:

1. The canonical prompt no longer states that LinkedIn cannot be browsed directly.
2. Each affected command accepts URLs as first-class inputs.
3. The failure path is explicit and conservative.
4. Provider-facing prompt files remain in sync.
5. The `help` command communicates that URL-based intake is supported for the relevant commands.

## Testing Strategy

This repo does not need board-parser tests. It needs workflow-level verification.

### Manual Verification Cases

- `decode` with a public ATS URL
- `decode` with a LinkedIn job URL that requires authenticated reading
- `prep` with a JD URL and interviewer LinkedIn URL
- `research` with a company careers page URL
- `linkedin` with a LinkedIn profile URL
- blocked or thin-shell URL that must fail closed
- environment without authenticated browsing, where the agent must fall back to asking for pasted content

### Expected Outcomes

- the agent chooses the correct intake path
- the agent surfaces the resolved source it actually used
- the agent does not treat thin previews as full JDs
- the agent does not claim access it does not have
- downstream command output keeps its existing structure

## Risks

### Environment Variability

Authenticated LinkedIn access depends on the host agent runtime exposing browser or web tools and a live session. This capability cannot be guaranteed from prompt text alone.

**Mitigation:** make the prompt explicit that authenticated browsing is conditional, then fail closed to pasted content when the capability is absent.

### False Confidence From Thin Sources

Aggregators and wrapper pages often expose incomplete text that looks plausible enough to fool an agent into continuing.

**Mitigation:** explicitly forbid treating teaser text or thin shells as complete source material.

### Provider Drift

If only one provider-facing prompt file is updated, the repo will present different behavior depending on entrypoint.

**Mitigation:** keep `SKILL.md` canonical and mirror into `CLAUDE.md` and `AGENTS.md` during implementation.

## Decisions

- Chosen approach: prompt-native external source intake
- Auth scope: authenticated, read-only retrieval is allowed
- Porting decision: do not port the scraper and browser codebase from `11-job-application-material-creation`
- Command scope: `decode`, `prep`, `research`, `linkedin`, and `help`
- Failure posture: fail closed, never synthesize missing JD or profile content

## Implementation Boundary

This design intentionally stops at the spec layer. The follow-up implementation plan should cover:

- exact prompt edits in `SKILL.md`
- provider mirror generation strategy
- precise wording changes in each command reference
- verification steps for URL intake and fail-closed behavior

