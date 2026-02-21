---
name: job-applier
description: "Apply to jobs using the OpenClaw-managed browser on your MacBook. Usage: /job-applier run --profile <path> [--max 10] [--dry-run] [--auto-submit]"
user-invokable: true
metadata: { "openclaw": { "emoji": "🧑‍💼", "requires": { "bins": ["jq"] } } }
---

# Job Applier Skill

Automate job applications with your own resume and preferences using OpenClaw browser controls.

## Goal

Find matching jobs at user-specified companies and submit applications from your MacBook with a repeatable, auditable flow.

## Safety + Policy

- Only use domains listed in `targetDomains` from the profile.
- Never apply to jobs outside `allowLocations` / `allowRemote` / salary bounds.
- Never fabricate experience, titles, dates, employers, or degrees.
- If a question cannot be answered from the profile, stop and ask the user.
- Respect site terms/rate limits and add small delays between submissions.
- If captcha, MFA, or anti-bot challenge appears, pause and ask user to take over.

## Inputs

Arguments (after `/job-applier`):

- `run` (required)
- `--profile <path>` (required)
- `--max <n>` (optional, default 10)
- `--dry-run` (optional, browse + prefill, do not submit)
- `--auto-submit` (optional, submit without per-job confirmation)

## Profile Contract

The profile JSON must include:

- `identity`: name, email, phone, location, yearsExperience
- `documents`: resumePath, optional coverLetterTemplatePath
- `companyTargets`: list of companies with careers URLs and optional per-company title hints
- `targetJobTitles`: user-provided job titles to actively search/apply for
- `targetDomains`: allowed job board/company domains
- `jobFilters`: includeKeywords, excludeKeywords, allowLocations, allowRemote, minSalary
- `workPrefs`: visaStatus, sponsorshipNeeded, workAuthCountries, noticePeriod
- `screeningAnswers`: reusable answers for common questions
- `resumeMatching` (optional): enable resume-guided title inference and score threshold

## Execution Flow

### 1) Parse + validate

1. Parse args.
2. Require `run` and `--profile`.
3. Load and validate the profile file.
4. Resolve defaults: `max=10`, `dryRun=false`, `autoSubmit=false`.
5. Require `companyTargets` to be non-empty and each entry to include:
   - `company`
   - optional `careersUrl` (if missing, discover from company domain/search)
6. Require `targetJobTitles` to be non-empty.
7. For any company target without `careersUrl`, discover likely careers page URL from company domain/search results.
8. Ensure every resolved `companyTargets[*].careersUrl` hostname is allowed by `targetDomains`.
9. Read and parse the resume from `documents.resumePath`.
10. Build a candidate title set by combining:

- `targetJobTitles`
- `companyTargets[*].jobTitleHints` (if present)
- titles inferred from resume experience/skills (if `resumeMatching.enabled=true`)

11. Rank candidate titles by relevance to resume; keep top `resumeMatching.maxInferredTitles` inferred titles.
12. If no candidate titles remain, stop and ask user to provide/adjust titles.
13. Document path handling:

- In normal mode (no `--dry-run`), require `documents.resumePath` to exist.
- In `--dry-run`, if `resumePath` or `coverLetterTemplatePath` is missing, placeholder, or unreadable, auto-create dummy files under `~/.openclaw/workspace-support/skills/job-applier/dummy-docs/` and continue.
- In `--dry-run`, never hard-stop only because resume/cover-letter files are missing.
- For browser uploads (all modes), stage files into `/tmp/openclaw/uploads` first and upload from that staged path only.
- Staged upload files must be regular files (not symlinks). Always copy the resume into uploads root.

14. Identity handling:

- In normal mode (no `--dry-run`), try to extract identity fields from resume first (name, email, phone, location, yearsExperience estimate).
- If any required identity field is still missing after extraction, ask user before continuing.
- In `--dry-run`, allow placeholder identity values and continue.
- Do not edit the user's profile file automatically unless explicitly requested.

15. Print a concise plan summary before browsing:

- profile path
- max jobs
- dry-run/auto-submit mode
- target domains
- company targets
- candidate job titles (including resume-inferred titles)

### 2) Open browser session

Use OpenClaw-managed browser profile:

- Ensure browser is available.
- Use `openclaw` browser profile unless user asked otherwise.
- Start from each resolved `companyTargets[*].careersUrl` page.

Browser tool contract (must follow exactly):

- For every open/navigation/snapshot call, always pass `targetUrl`.
- Before acting on page elements, run a fresh `snapshot --interactive` and use only refs from that snapshot.
- Refs are not stable across navigation/DOM updates. If action fails with unknown/stale ref, run a new snapshot and retry once with the new ref.
- Prefer `click` / `type` / `fill` flows over keyboard-only shortcuts.
- Keep browser actions scoped to allowed domains only.

### 3) Find candidates

For each company target:

1. Search jobs using candidate title set + include keywords.
2. Skip any card matching exclude keywords.
3. Open role detail and collect:
   - title
   - company
   - location
   - salary (if shown)
   - link
4. Keep only roles matching filters.
5. Keep only roles where title strongly matches one of:
   - user `targetJobTitles`
   - per-company `jobTitleHints`
   - resume-inferred titles above `resumeMatching.minimumMatchScore`
6. Stop when collected `max` unique roles.

### 4) Apply per role

For each selected role:

1. Open application form.
2. Fill fields from profile:
   - contact info
   - work authorization / sponsorship
   - years experience
3. Upload resume from `documents.resumePath`.
   - First copy the resume into `/tmp/openclaw/uploads/<safe-file-name>` (never symlink).
   - Verify staged file exists and is a regular file before upload.
   - Use browser upload with the staged path only (OpenClaw browser rejects paths outside uploads root).
   - Always include `targetId` for upload actions so the file is attached to the intended tab.
   - Preferred flow: run fresh `snapshot --interactive`, find the `<input type=file>` ref, then call `upload` with `inputRef`.
   - If file input is hidden and only a trigger button exists: call a single `upload` action with `ref` included so arming + click happen in one step.
   - Do not combine `ref` with `inputRef` or `element` in the same upload action.
   - Never click upload first and then try to handle a native picker manually.
   - After upload, run another snapshot and confirm resume filename appears in the form UI.
4. Fill known screening answers from `screeningAnswers`.
5. If cover letter required:
   - Use template if provided.
   - Personalize with company + role; do not invent claims.
6. Validate required fields before submit.

If required fields are missing from profile/resume-derived data, stop and ask user.

Submission rule:

- If `--dry-run`, do everything except final submit.
- If not dry-run and not `--auto-submit`, ask for confirmation per role:
  - `submit` / `skip` / `edit`
- If `--auto-submit`, submit directly once checks pass.

### 5) Log outcomes

Write/update a JSONL log at:

- `~/.openclaw/workspace/skills/job-applier/applications-log.jsonl`

Each line should include:

- timestamp
- role title
- company
- careers page URL
- source URL
- status (`submitted` | `dry-run-ready` | `skipped` | `failed`)
- reason (for skipped/failed)
- matchedBy (`explicit-title` | `company-hint` | `resume-inferred`)

### 6) Final report

At the end, output a compact summary table:

- total inspected
- submitted
- dry-run-ready
- skipped
- failed

Then list the top failures with actionable next steps (e.g., missing answer, captcha, blocked upload).

## Failure Handling

- On navigation timeout: retry once, then skip role.
- On selector mismatch: refresh snapshot and retry once.
- On upload error: verify staged file exists under `/tmp/openclaw/uploads`, then retry once.
- On upload error mentioning invalid path/symlink/non-regular file: restage by copying (not linking) to `/tmp/openclaw/uploads` and retry with fresh snapshot refs.
- On repeated failure for same site: stop processing that site and continue others.
- On browser service/tool contract errors (`targetUrl required`, unreachable browser service, unknown ref): stop retries for the current step, refresh context once (new snapshot / valid target URL / valid staged upload path), then continue.
- Avoid `keyboard.press` with unsupported key names (for example `Left`). Prefer click/type flows; if keyboard is needed, use canonical keys like `ArrowLeft`, `ArrowRight`, `Enter`, `Tab`.

## Missing-Info Prompts

Ask user (once, upfront) for any missing critical details:

- companies and careers URLs (if `companyTargets` missing/empty)
- company names or domains (if careers URLs are omitted for discovery)
- target job titles (if `targetJobTitles` missing/empty)
- resume path (if unavailable in non-dry-run)
- identity fields unresolved from resume parsing
- missing screening answers required by a form

## User Commands

Examples:

- `/job-applier run --profile ~/.openclaw/workspace/skills/job-applier/profile.json --dry-run --max 15`
- `/job-applier run --profile ~/.openclaw/workspace/skills/job-applier/profile.json --max 10`
- `/job-applier run --profile ~/.openclaw/workspace/skills/job-applier/profile.json --max 20 --auto-submit`

## Output Quality

Keep responses short and operational:

- Show what is happening now.
- Ask only when required data is missing or confirmation is needed.
- Always end with the current counters and next action.
