---
name: linkedin-easy-apply
description: Search LinkedIn for remote job opportunities matching the user's resume and apply via Easy Apply. Use when the user wants to find jobs, apply to jobs, or do a job search on LinkedIn.
argument-hint: [search-keywords]
---

# LinkedIn Easy Apply Skill

Search LinkedIn for fully remote job opportunities and apply via Easy Apply using the user's pre-uploaded resume and profile.

## Requirements

- The Playwright MCP browser must be available
- The user must already be logged into LinkedIn in the browser
- The user's resume must already be uploaded to LinkedIn

## CRITICAL: Resume-Based Answers Only

**Before beginning any applications**, read the user's resume. Set `RESUME_PATH` below to point to your resume file:
```
<RESUME_PATH>
```

**NEVER hallucinate, fabricate, or exaggerate experience.** Every answer to application questions MUST be directly supported by what's in the resume. If the resume doesn't mention a specific technology, framework, or domain, do NOT claim experience with it. When asked about years of experience with something not on the resume, answer "0" or "No" rather than guessing. When answering free-text technical questions, only reference projects, skills, and accomplishments that appear in the resume. Getting caught with fabricated experience is worse than an honest "no."

If you need to re-check specific experience details mid-workflow, re-read the resume rather than guessing from memory.

## Workflow

### 1. Search for Jobs

Use `$ARGUMENTS` as the search keywords. If no arguments provided, ask the user what roles to search for.

Navigate to LinkedIn Jobs search filtered for **Remote**, **Easy Apply**, and **past month**:
```
https://www.linkedin.com/jobs/search/?keywords=<URL-encoded-keywords>&f_WT=2&f_AL=true&f_TPR=r2592000&sortBy=R
```

- `f_WT=2` : Remote only
- `f_AL=true` : Easy Apply only
- `f_TPR=r2592000` : Posted within last 30 days
- `sortBy=R` : Sort by relevance

Run **2-3 different keyword searches** in sequence targeting relevant roles. Tailor search keywords to the user's skills and target roles.

Collect job IDs, titles, and company names from each search into a deduplicated master list. Track applied count across all searches.

### 2. Apply-As-You-Go Loop

**Target: 20 applications per session.** Do not compile a full list before applying. Instead, process jobs one at a time: verify, apply if valid, move to next. Run additional keyword searches as needed until you hit 20 applications or exhaust all searches.

For each job from the search results:

#### Step 2a: Filter by relevance (from search results, no page visit needed)

Immediately skip jobs that are:
- **Irrelevant titles**: e.g. "Science tutor", "Viral Content Lead", "Content Writer" when searching for AI engineer roles. Only keep roles that plausibly match the user's skills and search keywords.
- **Known blocklisted companies**: See "Known Scam/Low-Quality Companies" section below.
- **Already applied**: Jobs showing "Applied" badge in search results.

#### Step 2b: Visit job page, verify, and apply immediately if valid

Navigate to `https://www.linkedin.com/jobs/view/<JOB_ID>/`, take a snapshot, and run a single grep to check everything:
```bash
grep -E '(Easy Apply to this|Applied.*ago|See application|Staffing|Recruiting|employees|followers|\$|/yr|/hr)' snap | head -15
```

**Auto-skip if ANY of these are true:**
1. **No Easy Apply button**: No `link "Easy Apply to this job"` in snapshot.
2. **Already applied**: "Applied X ago" or "See application" text present.
3. **ANY staffing/recruiting company**: If the snapshot contains `"Staffing and Recruiting"` as the industry type, skip immediately. No exceptions, regardless of company size.
4. **Salary below threshold**: Listed salary below your minimum. If no salary listed, proceed.
5. **Disproportionate followers**: <50 employees but >100K followers.

**If the job passes all checks, apply immediately**, then add a random wait (`sleep $((RANDOM % 11 + 5))`) and move to the next job.

#### Step 2c: Qualification check from job description

Before clicking Easy Apply, scan the job description in the snapshot for required skills/technologies. If the role's **primary required stack** consists of technologies the user has zero experience with (based on the resume), skip it.

**Rule of thumb**: If you would have to answer "0" or "No" to **3 or more** of the role's core technology questions, the user is not qualified. Skip it.

Keep a running count. When you reach 20 applied, stop and report results. If a search is exhausted, run the next keyword search and continue.

### 3. Application Form Strategy

For each Easy Apply job, click the Easy Apply button and work through the application form. If the click doesn't open a dialog, try navigating directly to `https://www.linkedin.com/jobs/view/<JOB_ID>/apply/?openSDUIApplyFlow=true`.

#### CRITICAL: Context-Efficient Strategy

LinkedIn pages produce huge snapshots (50-130KB). `browser_click` returns full incremental snapshot diffs even with `--snapshot-mode none`, consuming massive context. Use these rules:

**Rule 1: Use `browser_run_code` for ALL interactions.** This is the ONLY tool that respects `--snapshot-mode none` and returns tiny responses. Use it for everything: navigation, clicking job cards, clicking dialog buttons (Next, Review, Submit, Follow, Done/Dismiss), and filling radio buttons.

```javascript
// Navigate
async (page) => { await page.goto('https://...'); return 'navigated'; }

// Click job card in search results
async (page) => {
  await page.locator('[class*="job-card"]').filter({ hasText: 'Job Title' }).first().click();
  await page.waitForTimeout(2000);
  return 'clicked';
}

// Click dialog buttons (Next, Review, Submit, Done, Dismiss)
// NOTE: Button name varies between forms - try both patterns
async (page) => {
  const next = page.getByRole('button', { name: /Next|Continue to next step/ });
  await next.click();
  await page.waitForTimeout(2000);
  return 'Next';
}

// Uncheck Follow + Submit in one action
async (page) => {
  await page.getByText('Follow CompanyName to stay').click();
  await page.waitForTimeout(1000);
  await page.getByRole('button', { name: 'Submit application' }).click();
  await page.waitForTimeout(2000);
  return 'submitted';
}

// Click radio button labels (LinkedIn radio inputs are intercepted by labels)
async (page) => {
  await page.locator('[data-test-text-selectable-option__label="Yes"]').first().click();
  return 'selected';
}

// Click ALL Yes radio buttons at once (efficient for pages with many radio groups)
async (page) => {
  const yesLabels = page.locator('[data-test-text-selectable-option__label="Yes"]');
  const count = await yesLabels.count();
  for (let i = 0; i < count; i++) {
    await yesLabels.nth(i).click();
    await page.waitForTimeout(500);
  }
  return `clicked ${count} Yes buttons`;
}

// Click checkbox labels (same interception pattern as radio buttons)
async (page) => {
  await page.locator('[data-test-text-selectable-option__label="Python"]').click();
  return 'checked';
}
```

**Rule 2: NEVER use `browser_click`.** It always returns full snapshot diffs regardless of `--snapshot-mode none`, wasting 10-50KB of context per call. The only exceptions are `browser_fill_form` and `browser_select_option` which are efficient for their specific purposes.

**Rule 3: Use `browser_snapshot` with filename for inspecting page state.** Never read snapshots inline.
```
browser_snapshot with filename="snap"
```
The snapshot file is saved to the CWD. Extract ONLY what you need:

```bash
# Find Easy Apply button or Applied status in job detail
grep -E '(Easy Apply|Applied.*ago|See application).*ref=e' snap | head -5

# Find form fields and buttons in dialog
grep -E '(heading|percent|Next|Submit|Review|textbox|combobox|radio|checkbox)' snap | grep -v 'unchanged\|notification' | head -20

# Extract full application dialog content (most reliable anchor)
# Use 'dialog "Apply' as anchor - it captures heading, progress %, all fields and buttons
python3 -c "
with open('snap') as f: text = f.read()
start = text.find('dialog \"Apply')
if start >= 0:
    chunk = text[start:start+5000]
    lines = chunk.split('\n')
    filtered = [l for l in lines if 'option \"' not in l or 'selected' in l]
    print('\n'.join(filtered[:70]))
"
```

**Rule 4: Pre-filled form steps (Contact, Resume)** are almost always correctly pre-filled. Click Next without inspecting. Only take a snapshot if clicking Next returns an error. **However**, for Additional Questions steps, ALWAYS take a snapshot and verify pre-filled textbox values against the resume. LinkedIn frequently pre-fills incorrect years of experience. Correct all wrong values before proceeding.

**Rule 5: Review page pattern.** Uncheck Follow and Submit in a single `browser_run_code` call (see example above). Always use `getByText('Follow CompanyName to stay')` to uncheck.

**Rule 6: Use `browser_select_option` for Yes/No combobox dropdowns.** LinkedIn's combobox questions (e.g. "Do you have experience with X?") work with `browser_select_option` using the ref and `["Yes"]` or `["No"]` values. Use `browser_fill_form` for textbox fields (years of experience, etc.).

**Rule 7: Radio buttons AND checkboxes in LinkedIn dialogs.** Both radio inputs and checkbox inputs are hidden behind labels that intercept clicks. Use `data-test-text-selectable-option__label` attribute to click them:
```javascript
// Radio button
await page.locator('[data-test-text-selectable-option__label="Yes"]').first().click();
// Checkbox (same pattern - skills checkboxes, etc.)
await page.locator('[data-test-text-selectable-option__label="Python"]').click();
```
When multiple radio groups exist on the same page, use `.nth(N)` to target the correct one. For pages with many Yes/No radio groups, use the batch loop pattern (see Rule 1 examples). Never use `getByLabel().check()` for checkboxes as the label interception causes timeouts.

**Rule 8: Additional Questions can span below the visible area.** Some forms have fields below the initial radio/combobox questions (skills checkboxes, English proficiency dropdowns, hourly rate textboxes) that aren't visible without scrolling. If "Review" fails with validation errors, take a fresh snapshot and look for `alert` tags with "Please make a selection" or "Please enter a valid answer" to find the missed fields. Always extract a large chunk (5000+ chars) from the dialog anchor to catch all fields.

#### Anti-Detection: Random Wait Times

**CRITICAL**: Between each major action, insert a random wait to avoid rate limiting and bot detection:

- Between job applications: `sleep $((RANDOM % 11 + 5))` (5-15s)
- Between form steps: `sleep $((RANDOM % 5 + 2))` (2-6s)
- Between field fills: `sleep $((RANDOM % 3 + 1))` (1-3s)
- Do NOT wait before the very first action

#### Handling Safety Reminder Dialog

LinkedIn may show a "Job search safety reminder" dialog before the Easy Apply form opens. This dialog has a "Continue applying" **link** (not button). Dismiss it with:
```javascript
await page.getByRole('link', { name: 'Continue applying' }).click();
```

#### Workable-Powered Applications

Some Easy Apply jobs use Workable as the ATS. Key differences:
- Form headings may be in French (e.g. "Coordonnees" for Contact Info, "CV" for Resume, "Questions supplementaires" for Additional Questions)
- The Resume step may include a **Headline** textbox that needs filling with a summary of the user's professional title and key skills
- Text answer fields have character/validation limits. Keep answers under ~80 characters to avoid "Veuillez saisir une reponse valable" (invalid response) errors. Alerts may show as just `alert` + `img` tags with the error text in a sibling `generic` node
- Salary fields in Workable forms require plain numbers (e.g. `160000`), not formatted strings (e.g. `$150,000 - $180,000`). The error message is: "Enter a decimal number between 0.0 and 9.9999998E12"
- Search for `alert` tags in snapshots to find validation errors

#### Navigate to Jobs by Direct URL

Always navigate directly to jobs via their ID rather than clicking job cards in search results:
```
https://www.linkedin.com/jobs/view/<JOB_ID>/
```
Save all job IDs from a search page before navigating away, since search results change on reload.

#### Easy Apply Dialog Fallback

If clicking the Easy Apply link doesn't open a dialog, navigate directly to the apply URL:
```
https://www.linkedin.com/jobs/view/<JOB_ID>/apply/?openSDUIApplyFlow=true
```

#### Education Step (Select Dropdowns)

Some applications (e.g. Lever-powered) have an Education step with `<select>` dropdowns for School and Degree (not typeahead comboboxes). Use `browser_select_option` for these:
```javascript
browser_select_option ref="eXXX" values=["<Your University>"]
browser_select_option ref="eYYY" values=["<Your Degree>"]
```

#### Location Typeahead Combobox

Some forms have a Location combobox that is a typeahead (not a `<select>`). Fill it by typing and clicking the first suggestion:
```javascript
const locField = page.getByRole('combobox', { name: 'Location (city)' });
await locField.click();
await locField.fill('<Your City>');
await page.waitForTimeout(2000);
await page.locator('[role="option"]').first().click();
```

#### One-Page Applications

Some Easy Apply jobs have a single-page form with Contact + Resume + Follow checkbox + Submit all visible at once (0% progress). Just uncheck Follow and submit directly without clicking Next.

#### Form Filling Strategy

1. **Contact Info**: Pre-filled. Click Next.
2. **Resume**: Pre-selected (may have Headline field on Workable apps). Click Next.
3. **Work Experience**: Pre-filled (if present). Click Next.
4. **Education**: Pre-filled (if present). Click Next.

Note: Some forms have 5 steps (Contact, Resume, Work Experience, Education, Additional Questions) while others skip Work Experience/Education entirely. Just keep clicking Next through all pre-filled steps until you reach Additional Questions or Review.

5. **Additional Questions**: Answer based ONLY on what's in the resume. Re-read the resume if unsure.

   **BAIL-OUT RULE**: When filling Additional Questions, if you find yourself answering "0" or "No" to **3 or more** core skill/technology questions, the user is not qualified for this role. **Immediately dismiss the application dialog** (click Dismiss/X button) instead of continuing. Do not submit unqualified applications. Log it as "Skipped (not qualified)" in the results table and move to the next job.
   - "How did you hear about us?" -> "LinkedIn"
   - Work authorization -> Answer based on your citizenship/visa status
   - Years of experience -> Derive from resume dates/descriptions. If a skill isn't mentioned, answer "0".
   - LinkedIn profile URL -> Your LinkedIn profile URL
   - Location -> Your city, state/province, country
   - GitHub -> Your GitHub URL (if applicable)
   - Portfolio/Website -> Your portfolio URL (if applicable)
   - Technical questions -> Craft concise answers using ONLY skills and projects from the resume. Never fabricate.
   - Background check -> "Yes"
   - English proficiency -> "Native or bilingual" or "Bilingual / Native" (varies by form)
   - Expected hourly rate / total compensation -> Your target compensation
   - Visa sponsorship required -> Answer based on your work authorization status
   - Comfortable working remote -> "Yes"

6. **Review page**: Uncheck "Follow [Company]..." checkbox, then click Submit.
7. **Post-submit**: Dismiss the confirmation dialog (click Done or Dismiss), then proceed to next job.

### 4. Report Results

After reaching 20 applications (or exhausting all searches), present a summary table:

| # | Company | Role | Salary | Status |
|---|---------|------|--------|--------|
| 1 | Company | Title | $X | Applied |
| 2 | Company | Title | N/A | Skipped (staffing) |

Include both applied and skipped jobs so the user can see the full picture. Show the total applied count prominently.

## Playwright MCP Config Requirements

The following flags in `~/.claude.json` are required for context-efficient operation:
- `--snapshot-mode none` : Prevents inline snapshots on every MCP tool response
- `--block-service-workers` : Reduces background error noise
- `--blocked-origins chrome-extension://invalid/` : Blocks extension error source
- `--output-mode file` : Saves large outputs to file
- `--no-sandbox` : Required for WSL2 environments
- `--init-script ~/.claude/playwright/stealth.js` : Stealth script that hides webdriver flag and intercepts `chrome-extension://` fetch/XHR requests to prevent console error spam

If the MCP responses are bloated (>5KB for simple actions), check that these flags are set.

## Staffing Company Ban

**Skip ALL companies with "Staffing and Recruiting" as their LinkedIn industry type.** No exceptions. These are intermediaries, not the actual employer. Apply directly to hiring companies only.

## Known Blocklisted Companies (auto-skip)

Skip any listings from these specific companies even if they change their industry type:
- **Great Value Hiring** - Fake staffing agency, absurdly wide salary ranges
- **Crossing Hurdles** - Fake staffing agency, 2.5M+ fake followers, posts across wildly different domains
- **Hanalytica GmbH** - Staffing agency disguised as tech company
- **Doghouse Recruitment** - Staffing agency with disproportionate followers
- **StafinGo** - Staffing agency, 201-500 employees
- **Orbis Group** - Staffing agency, 51-200 employees, 432K disproportionate followers

## Customization

Before using this skill, update these placeholders with your own information:

- `<RESUME_PATH>` - Path to your resume file (markdown or text format)
- `<Your City>` - Your city for location typeahead fields
- `<Your University>` - Your university name for education dropdowns
- `<Your Degree>` - Your degree type for education dropdowns
- Salary thresholds in Step 2b to match your minimum acceptable compensation
- Target compensation values in the form filling strategy
- Search keywords in Step 1 to match your target roles and skills
- Add any companies you want to blocklist
