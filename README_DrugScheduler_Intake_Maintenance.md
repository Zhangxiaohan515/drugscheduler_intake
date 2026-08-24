# DrugScheduler Intake — Maintenance & Deployment Guide

## 1. Purpose of this interface

`index.html` is the public user-intake interface for the DrugScheduler project.

Its purpose is to collect a compact, user-friendly set of information that may later support personalized medication/supplement scheduling.

The current interface collects five main layers of information:

1. **Current Stack**
   - medications
   - vitamins/minerals
   - gut/digestive products
   - sleep/stress products
   - performance/cognition products
   - longevity/other supplements
   - frequency
   - rough timing
   - dose, when provided
   - product importance

2. **Daily Routine**
   - wake time
   - bedtime
   - number of meals
   - meal times
   - routine consistency
   - manageable number of intake events per day

3. **Main Goals**
   - general health
   - sleep/stress
   - focus/cognition
   - energy/performance
   - gut/immune health
   - healthy aging/longevity

4. **Relevant Context**
   - stomach sensitivity
   - fasting / skipped meals
   - common food exposures
   - caffeine timing
   - a limited set of condition/life-stage follow-up questions

5. **Scheduling Preferences**
   - preferred time of day
   - meal relationship preference

The interface ends with a **Review** step and then stores the response in Supabase.

---

## 2. Current architecture

The application uses:

```text
GitHub Pages
    ↓
index.html
    ↓
Supabase JS client
    ↓
Supabase Anonymous Authentication
    ↓
Row Level Security (RLS)
    ↓
public.intake_responses
    ↓
Admin collector view (?admin=1)
    ↓
CSV / JSON export
```

### GitHub Pages

GitHub Pages hosts only the front-end application.

It contains:

- HTML
- CSS
- JavaScript
- questionnaire logic
- Supabase public configuration

GitHub does **not** store questionnaire responses.

### Supabase

Supabase stores respondent data.

Main table:

```text
public.intake_responses
```

Important fields include:

```text
owner_user_id
respondent_id
status
stage
step_index
started_at
updated_at
completed_at
response_json
```

The complete questionnaire response is stored in:

```text
response_json
```

with a structure similar to:

```json
{
  "stack": [],
  "routine": {},
  "goals": [],
  "context": {},
  "prefs": {}
}
```

This JSON structure is intentionally flexible so the questionnaire can continue changing without requiring a database migration every time.

---

## 3. Respondent identity

Each respondent has two different identifiers.

### Respondent UUID

The front end creates a UUID using:

```js
crypto.randomUUID()
```

and stores it in browser `localStorage`:

```text
ds_respondent_uuid
```

This helps the same browser return to the same questionnaire session.

### Supabase Auth UID

Supabase also creates an anonymous authenticated user.

This is used for security and Row Level Security.

Conceptually:

```text
respondent_id
= continuity / research identifier

auth.uid()
= security identifier
```

These values are expected to be different.

---

## 4. Admin access

Admin collector URL:

```text
https://<github-username>.github.io/<repo-name>/?admin=1
```

Current admin email:

```text
xiaohz@umich.edu
```

### Important security rule

Do **not** store the admin password in:

- README
- GitHub repository
- `index.html`
- JavaScript
- public notes
- commit history

Keep the admin password in a private password manager or another private location.

The admin account must also be listed in:

```text
public.intake_admins
```

The admin UID should match the UID shown in:

```text
Supabase
→ Authentication
→ Users
```

To check admin records:

```sql
select * from public.intake_admins;
```

To check the corresponding authentication user:

```sql
select id, email
from auth.users;
```

To add an admin safely:

```sql
insert into public.intake_admins(user_id)
values ('YOUR_ADMIN_AUTH_USER_UUID')
on conflict do nothing;
```

---

## 5. Supabase configuration in `index.html`

Search inside `index.html` for:

```js
window.DS_SUPABASE_CONFIG
```

The configuration should look like:

```js
window.DS_SUPABASE_CONFIG = {
  url: 'https://YOUR_PROJECT_REF.supabase.co',
  publishableKey: 'sb_publishable_YOUR_PUBLIC_KEY'
};
```

This is JavaScript inside the HTML file.

It is **not a Python script**.

The important object is:

```js
window.DS_SUPABASE_CONFIG
```

---

## 6. Where to find the Supabase Project URL

In Supabase:

```text
Project Dashboard
→ project home page
```

The Project URL appears near the project name and looks like:

```text
https://xxxxxxxxxxxxxxxx.supabase.co
```

It can also be found under the project's API / connection settings.

---

## 7. Where to find the Supabase Publishable Key

In Supabase:

```text
Project
→ Settings
→ API Keys
```

Use the **Publishable key**, which normally looks like:

```text
sb_publishable_...
```

This key is intended for use in browser-side applications when Row Level Security is enabled.

It may appear in `index.html`.

---

## 8. Keys that must NEVER be placed in GitHub

Never place any of the following inside `index.html`, README, or the public repository:

```text
service_role key
secret key
sb_secret_...
database password
private database connection credentials
```

These are privileged credentials.

Only the following front-end values should be present in the public HTML:

```text
Project URL
Publishable key
```

Security for respondent data is enforced through:

```text
Supabase Authentication
+
Row Level Security
```

---

## 9. Supabase tables and policies

The initial database was created using:

```text
supabase_intake_setup.sql
```

Main objects:

```text
public.intake_responses
public.intake_admins
public.is_intake_admin()
```

RLS behavior is intended to be:

### Respondent

```text
INSERT own response    allowed
SELECT own response    allowed
UPDATE own response    allowed
SELECT other users     denied
DELETE                 denied
```

### Admin

```text
SELECT all responses   allowed
```

Before changing RLS policies, make a backup of the SQL setup.

---

## 10. Admin collector

Open:

```text
https://<site-url>/?admin=1
```

After admin login, the collector shows:

- total respondents
- completed respondents
- in-progress respondents
- average layer reached
- respondent status
- product count
- goal count
- detailed questionnaire responses

Buttons:

```text
Export CSV
Export JSON
Refresh
Sign out
```

### CSV

Use CSV for:

- Excel
- pandas
- R
- quick analysis

### JSON

Use JSON as the more complete raw archive because it preserves nested fields.

Recommended practice:

```text
JSON = raw archive
CSV  = analysis/export format
```

---

## 11. How to maintain and update the interface

For normal questionnaire/UI updates, the main file to modify is:

```text
index.html
```

Typical changes include:

- adding/removing a question
- changing wording
- changing buttons
- modifying product categories
- adjusting brand lists
- adding conditional follow-up questions
- changing layout or styles

The current UI uses JavaScript constants such as:

```js
CATEGORIES
ITEM_BRANDS
GOALS
FOOD_EXPOSURE
CONDITION_CATS
```

and rendering functions for each layer.

Avoid changing the Supabase storage/authentication sections unless the backend architecture itself needs to change.

---

## 12. Normal update workflow

For most future interface changes, the workflow is simply:

```text
1. Modify index.html
2. Test locally
3. Upload / commit the new index.html to GitHub
4. GitHub Pages automatically redeploys
5. Existing Supabase responses remain stored
```

If using Git locally:

```bash
git add index.html
git commit -m "Update DrugScheduler intake interface"
git push
```

GitHub Pages will deploy the latest `index.html` from the configured branch.

---

## 13. Important: replacing `index.html` does NOT delete Supabase data

The questionnaire front end and response database are separate.

Therefore:

```text
Updating index.html
≠ deleting historical responses
```

Historical data remains in:

```text
Supabase
→ public.intake_responses
```

unless the database rows are manually deleted.

---

## 14. When a database update is actually needed

A Supabase schema change is normally **not** needed when:

- adding a new optional question
- changing wording
- changing a button
- adding a new field inside `response_json`
- changing UI layout

A database/schema update may be needed if:

- creating a new relational table
- changing RLS/security behavior
- adding required database columns
- changing admin permissions
- normalizing JSON fields into structured tables

For the current prototype stage, keeping questionnaire answers inside `response_json` is intentional.

---

## 15. How to test before deploying an update

Before replacing the production `index.html`, test locally.

Example:

```bash
cd <folder-containing-index.html>
python3 -m http.server 8000
```

Then open:

```text
http://localhost:8000/
```

Test:

1. add a product
2. continue through all layers
3. submit
4. confirm the response appears in Supabase
5. open:

```text
http://localhost:8000/?admin=1
```

6. verify admin access
7. verify CSV / JSON export

Only then upload the updated `index.html`.

---

## 16. GitHub Pages deployment

The repository should contain:

```text
index.html
README.md
supabase_intake_setup.sql   (optional but recommended)
```

GitHub Pages configuration:

```text
Settings
→ Pages
→ Deploy from a branch
→ main
→ /(root)
```

Public intake URL:

```text
https://<github-username>.github.io/<repo-name>/
```

Admin URL:

```text
https://<github-username>.github.io/<repo-name>/?admin=1
```

---

## 17. If the website says “Supabase is not configured yet”

Check `index.html`.

Search:

```js
window.DS_SUPABASE_CONFIG
```

Make sure the placeholders have been replaced:

```js
url: 'https://YOUR_PROJECT_REF.supabase.co'
publishableKey: 'sb_publishable_...'
```

Then save and redeploy.

---

## 18. If final submission says “Saved locally — sync pending”

Possible causes include:

- internet connection problem
- Supabase auth session problem
- RLS policy problem
- invalid Supabase configuration
- old browser test state during development

Check the browser Console first.

For development/test environments, local respondent state can be cleared with:

```js
localStorage.removeItem('ds_respondent_uuid');
localStorage.removeItem('ds_intake_draft');
localStorage.removeItem('ds_supabase_respondent_auth');

Object.keys(localStorage)
  .filter(k => k.startsWith('ds_pending_'))
  .forEach(k => localStorage.removeItem(k));
```

Do not routinely clear real users' local data.

---

## 19. Useful Supabase checks

### View responses

```sql
select
  respondent_id,
  status,
  stage,
  updated_at,
  completed_at
from public.intake_responses
order by updated_at desc;
```

### Count responses

```sql
select
  status,
  count(*)
from public.intake_responses
group by status;
```

### Check admins

```sql
select * from public.intake_admins;
```

### Match admins to emails

```sql
select
  u.id,
  u.email,
  case when a.user_id is not null then true else false end as is_admin
from auth.users u
left join public.intake_admins a
  on u.id = a.user_id;
```

---

## 20. Backup recommendation

Periodically export:

```text
CSV
JSON
```

from the admin collector.

For important research milestones, also keep:

```text
date-stamped JSON raw export
+
date-stamped CSV analysis export
```

Example:

```text
responses_2026-08-24_raw.json
responses_2026-08-24_analysis.csv
```

---

## 21. Practical maintenance rule

For ordinary future updates, remember this:

```text
UI / questions change
        ↓
edit index.html
        ↓
test locally
        ↓
upload / push new index.html to GitHub
        ↓
GitHub Pages redeploys automatically
        ↓
Supabase data remains intact
```

In most cases, **you only need to update `index.html`**.

Do not edit Supabase tables, RLS, or authentication settings unless the backend architecture itself needs to change.

---

## 22. Security reminder

Because this interface may collect medication, supplement, routine, and health-context information:

- do not expose admin passwords
- do not expose Supabase secret/service-role keys
- keep the repository limited to public front-end credentials only
- periodically export/back up response data
- review institutional privacy / research requirements before large-scale or formal human-subject data collection

---

## Quick reference

### Public interface

```text
https://<github-username>.github.io/<repo-name>/
```

### Admin interface

```text
https://<github-username>.github.io/<repo-name>/?admin=1
```

### Admin email

```text
xiaohz@umich.edu
```

### Admin password

```text
Stored privately — DO NOT COMMIT TO GITHUB
```

### Front-end Supabase config

```js
window.DS_SUPABASE_CONFIG = {
  url: 'https://YOUR_PROJECT_REF.supabase.co',
  publishableKey: 'sb_publishable_YOUR_KEY'
};
```

### Main response table

```text
public.intake_responses
```

### Main file to update

```text
index.html
```
