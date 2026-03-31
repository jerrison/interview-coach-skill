# External Source Intake Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add prompt-native, authenticated read-only URL intake so this interview-coach repo can fetch job, company, and LinkedIn context directly from supported URLs before falling back to pasted text.

**Architecture:** Create one shared `references/external-source-intake.md` protocol, wire it into the canonical `SKILL.md` prompt plus provider mirrors, then update the affected command docs to consume the shared intake layer. Keep verification prompt-level: shell consistency checks plus manual scenario drills, with no scraper-code port from the job-application repo.

**Tech Stack:** Markdown prompt files, repo docs, shell verification (`test`, `rg`, `diff`), git

---

## File Structure

- Create: `references/external-source-intake.md`
  Responsibility: shared URL classification, resolution rules, retrieval ladder, normalized context contract, fail-closed behavior.
- Create: `AGENTS.md`
  Responsibility: Codex-facing mirror of the canonical prompt.
- Modify: `SKILL.md`
  Responsibility: canonical prompt source; restore it in the worktree if still deleted.
- Modify: `CLAUDE.md`
  Responsibility: Claude-facing prompt mirror.
- Modify: `references/commands/decode.md`
  Responsibility: accept job URLs as first-class inputs and run external intake before JD analysis.
- Modify: `references/commands/prep.md`
  Responsibility: accept JD and interviewer LinkedIn URLs; remove the hard prohibition on direct LinkedIn browsing.
- Modify: `references/commands/research.md`
  Responsibility: treat provided company or role URLs as Tier 1 session evidence.
- Modify: `references/commands/linkedin.md`
  Responsibility: accept LinkedIn profile URLs directly; use capability-sensitive fallback wording.
- Modify: `references/commands/help.md`
  Responsibility: surface URL-based intake in relevant command descriptions.
- Modify: `README.md`
  Responsibility: document URL-based intake and the repo's provider-facing prompt files.

No automated test harness exists in this repo today. Verification in this plan uses exact shell checks plus manual prompt drills.

### Task 1: Add The Shared External Intake Reference

**Files:**
- Create: `references/external-source-intake.md`
- Test: shell verification only; no test file exists in this repo

- [ ] **Step 1: Verify the shared intake reference does not exist yet**

Run:

```bash
test ! -f references/external-source-intake.md && echo "missing"
```

Expected: `missing`

- [ ] **Step 2: Create `references/external-source-intake.md` with the shared protocol**

Write this file with these exact sections and starter content:

```md
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
```

- [ ] **Step 3: Verify the new reference contains the required sections**

Run:

```bash
rg -n "When To Run|Source Classification|Retrieval Ladder|Normalized Context|Fail Closed" references/external-source-intake.md
```

Expected: five matches, one for each required section heading

- [ ] **Step 4: Commit the reference file**

Run:

```bash
command git add references/external-source-intake.md
command git commit -m "docs: add external source intake reference"
```

Expected: a new commit containing only `references/external-source-intake.md`

### Task 2: Restore And Update The Canonical Prompt And Claude Mirror

**Files:**
- Modify: `SKILL.md`
- Modify: `CLAUDE.md`
- Test: shell verification only; no test file exists in this repo

- [ ] **Step 1: Confirm the current worktree is missing the canonical `SKILL.md` and lacks the new intake rules**

Run:

```bash
test ! -f SKILL.md && echo "missing-skill"
rg -n "External Source Intake|references/external-source-intake.md|authenticated_browser" CLAUDE.md SKILL.md 2>/dev/null || true
```

Expected: `missing-skill`, plus no matches for the new intake rules

- [ ] **Step 2: Restore `SKILL.md` from the active Claude mirror if it is still missing**

Run:

```bash
cp CLAUDE.md SKILL.md
```

Expected: `SKILL.md` exists again and starts as an exact copy of `CLAUDE.md`

- [ ] **Step 3: Add the shared intake rules to `SKILL.md`**

Insert this exact block between the Non-Negotiable Operating Rules section and the Command Registry:

```md
## External Source Intake

When a user provides a job, company, or LinkedIn URL, or asks the agent to fetch content from one, run `references/external-source-intake.md` before asking for pasted text.

- Use fetched URLs when the environment supports browsing or web retrieval.
- Prefer the highest-fidelity readable source after resolution.
- Treat authenticated browsing as read-only.
- If the source is thin, blocked, or ambiguous, fail closed and ask for the missing content.
- Do not fabricate JD or profile context from previews or memory.
```

Update the File Routing section so it includes this exact bullet:

```md
- **`decode`**, **`prep`**, **`research`**, **`linkedin`**, **`help`**: Also read `references/external-source-intake.md` when URL-based source intake is in play.
```

Add this exact sentence immediately above the Mode Detection Priority list:

```md
When a request includes a job, company, or LinkedIn URL, resolve the source through External Source Intake before selecting or executing the final command workflow.
```

- [ ] **Step 4: Mirror the canonical prompt back into `CLAUDE.md`**

Run:

```bash
cp SKILL.md CLAUDE.md
```

Expected: `CLAUDE.md` contains the same new intake section and routing bullet

- [ ] **Step 5: Verify the canonical and Claude prompt files are aligned**

Run:

```bash
diff -u SKILL.md CLAUDE.md
rg -n "## External Source Intake|references/external-source-intake.md" SKILL.md CLAUDE.md
```

Expected: `diff -u` prints nothing; `rg` prints matching lines from both files

- [ ] **Step 6: Commit the prompt updates**

Run:

```bash
command git add SKILL.md CLAUDE.md
command git commit -m "feat: add shared external source intake to core prompt"
```

Expected: one commit containing only the prompt-file updates

### Task 3: Add The Codex Mirror

**Files:**
- Create: `AGENTS.md`
- Test: shell verification only; no test file exists in this repo

- [ ] **Step 1: Confirm the Codex-facing mirror does not exist yet**

Run:

```bash
test ! -f AGENTS.md && echo "missing-agents"
```

Expected: `missing-agents`

- [ ] **Step 2: Create `AGENTS.md` as an exact mirror of the canonical prompt**

Run:

```bash
cp SKILL.md AGENTS.md
```

Expected: `AGENTS.md` exists and contains the same intake rules as `SKILL.md`

- [ ] **Step 3: Verify the Codex mirror matches the canonical prompt**

Run:

```bash
diff -u SKILL.md AGENTS.md
rg -n "## External Source Intake|references/external-source-intake.md" AGENTS.md
```

Expected: `diff -u` prints nothing; `rg` shows the new intake section and routing bullet in `AGENTS.md`

- [ ] **Step 4: Commit the Codex mirror**

Run:

```bash
command git add AGENTS.md
command git commit -m "feat: add Codex prompt mirror"
```

Expected: one commit containing only `AGENTS.md`

### Task 4: Update `decode` And `research` To Accept URL-Based Intake

**Files:**
- Modify: `references/commands/decode.md`
- Modify: `references/commands/research.md`
- Test: shell verification only; no test file exists in this repo

- [ ] **Step 1: Confirm the command docs do not yet describe URL-first intake**

Run:

```bash
rg -n "LinkedIn job URL|job-board URL|company-hosted role URL|External Source Intake" references/commands/decode.md references/commands/research.md || true
```

Expected: no matches

- [ ] **Step 2: Update `references/commands/decode.md`**

Replace the current Required Inputs block with:

```md
### Required Inputs

- At least one JD source: pasted JD text, LinkedIn job URL, direct ATS/job-board URL, or company-hosted role URL
- For batch triage: 2-5 JD sources
```

Update Step 1 in the Logic / Sequence section so it says:

```md
**Step 1: JD Intake**
Accept JD in any format — full posting, bullet-point paste, screenshot description, or supported URL. If a URL is provided, run `references/external-source-intake.md` first. Only continue with full JD analysis when the retrieved source is sufficient for full-JD work; otherwise fail closed and ask for the missing content.
```

- [ ] **Step 3: Update `references/commands/research.md`**

Add this sentence to the Sequence section immediately after the company-name prompt:

```md
If the candidate already provided a careers page, company page, or role URL, use that as the first Tier 1 source instead of starting from a generic search result.
```

Update the Tier 1 verification bullet so it reads:

```md
- **Tier 1 — Verified**: Information directly retrieved from the company's own website, careers page, a provided role URL, blog, or from the job description/candidate-provided context. Cite the source.
```

- [ ] **Step 4: Verify the updated command docs reference URL intake**

Run:

```bash
rg -n "LinkedIn job URL|job-board URL|company-hosted role URL|provided role URL|External Source Intake" references/commands/decode.md references/commands/research.md
```

Expected: matches in both files for the new URL-oriented language

- [ ] **Step 5: Commit the `decode` and `research` updates**

Run:

```bash
command git add references/commands/decode.md references/commands/research.md
command git commit -m "docs: add URL intake guidance to decode and research"
```

Expected: one commit containing only the two command-reference updates

### Task 5: Update `prep`, `linkedin`, And `help` For Capability-Sensitive Browsing

**Files:**
- Modify: `references/commands/prep.md`
- Modify: `references/commands/linkedin.md`
- Modify: `references/commands/help.md`
- Test: shell verification only; no test file exists in this repo

- [ ] **Step 1: Confirm the old LinkedIn browsing prohibition is still present**

Run:

```bash
rg -n "can't browse LinkedIn directly|cannot browse LinkedIn profiles directly" references/commands/prep.md references/commands/linkedin.md
```

Expected: matches in `prep.md` and/or `linkedin.md`

- [ ] **Step 2: Update `references/commands/prep.md`**

Replace the first paragraph under **Interviewer Intelligence** with:

```md
When the candidate provides interviewer LinkedIn URLs or profile links, use `references/external-source-intake.md` to retrieve profile context when the environment supports browsing. If browsing is unavailable or the profile remains blocked, ask the candidate to share the key professional details manually. This is still one of the highest-leverage prep activities — knowing who's across the table changes story selection, framing, and signal-reading.
```

Replace the final sentence of the Input requirement paragraph with:

```md
If the candidate shares a URL but not the profile content, first attempt external source intake. Only ask them to share key details manually when browsing is unavailable or the retrieved source is insufficient.
```

- [ ] **Step 3: Update `references/commands/linkedin.md`**

Replace the Required Inputs block with:

```md
## Required Inputs

- LinkedIn profile URL or full profile text (pasted sections)
- Target role context (from coaching_state.md Profile, or ask)
```

Replace Step 1 in the Logic / Sequence section with:

```md
### Step 1: Profile Intake

Ask for the LinkedIn profile URL or pasted text. If a URL is provided, run `references/external-source-intake.md` first. If the environment supports browsing and the source is sufficient, use the fetched profile text directly. If browsing is unavailable or the source is blocked or too thin, ask the candidate to paste the relevant sections instead.
```

Update the `help` descriptions so these lines mention URL intake explicitly:

```md
| `research [company]` | Company research ... Accepts company and careers-page URLs as Tier 1 starting points. |
| `decode` | JD decoder ... Accepts pasted JDs or supported job URLs. |
| `prep [company]` | Full prep brief ... Accepts JD URLs and interviewer LinkedIn URLs. |
| `linkedin` | LinkedIn profile optimization ... Accepts profile URLs or pasted profile text. |
```

- [ ] **Step 4: Verify the prohibition is gone and capability-sensitive wording is present**

Run:

```bash
rg -n "can't browse LinkedIn directly|cannot browse LinkedIn profiles directly" references/commands/prep.md references/commands/linkedin.md && exit 1 || true
rg -n "environment supports browsing|External Source Intake|Accepts pasted JDs or supported job URLs|Accepts profile URLs" references/commands/prep.md references/commands/linkedin.md references/commands/help.md
```

Expected: first command prints nothing; second command prints matches from all three files

- [ ] **Step 5: Commit the `prep`, `linkedin`, and `help` updates**

Run:

```bash
command git add references/commands/prep.md references/commands/linkedin.md references/commands/help.md
command git commit -m "docs: make LinkedIn and command intake capability-sensitive"
```

Expected: one commit containing only the three command-reference updates

### Task 6: Update The README And Run The Final Verification Pass

**Files:**
- Modify: `README.md`
- Test: shell verification only; no test file exists in this repo

- [ ] **Step 1: Confirm the README does not yet document URL-based intake or bundled provider files**

Run:

```bash
rg -n "URL Intake|job-board URL|LinkedIn job URL|AGENTS.md is already included|CLAUDE.md is already included" README.md || true
```

Expected: no matches

- [ ] **Step 2: Update the Quick Start activation text and add a short URL-intake note**

Make these exact README changes:

Replace the Claude quick-start rename step with:

```md
2. Open the folder in Claude Code.

`CLAUDE.md` is already included in the repo.
```

Replace the Codex quick-start rename step with:

```md
2. Open the folder in Codex.

`AGENTS.md` is already included in the repo.
```

Update the repository tree snippet so the prompt-file lines read:

```md
├── SKILL.md                            # Canonical skill source
├── CLAUDE.md                           # Claude-facing prompt mirror
├── AGENTS.md                           # Codex-facing prompt mirror
```

Add this short section after the Quick Start block:

```md
## URL Intake

`decode`, `prep`, `research`, and `linkedin` can now work from supported LinkedIn, careers-page, and job-board URLs. When the environment supports browsing, the coach will fetch the source first. When browsing is unavailable or the page is too thin, the coach fails closed and asks you for the missing content instead of guessing.
```

- [ ] **Step 3: Run the final shell verification suite**

Run:

```bash
rg -n "can't browse LinkedIn directly|cannot browse LinkedIn profiles directly" SKILL.md CLAUDE.md AGENTS.md references/commands README.md && exit 1 || true
diff -u SKILL.md CLAUDE.md
diff -u SKILL.md AGENTS.md
rg -n "## External Source Intake|references/external-source-intake.md|LinkedIn job URL|Accepts profile URLs|## URL Intake" SKILL.md CLAUDE.md AGENTS.md references/commands README.md
```

Expected:
- first command prints nothing
- both `diff -u` commands print nothing
- final `rg` command prints matches from the canonical prompt, mirrors, command refs, and README

- [ ] **Step 4: Run the manual prompt-drill verification**

Run these five manual prompt checks in a browser-capable agent session, using real URLs from your current job-search materials:

1. `decode this role:` followed by one real public ATS posting URL
2. `decode this LinkedIn job:` followed by one real LinkedIn job URL from your signed-in session
3. `prep Stripe using this JD:` followed by one real job URL, then `and this interviewer:` followed by one real LinkedIn profile URL
4. `research this company from its careers page:` followed by one real careers-page URL
5. `linkedin audit this profile:` followed by one real LinkedIn profile URL

Expected for every prompt:
- the agent tries URL intake before asking for pasted text
- the agent surfaces the resolved source it used
- the agent does not treat thin previews as complete JDs
- the agent asks for pasted content only when browsing is unavailable or insufficient

- [ ] **Step 5: Commit the README and verification-driven prompt/doc updates**

Run:

```bash
command git add README.md
command git commit -m "docs: document URL intake and provider prompt files"
```

Expected: one commit containing only the README update

## Self-Review Checklist

- Spec coverage:
  - shared intake contract: Task 1 and Task 2
  - canonical prompt plus mirrors: Task 2 and Task 3
  - `decode`, `prep`, `research`, `linkedin`, `help`: Task 4 and Task 5
  - fail-closed behavior and README communication: Task 1, Task 5, and Task 6
- Placeholder scan:
  - no `TBD`, `TODO`, or "implement later" language remains in this plan
  - every shell verification command includes an expected result
  - every commit step includes an exact commit command
- Type consistency:
  - shared terms stay consistent across tasks: `External Source Intake`, `references/external-source-intake.md`, `authenticated browser fetch`, `fail closed`
