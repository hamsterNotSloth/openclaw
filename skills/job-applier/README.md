# Job Applier Skill (OpenClaw)

This skill lets OpenClaw use your MacBook browser session to find and apply for matching jobs.

## 0) Install skill to your OpenClaw workspace

```bash
mkdir -p ~/.openclaw/workspace/skills
cp -R ./skills/job-applier ~/.openclaw/workspace/skills/
```

## 1) Prepare profile

1. Copy `profile.example.json` to `profile.json`.
2. Set your resume path and preferences (identity can be inferred from resume and only missing fields are prompted).
3. Add `companyTargets` (company + optional careers URL or company website + optional title hints).
4. Add `targetJobTitles` (the job titles you want to apply for).
5. Keep only trusted job domains in `targetDomains`.

Example shape:

```json
{
  "companyTargets": [
    {
      "company": "Stripe",
      "careersUrl": "https://stripe.com/jobs/search",
      "jobTitleHints": ["Software Engineer", "Backend Engineer"]
    }
  ],
  "targetJobTitles": ["Software Engineer", "Backend Engineer"],
  "resumeMatching": {
    "enabled": true,
    "maxInferredTitles": 10,
    "minimumMatchScore": 0.5
  }
}
```

If `careersUrl` is missing, the skill can discover it from company name/domain and proceed.

## 2) Refresh skills

- In OpenClaw, ask: `refresh skills`
- Or restart your gateway.

## 3) Run it

Dry-run first:

```bash
openclaw agent --message "/job-applier run --profile ~/.openclaw/workspace/skills/job-applier/profile.json --dry-run --max 10"
```

Then real submissions:

```bash
openclaw agent --message "/job-applier run --profile ~/.openclaw/workspace/skills/job-applier/profile.json --max 10"
```

Optional (fully automatic submit):

```bash
openclaw agent --message "/job-applier run --profile ~/.openclaw/workspace/skills/job-applier/profile.json --max 20 --auto-submit"
```

## Notes

- Keep `--dry-run` on until form filling is reliable for your target sites.
- The skill uses your resume + target titles to rank role fit before applying.
- The skill pauses when captcha/MFA appears.
- Results are logged to:
  - `~/.openclaw/workspace/skills/job-applier/applications-log.jsonl`
