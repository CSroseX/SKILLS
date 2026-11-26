---
name: job-finder
description: Run an unattended or on-demand daily job search across multiple tracks, validate live links, deduplicate against a spreadsheet tracker, score fit against the user's resumes, and append new leads.
---

# Daily Job Finder

Discovers, validates, scores, and logs new job openings into the user's application tracker. Designed to work both interactively and in unattended/scheduled runs.

## Configuration & Resumes

Before searching, load `references/preferences.md` in this skill folder for:
- Candidate profile, graduation year, and key projects/experience.
- Resume files or links mapped by track (e.g., AI/ML vs. SDE/Backend). Read the resumes as the source of truth for skills when scoring fit. If a resume cannot be accessed, note that in the report and fall back to the skill list in `references/preferences.md`.
- Target titles, adjacent roles, hard excludes, experience range, location/remote rules, and compensation floor.
- Target daily minimum (default: **10+ new rows**).
- Tracker location (Google Sheet ID/URL + tab name, or Excel/CSV path) and column headers.

If `references/preferences.md` is missing, stop and ask the user to set it up from `references/preferences.example.md`. In unattended mode, do not ask clarifying questions mid-run—make reasonable calls and document them in the report.

## Application Tracker (Read First, Write Last)

1. **Read headers and existing rows first:** Read only the configured tracker tab/columns (default `A:J`). Map columns `B`–`J` by header name (typically `Company`, `Role`, `Lead Site`, `Status`, `Salary`, `Lead Name`, `Lead Position`, `Contact Details`, `Notes`). Never touch columns beyond the configured range, never add tabs, never change headers, and never edit or delete existing rows.
2. **Deduplicate:** Exclude any job already in the tracker—matching by URL (`Lead Site`), platform job ID (e.g., LinkedIn job ID or Indeed `jk`), or `Company + Role` regardless of `Status`.
   - For company-level rows (where `Role` lists multiple titles or is blank and `Lead Site` is a careers root), treat a job as a duplicate only if its title appears in that `Role` cell for the same company.
   - Ignore summary/log rows (e.g., counts like `45++`). A different role at an already-listed company is not a duplicate.
   - Report only the count of skipped duplicates.
3. **Append safely:** Find the last row with any value in columns `A`–`D` (re-check right before writing in case the user added rows during the run). Append new rows immediately after it—never fill intentional blank gaps in the middle of the sheet.
4. **Preserve formatting:** Never modify borders, colors, fonts, wraps, row heights, column widths, conditional formatting, or data validation.
   - If the `Date` column (`A`) uses a fixed `DD-MM-YYYY` real-date validation (Google Sheets), write the date serial number (`(run_date - date(1899, 12, 30)).days`) as a numeric value, apply only `userEnteredFormat.numberFormat` (`type: "DATE"`, `pattern: "dd-mm-yyyy"`), and read the cells back to confirm they display as `DD-MM-YYYY`.
5. **Row fields:**
   - `Date`: run date
   - `Company`, `Role`, `Lead Site` (direct job URL)
   - `Status`: leave **blank** (leads are not applied yet)
   - `Salary`: listed pay or `Not listed`
   - `Lead Name`, `Lead Position`, `Contact Details`: fill only if named in the posting, else blank
   - `Notes`: single-line structured summary:
     `NOT APPLIED | Posted: <DD-MM-YYYY or ~DD-MM-YYYY or Not shown> | Tier: A/B/C | Verified: <check details> | Job ID: <id> | <Location> | Exp: <asked> | Track: <track> | Resume: <track> | Fit: NN% | Gaps: <gaps> | JD: <cleaned one-line JD>`
6. **JD formatting in Notes:** Strip company boilerplate, benefits, and EEO text; keep role summary, responsibilities, requirements, and skills. Replace line breaks/bullets with ` · `, collapse repeated spaces, and trim at a word boundary around ~1,500 characters ending with `…` so the cell never wraps taller.

If the tracker cannot be read, note the error in the report and continue without deduplication, still appending only to the configured tab.

## Discovery & Search

Use all available search, job-board, and web-fetch tools in your host environment and combine results into one deduplicated pool:
- Run up to ~40 varied queries across titles, target cities/remote, and sources (job boards, company career pages, and ATS boards such as Greenhouse, Lever, Ashby, Workday, Keka, Workable) until the minimum daily target (e.g., 10+ rows) is met.
- **Recency:** Prefer postings from the last 48 hours, then 7 days, 14 days, and up to 21 days. Skip anything confirmed older than 21 days.
- Track which discovery sources surfaced candidates and which made the final list; note any tool errors and continue with the remaining tools.

## Validation Tiers

Every logged row must be at least **Tier C**:

1. **Apply role & exclusion filters first** from `references/preferences.md`.
2. **Tier A (verified):** Two independent checks/tools confirm the posting is open with a visible apply control and posting date.
3. **Tier B (likely open):** At least one check confirms the page loads (HTTP 200, no closed/expired/404 text) with an apply control or direct employer ATS listing; date may come from an aggregator or listing.
4. **Tier C (lead):** Named company + working-looking direct link + date from a listing/snippet, not yet page-verified. Use only as needed to reach the daily target.
5. **Drop only on hard closure evidence:** 404/410, explicit closed/expired/filled text, passed closing date, redirect to a generic search page, or confirmed >21 days old. Downgrade blocked pages to Tier B/C instead of dropping.
6. **Direct links only:** Resolve short/redirect links (such as `to.indeed.com`) to the canonical job URL (`https://in.indeed.com/viewjob?jk=<id>`) or, better, the employer's direct ATS/careers page. Re-check each URL once before writing.

## Output & Reporting

1. **Summary table:** `Role | Company | Track | Tier | Job ID | Location | Pay | Exp asked | Posted date | Fit | Verified-by | Link`
2. **Per-job note:** One line per job explaining why it fits and key gaps.
3. **Run diagnostics:**
   - `Sources used`: candidates surfaced vs. finalized per search tool/source, plus any errors.
   - `Link check`: links resolved/fixed, jobs dropped and why, and count of Tier A / B / C.
   - `Tracker range written`: exact tab and row range updated.
4. **Notifications (if supported by host):** Send a brief summary notification when new jobs are added (counts by tier + top 3) or if the run falls short of the daily minimum; stay silent if nothing new is found.
