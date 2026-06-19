---
name: bcfood-bc-case-pilot
description: >
  Use this skill for any Aptean BCFood / Business Central support case.
  Identify the customer's Business Central version from the case, then
  investigate using the matching source repositories. Trigger on:
  BCFood, Business Central, BC18, BC18US, BC22, BC22US, BC24, BC24US,
  BC26, BC26US, BCFood_BC18, BCFood_BC22, BCFood_BC24,
  BCFood_BC26, error, exception, cannot see, missing, not working,
  permission denied, posting error, page error, codeunit, AL extension,
  customization, upgrade.
---

# BCFood Business Central — Version-Aware Case Skill

This skill investigates BCFood on Microsoft Dynamics 365 Business Central
support cases. The investigation path depends on which **BC version** the
customer is running, because the codebase, object IDs, and AL extensions
differ between versions. Always determine the version **first**, then route
to the correct local repositories.

All product repositories are pre-cloned on this agent at `/base/<repo-name>/`.
Use the Grep, Glob, and Read tools to search them directly — no credentials needed.

> **EXCLUSION RULE:** If the case is about **deploying app files** or
> **refreshing the database with the latest PROD database**, do **not** search
> the source repositories. These are environment/ops tasks — route directly to
> the DevOps or environment team and close the investigation here.

## Step 1 — Identify the customer's Business Central version

Read the case and extract the BC version from any of the following
(in priority order):

1. Explicit version field on the case (e.g. "BC Version: 22.1",
   "Platform: 24", "BC 18.2 US").
2. Customer environment metadata attached to the case.
3. Phrases in the case body: "BC18", "BC 18.2", "BC22", "BC 22.1",
   "BC24", "BC 24", "BC26", "BC 26.7", "Business Central 22",
   "v24", "version 26", "26.x", etc.
4. The country / localisation when relevant — BC 22, 24, and 26 have
   separate repos for US and CA, so look for hints like "Canadian customer",
   "CAD", "GST/HST", "US customer", "USD", "sales tax".
5. The extension name itself: anything containing `BC18US`, `BC22US`,
   `BC22CA`, `BC24US`, `BC24CA`, `BC26US`, or `BC26CA`
   (e.g. `BCFOOD_BC22_US`, `bcFood_BC18US`) indicates the matching version.

If the version cannot be determined, state that explicitly in the output
and ask the support agent to confirm the customer's BC version (and
country, for BC 22+) before deeper investigation.

## Step 2 — Route to the correct local repositories

Use the table below to pick the local paths to investigate.

| BC Version | Country | Local path |
| --- | --- | --- |
| **BC 18** | US only | `/base/bcFood_BC18US` |
| **BC 22** | US | `/base/BCFOOD_BC22_US` |
| **BC 22** | CA | `/base/BCFOOD_BC22_CA` |
| **BC 24** | US | `/base/BCFOOD_BC24_US` |
| **BC 24** | CA | `/base/BCFOOD_BC24_CA` |
| **BC 26** | US | `/base/BCFOOD_BC26_US` |
| **BC 26** | CA | `/base/BCFOOD_BC26_CA` |
| Other versions | — | Not covered by this skill — flag for triage |

### Notes on localisation routing

- **BC 18** has a single US-only repo (`bcFood_BC18US`). There is no BC 18 CA
  variant. If a case claims BC 18 CA, flag it for triage.
- **BC 22, 24, and 26** have both a US and a CA repo. Always confirm the
  customer's country before searching, because object IDs, tax logic, and
  field layouts may differ.
- If the country cannot be determined from the case, search the US repo first
  (US is the more common localisation) and note in the report that the CA repo
  was not checked.

### Verify the path exists before searching

Before searching, confirm the repo directory is present:

Glob: /base/<repo>/**/*.al  (just check a few results exist)

If the directory is missing, note it in the report and skip that repo.

## Step 3 — Investigate the case against the selected repos

For each repo selected in Step 2, use the following approach:

### 3a — Search for the failing object

Use the error message, page name, codeunit number, table ID, field name,
or report ID from the case to locate relevant `.al` files.

BCFood-specific areas to consider when narrowing the search:

- **Lot tracking / traceability** — lot numbers, item tracking, recall,
  forward/backward trace.
- **Recipe / formula management** — production BOM, routing, ingredient
  substitutions, yield.
- **Catch weight** — dual-unit items, catch weight postings, weight
  discrepancies.
- **Food safety & quality** — QC orders, certificates of analysis, hold
  management, inspection.
- **Pricing & contracts** — customer-specific pricing, contract pricing,
  price lists.
- **Warehouse & inventory** — bin management, directed put-away/pick,
  expiry date handling.
- **Financials** — posting groups, dimensions, landed cost, item charges.

Search strategies (use in order, stop when you find the relevant code):

1. **By object name or keyword** — grep for the exact error text, page
   caption, or procedure name:
   Grep pattern: "<keyword>" in /base/<repo>/**/*.al

2. **By object number** — if the case mentions a codeunit/table/page ID:
   Grep pattern: "Codeunit 12345" or "Table 12345" in /base/<repo>/**/*.al

3. **By field or variable name** — grep for the specific field or variable
   referenced in the stack trace or error:
   Grep pattern: "<FieldName>" in /base/<repo>/**/*.al

### 3b — Read the relevant procedure / trigger

Once the file is located, read the full procedure or trigger that raises
the error or controls the missing behaviour. Note:
- The procedure name and AL file path
- The last `git log -1` commit that touched this file:
  Bash: git -C /base/<repo> log -1 --oneline -- <file>

### 3c — Check recent changes

Look at the last 10 commits on the repo's default branch for regressions:
Bash: git -C /base/<repo> log -10 --oneline

If any commit touches the file or area identified in 3b, list them as
candidates for the regression.

### 3d — Cross-reference between US and CA repos (when both exist)

If a procedure in the US repo behaves differently from the CA repo, or the
case mentions localisation-specific behaviour, grep the other country's
repo for the same procedure name to compare implementations.

### 3e — Stay on the correct version

Never quote behaviour or commits from a repo that doesn't match the
customer's BC version — object IDs and signatures differ between BC 18,
22, 24, and 26.

## Step 4 — Classify the issue

Based on what you found:

- **Code defect** — a bug in the AL code; identify the file, procedure,
  and line range.
- **Configuration** — feature behaves correctly per code; customer setup
  (No. Series, posting groups, item tracking codes, lot templates,
  permissions) is the cause.
- **Upgrade artefact** — issue introduced by a recent BC upgrade or
  a recent commit in one of the version's repos.
- **Localisation** — behaviour differs between US and CA repos; confirm
  the customer is on the right country build.
- **Data** — bad data in the customer's tenant (orphaned lot entries,
  invalid item tracking state, corrupt recipe lines); code is working
  as designed.
- **Out of scope** — not covered by BCFood or the BC base; route to
  Microsoft or another product team.

## Output Format

Produce the report using **exactly** the fields below, in this order. Do
not add or remove fields. Keep each field tight and factual.

**Case Description:**
[One short paragraph describing what the customer reported, in your own
words. Include the BC version (e.g. "BC 22.1") and country (US/CA) if
relevant.]

**Any Setups (if applicable):**
[List any setup pages, configuration tables, posting groups, item tracking
codes, lot templates, permission sets, dimensions, number series, or
feature toggles that are involved in this scenario. If none apply, write
"N/A".]

**Steps to Replicate:**
1. [First action — page opened, record selected, field entered]
2. [Next action]
3. [Continue until the failure / observed behaviour is reached]

**Expected Result:**
[See the rules below for what to write here.]

**Actual Result:**
[See the rules below for what to write here.]

### Rules for "Expected Result" and "Actual Result"

Pick exactly one of the four categories below based on the Step 4
classification, and follow its rule:

1. **Code is working as designed** (no defect, behaviour matches the
   AL code and product intent):
   - **Expected Result:** `As Designed`
   - **Actual Result:** Briefly describe what the customer is seeing
     and why it is in fact the designed behaviour. Reference the file
     and procedure that defines this behaviour.

2. **Setup / configuration issue** (code is fine; customer's setup is
   wrong or missing):
   - **Expected Result:** `Setup Issue`
   - **Actual Result:** Name the specific setup that is wrong or
     missing and what value it should have.

3. **Enhancement request** (working as designed but the customer is
   asking for new behaviour, or the fix requires significant new
   development rather than a bug fix):
   - **Expected Result:** `Enhancement Request`
   - **Actual Result:** Summarise what the customer wants that the
     product does not currently do, and what change would be needed.

4. **Anything else — true defect, data issue, upgrade regression,
   localisation mismatch, out of scope, etc.**:
   - **Expected Result:** Describe normally what *should* happen
     according to the product design / specification.
   - **Actual Result:** Describe normally what is *actually* happening
     in the customer's environment, with evidence from the repo
     (file, procedure, last relevant commit if available).

### Footer (always include after the five main fields)

**BC Version & Repositories Investigated:**
- BC Version: [e.g. "BC 22.1 US", "BC 18 US", or "Unknown — please confirm"]
- Country: [US / CA / Unknown]
- Local Repos Searched: [list paths searched, or "Repo not found at /base/<name>"]

**Notes:** [anything skipped, anything that needs the support agent to confirm
with the customer — especially BC version or country; also note if the case
was excluded from repo investigation due to deploy/database-refresh scope]
