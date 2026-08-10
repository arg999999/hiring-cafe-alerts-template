# hiring.cafe job alerts

Get emailed when new [hiring.cafe](https://hiring.cafe) jobs match your saved search -- on a schedule, for free, with **no server and no database**. It runs on GitHub Actions and sends from your own Gmail.

## How it works

A GitHub Actions cron runs `check.mjs` every 10 minutes. It scrapes the current hiringcafe.com build, fetches the results for your saved filter (`searchState.json`), diffs the job IDs against `seen.json`, and emails you only the new ones. The updated `seen.json` is committed back so state survives between runs.

The **first run is silent by design**: it records everything currently matching and sends no email, so you don't get a huge blast on day one. Alerts start from the second run onward.

## Setup (about 10 minutes)

### 1. Create your own copy

Click the green **Use this template** button at the top of this repo, then **Create a new repository**. Make it **public** -- public repos get unlimited free Actions minutes; a private repo only includes 2,000 minutes/month, which 10-minute polling can exceed.

### 2. Create a Gmail App Password (free)

Sending happens through your own Gmail over SMTP -- no third-party service, no subscription.

1. Turn on **2-Step Verification**: https://myaccount.google.com/security
2. Create an **App Password**: https://myaccount.google.com/apppasswords -- name it e.g. "hiring alerts". Google gives you a 16-character code like `abcd efgh ijkl mnop`.
3. That code is what the script logs in with -- **not** your normal Gmail password. (Spaces are fine; the script strips them.)

### 3. Add repository secrets

In your new repo: **Settings -> Secrets and variables -> Actions -> New repository secret**. Add:

| Secret | Value |
| --- | --- |
| `GMAIL_USER` | the Gmail address that sends the alerts |
| `GMAIL_APP_PASSWORD` | the 16-character App Password from step 2 |
| `MAIL_TO` | where alerts go (optional; defaults to `GMAIL_USER`; comma-separate for several) |

### 4. Set your filter

Open [hiringcafe.com](https://hiringcafe.com), build the search you want, then open **DevTools -> Network** and find the `index.json?searchState=...` request. Copy the decoded `searchState` value and paste it into `searchState.json` in your repo, replacing the example. That is your filter -- location, departments, keywords, experience, and so on.

### 5. Turn it on

Go to the **Actions** tab, enable workflows if prompted, then open **hiring.cafe job alerts -> Run workflow**. Leave **Dry run** checked for a no-email test (it lists matches in the log), or uncheck it for a real run. After the first real run, the 10-minute cron takes over automatically.

## Optional: skip jobs that require a tech you don't want

`check.mjs` can drop jobs whose **requirements** demand a tech you won't use, while keeping jobs where that tech is only an alternative (for example "Node/Python", or "X or Y"). Edit the `EXCLUDED_TECH` list near the top of `check.mjs`:

```js
const EXCLUDED_TECH = [ /\bjava\b/i, /\bpython\b/i, /\.net\b/i, /\bangular(?:\.?js)?\b/i ];
```

Leave it as `[]` to disable this filter. Note: it reads the AI-summarized requirements text and treats "/" and "or" as "optional", so it is a good heuristic but not perfect.

## Tuning

| Want to... | Do this |
| --- | --- |
| Change how often it checks | Edit the `cron` in `.github/workflows/job-alerts.yml` (UTC). Default is every 10 minutes. |
| Change the filter | Re-copy `searchState` from DevTools into `searchState.json`. |
| Reset alerts (re-seed) | Set `seen.json` back to `{ "ids": [] }` and commit. |
| Preview without sending | Run the workflow with **Dry run** checked. |

## Good to know

GitHub's scheduled runs are best-effort -- under load they can be delayed or occasionally skipped, so "every 10 minutes" is a target, not a guarantee. For precise polling, run `check.mjs` from an always-on machine's cron instead; the script is identical, only where it runs changes.

hiringcafe.com is behind Cloudflare and its internal data route is undocumented, so it can change. `check.mjs` re-scrapes the build ID each run and retries with backoff, which makes it self-heal across most site changes. If runs ever start coming back empty, re-copy your `searchState` from DevTools.

## Files

| File | Purpose |
| --- | --- |
| `check.mjs` | The whole thing: fetch, diff, email via Gmail. |
| `searchState.json` | Your decoded hiring.cafe filter. |
| `seen.json` | Job IDs already alerted on. Committed back each run. |
| `.github/workflows/job-alerts.yml` | The cron, run, and commit workflow. |
| `.env.example` | Template for local testing only. |
