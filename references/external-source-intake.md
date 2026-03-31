# external-source-intake — Shared URL Intake Protocol

## When To Run
- A user provides a job, company, or LinkedIn URL
- A user asks the agent to pull a JD from LinkedIn or a job board
- A user provides interviewer LinkedIn URLs for prep

## Source Classification
- LinkedIn job URL
- LinkedIn profile URL
- aggregator job URL
- direct ATS or job-board URL
- company careers page URL
- unknown URL

## Resolution Rules
1. Prefer the direct ATS or board posting when already provided
2. Resolve aggregators or wrappers to the underlying posting when possible
3. If the first page is a thin shell, try the richer same-site posting before failing

## Retrieval Ladder
1. Public fetch
2. Authenticated browser fetch
3. User-pasted fallback

## Normalized Context
- `source_url`
- `resolved_url`
- `source_kind`
- `access_mode`
- `title`
- `company`
- `location`
- `body_text`
- `retrieval_confidence`
- `blockers_or_gaps`

## Fail Closed
- Do not treat teaser text or previews as a full JD
- If the page is blocked or too thin, say so directly
- If authenticated browsing is unavailable, ask for pasted content
- Never fabricate missing JD or profile context
