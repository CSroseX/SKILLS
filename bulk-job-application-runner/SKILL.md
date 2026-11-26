---
name: bulk-job-application-runner
description: Triage and apply to job links in bulk from a tracker range the user names (a Google Sheet tab or an Excel/CSV file, plus the rows to work): validity checking, platform-specific apply flows, resume selection, and status bookkeeping.
---

# Bulk Job Application Runner

Processes the tracker rows the user specifies in their own logged-in browser (e.g., LinkedIn, Naukri, Foundit, Wellfound, Weekday, Instahyre, or company ATS portals). Goal: fast, accurate applications without touching unrelated rows.

## Required capabilities

Use whatever tools your host provides, as long as together they can do all of the following:

| Capability | Examples of what provides it |
| --- | --- |
| Drive the user's logged-in browser: open tabs, navigate, keep tabs open | Playwright MCP (extension or CDP-attach mode), browser-use, browser extension agent |
| Run JavaScript in a page and read the result | Any browser tool with an evaluate / execute-JS action |
| Read page text and find elements | Page-text or accessibility-snapshot action |
| Click, type, select, and upload a file by selector or element ref | Click, fill, and upload actions |
| Read and write individual spreadsheet cells and dropdown validation values | Google Sheets API/MCP, or `openpyxl` / `pandas` for Excel and CSV |
| Receive attached files and return an updated file to the user | The host's file attachment / delivery mechanism |

A fresh headless browser will hit login walls. If you cannot drive a logged-in browser session, stop and say so before touching any row.

Small-payload note: if your JS tool limits script size, keep each payload small and inject helpers in separate calls.

## Which tracker and rows

There is no fixed sheet, tab, or file for this skill. The user specifies the source every time: a Google Sheet (link or name) and tab, or an Excel/CSV file they attach, plus the range (e.g., rows 100-120, rows dated `25-11-2026`, or every row with blank `Status`).

- Work only the range given. Never pick a sheet, tab, file, or rows yourself, never default to a tab from an earlier run, and never extend past the range.
- If the user has not said which tracker or rows to work, ask before opening anything. Confirm back once in one line: tracker, tab, row numbers, row count.
- Skip rows inside the range that already have `Applied` (or any final status set by the user).
- Match date-based ranges against the tracker's display format (e.g., `DD-MM-YYYY` if configured in `references/form-details.md`).

## Setup

- Open the tracker in its own tab and keep it there for the whole run; apply in other tabs. For an Excel/CSV file, work on a copy and return the updated file at the end (and after every ~10 rows so progress is not lost).
- Load personal answers, contact details, standard compensation/availability responses, and resume file mappings from `references/form-details.md` in this skill folder. If `references/form-details.md` is missing, ask the user for it; never guess contact details or personal facts.
- Verify that the resume PDFs listed in `references/form-details.md` are available. If you cannot see them, ask the user to attach them or provide their file paths before starting.

## Sheet & Status Bookkeeping

Columns differ between trackers, so read the header row first and map by header name: company, role, link (`Lead Site`), status, notes.

- **Statuses:** Use the tracker's own vocabulary. If the status column has a dropdown, read its allowed values and use only those (e.g., `Applied` for submitted, `N/A` for dead/invalid listings when `Not valid` is not an option). If there is no dropdown, use `Applied` and `Not valid`. State in the first status update which value is being used for dead listings.
- **Edit scope:** Edit only the `Status` cell of rows in the given range, with one optional exception below. Never touch other columns, other rows, or other tabs.
- **Notes tag (optional convention):** If a row's `Notes` starts with `NOT APPLIED | ...` (such as rows created by `job-finder`), replace that leading `NOT APPLIED` with `APPLIED DD-MM-YYYY` (today's date) when marking it `Applied`, keeping the rest of `Notes` unchanged. Leave `Notes` untouched for dead, blocked, skipped, or non-tagged rows.
- **Unfinished rows:** Leave `Status` blank for rows you could not finish (blocked, or not submitted because unsure) and list them in the report. Never invent a status value.
- **Formatting:** If the sheet uses conditional formatting by status, writing the text is enough. Otherwise copy the cell format from an existing cell with that status; never set colors manually.
- **Row safety:** Read the company column before writing to confirm row mapping. Write the status as each row finishes, not in a batch at the end.
- **Already applied by user:** If the user says they already applied to a row, mark `Applied` (and update the `NOT APPLIED` prefix if present) and skip.

## Use what Notes already hold

Rows populated by `job-finder` may carry `Posted: ...`, `Track:`, `Resume:`, `Fit:`, `Gaps:`, and `JD: ...`. Read `Notes` before opening the link:

- **JD:** Use it to answer screening questions, tailor short "why this company/role" responses, and pick the resume. If the live page clearly differs from the JD, trust the live page.
- **Posted:** If the posting date is more than ~3 weeks old, check dead-listing signals first and move on quickly.
- Rows without these tags are normal: fall back to reading the job page.

## Per row

Open the link. Whenever a listing turns out dead, on load or partway through applying, mark the status as the dead-listing value (`Not valid` or `N/A`), close the tab, and move on. Never skip a row silently.

Quick check on load:

```js
var b=document.body.innerText;
JSON.stringify({t:document.title.slice(0,50),
d:(b.match(/no longer accepting|no longer available|\bclosed\b|not found|\b404\b|has been removed|expired/gi)||[]).slice(0,2)})
```

| Platform | Dead signal |
| --- | --- |
| LinkedIn | `No longer accepting applications`; `Job id provided may not be valid or the job posting has been removed` |
| Internshala | `Applications are closed for this job` |
| HireDoor | Title ends `(Closed)`; `Job not found`; redirect to `/premium` |
| Naukri | Redirects to a search page with `expJD=true` |
| Wellfound / Instahyre | `no longer available` / `no longer accepting applications` |
| Indeed | `This job has expired` |

Also mark dead (`Not valid` or `N/A`): careers or aggregator pages where the role is gone, broken forms that refuse to submit, paywalls, or unreachable apply paths.

False positives (not dead):
- `Not disclosed` matches `closed` if you omit the `\b` word boundary.
- Warning links like "Is this job expired?" or "Posted over 15 days ago... may have closed" appear on live pages.
- Similar-jobs sidebars can fake `Easy Apply` in page text. Detect the visible button instead:

```js
!![].slice.call(document.querySelectorAll('button')).filter(x=>x.offsetParent&&/Easy Apply/i.test(x.innerText))[0]
```

## Applying

Move quickly (target under one minute per application): batch tool calls, skip confirmation screenshots between obvious steps, and keep each JS payload small.

Do not submit if unsure about any filled detail (a guessed answer, stale autofill, or a question not covered by `references/form-details.md`). Leave that tab open, move to the next row, and list it in the report with the exact field. Otherwise submit without asking. Follow external ATS redirects and finish there.

- **Resume selection:** Follow the `Resume:` tag in `Notes` if present; otherwise match the role title and JD against the track-to-resume mapping in `references/form-details.md`. Always verify the attached resume on the platform before submitting and replace stored defaults if wrong.
- **Format traps:**
  - Numeric-only fields (e.g., LinkedIn): enter plain numbers (e.g., `2` instead of `1-2`, `0` instead of `Immediate`).
  - Salary fields: read the label carefully (LPA vs. annual figure in INR/USD) and convert from `references/form-details.md` accordingly.
  - Yes/no asked as a number field: `1` for yes, `0` for no.
  - If a form parses the resume and fills a contact email that differs from the platform login email, keep the parsed resume email unless `references/form-details.md` specifies otherwise.

## Platform notes

| Platform | Notes |
| --- | --- |
| LinkedIn Easy Apply | Inject small helpers in separate calls: native-setter fill with `input`/`change`/`blur` events for text, radios, selects, and resume upload; loop fill, Next/Review, Submit, and return `stuck@<page>`. On stuck, read the modal (usually one unmatched question). Invalid input stays stale until you focus then blur the field. |
| Naukri | Often one click; success is the page title `Apply Confirmation`. An incomplete-information error means a skipped questionnaire. `Apply on company site` leads to the employer ATS. |
| Foundit | `Quick Apply` is instant; `Apply Now` goes to an external site. |
| Wellfound | For "What interests you...", research the company from the page and JD and write one concise paragraph connecting their mission to the user's relevant experience. Sends after a ~5-minute undo timer. |
| Weekday | Saves name and phone, then opens a LinkedIn posting; apply there. |
| HireDoor | Pass-through. Run `window.open=function(u){window.__cap=u;return{focus(){},close(){}}}`, click Apply, and read `window.__cap`. |
| RecruiterFlow / Keka / JazzHR | Upload resume first (auto-parses). Keka and JazzHR need experience blocks and date pickers (typed dates often do not stick). |
| Workday | Programmatic field-fill often does not register, so click and type like a user. Re-click `Save and Continue` to clear stale errors. Dropdowns: open, then click the option. Consent: click the label text. Upload resume via the hidden file input. Done means `Application Submitted` appears in the page text. |
| Oracle HCM | Profile may be pre-filled: remove the stored resume and upload the track's resume. For conflict-of-interest pages, set all radios to `No` with a JS loop and verify each group. `Go to Next Issue` indicates a missing field. |
| iCIMS | Form lives in a same-origin iframe (`iframe.contentDocument`). Pre-filled password or `Returning Candidate?` means a create-account wall (blocker). |
| eFinancialCareers | `Sign in with Google` when signed out is a blocker. |

## Blockers: stop that row, keep its tab open

- CAPTCHA (e.g., Keka, JazzHR): fill everything else first, then stop.
- Account creation, passwords, OAuth sign-in: leave the tab open for the user to log in.
- `mailto:` applications: offer to draft the email.
- Missing required personal data not in `references/form-details.md`: ask.

Never close or reuse a blocker tab (a reopened tab may lose session state); open a fresh tab for the next row.

Not blockers: a transient tool failure or an unclear safety verdict. Re-read the page and retry once with a different input method (e.g., click by ref instead of fill). Never repeat a mutating action that may have already succeeded without re-reading first.

## Reporting

- Refer to rows by row number in the tracker, not company name.
- Final report states: tracker, tab, and range worked; counts for done, dead, blocked, and skipped; plus a running count of submissions separate from rows marked `Applied`.
- Send a brief one-line status update during long runs.
- Group recurring blockers so one user fix unblocks multiple rows.
- Lead the final report with unfinished rows: row number, reason, and the exact action needed from the user (including any unsure fields).
