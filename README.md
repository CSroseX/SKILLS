# Agent Skills for Job Search & Bulk Applications

Two provider-neutral agent skills that work together (or standalone) to automate daily job discovery and bulk job applications:

1. **`job-finder`** — Searches job boards and company career pages, scores roles against your resume(s), validates live links, deduplicates against your tracker, and logs new leads.
2. **`bulk-job-application-runner`** — Reads target rows from your tracker, opens postings in your logged-in browser session, filters out closed listings, fills applications using your local profile, and updates tracker statuses.

## Prerequisites

Your AI agent or host environment needs:
- **Browser automation** that can drive your **logged-in** Chrome/browser session (e.g., Playwright MCP in CDP/extension mode, browser-use, or a browser extension agent) and execute small JavaScript snippets.
- **Spreadsheet access** to read and write cells in Google Sheets or Excel/CSV files.
- **Web search / page fetch** tools (for `job-finder`).

## Quick Start

1. **Clone or copy** the skill folders into your agent's skills directory.
2. **Create your local config files** from the templates (these are gitignored so your personal info is never committed):
   - Copy `job-finder/references/preferences.example.md` to `job-finder/references/preferences.md` and fill in your profile, resume paths, target roles, and tracker link.
   - Copy `bulk-job-application-runner/references/form-details.example.md` to `bulk-job-application-runner/references/form-details.md` and fill in your contact details, education, standard answers, and resume filenames.
3. **Set up your tracker** (Google Sheet or Excel/CSV) with these columns:
   `Date` | `Company` | `Role` | `Lead Site` | `Status` | `Salary` | `Lead Name` | `Lead Position` | `Contact Details` | `Notes`
4. **Run `job-finder`** (manually or on a daily schedule) to populate new `NOT APPLIED` leads, then **run `bulk-job-application-runner`** on a chosen row range or date.

## Privacy Note

Never commit `references/form-details.md`, `references/preferences.md`, or resume PDFs to a public repository. Both are excluded in `.gitignore`.
