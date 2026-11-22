# Private Donor Monitor

A tracker of private-sector (corporate) donations to UN agencies, INGOs,
and NGOs — built to surface donations that aren't centrally logged
anywhere else. Published as a static GitHub Pages site styled as a daily
newspaper, with an optional weekly email digest of the highest-confidence
findings.

**This is a discovery aid, not an audited financial record.** Every entry
carries a confidence score and a list of what's uncertain about it — see
`docs/sources.html` (the "Sources & Limitations" page on the live site)
for what this can and can't catch.

---

## ⏸ Currently paused

Both scheduled workflows (`.github/workflows/daily.yml` and
`.github/workflows/weekly-email.yml`) have their `schedule:` trigger
commented out. In this state:

- Nothing runs automatically. No code executes on any schedule at all.
- No outbound requests are made to Google News, GDELT, Media Cloud, AWS
  Bedrock, Gemini, or any email provider — the only way any of this
  pipeline's code runs is by someone manually triggering a workflow from
  the **Actions** tab, and even the weekly email additionally requires
  `email.enabled: true` in `config/settings.yaml` (currently `false`).
- All stored secrets (API keys, the AWS role ARN) remain configured and
  unchanged. Pausing is purely about not *firing* — nothing has been
  revoked.

### To resume

1. Open `.github/workflows/daily.yml` and `.github/workflows/weekly-email.yml`.
2. In each, uncomment the `schedule:` block (remove the `#` from the
   `schedule:` line and the `- cron: "..."` line beneath it).
3. Commit and push. The next scheduled time in that cron expression is
   when it will next run automatically.
4. Before relying on the schedule, trigger each workflow manually once
   from the **Actions** tab (pick the workflow → **Run workflow**) to
   confirm everything still works — secrets can expire, API free tiers
   can change, and it's cheaper to find that out by hand than to
   discover it silently failed overnight.

If this repository has moved to a different AWS account or a different
GitHub owner/repo path since it was paused, read **AWS Bedrock setup**
below before resuming — the AWS role trust relationship is tied to the
exact repo path and will need to be re-created, not just re-enabled.

---

## How it works

Each run of the daily pipeline (`scripts/run_daily.py`):

1. **Fetches candidate news from four independent sources**, each
   covering gaps the others miss:
   - **Google News** — one search query per monitored recipient, built
     from the recipient's name plus a donation-language trigger phrase.
   - **GDELT** (Global Knowledge Graph) — a broad, multilingual, near-
     real-time bulk feed, scanned via downloaded files rather than a
     live search API.
   - **Media Cloud** (mediacloud.org) — a structured news search API
     over a curated set of ~1,600 English-language sources.
   - **PR wire feeds** — direct company press-release feeds (e.g. PR
     Newswire, Business Wire), which catch company-published
     announcements that general news coverage sometimes skips.
2. **Filters** out obviously irrelevant results with a cheap keyword
   pass against `config/trigger_phrases.txt` (a smaller, separate list
   for Media Cloud specifically — see that file's own notes on why).
3. **Clusters** near-duplicate articles describing the same event
   together, so five outlets covering one donation become one AI call,
   not five.
4. **Extracts structured facts** (donor, recipient, amount, in-kind
   description, country/crisis scope, etc.) for each event cluster using
   an AI model — see **AI extraction** below for which one and why.
5. **Scores a confidence value (0–10)** per finding, and runs it through
   a series of rejection checks before publishing — see **Rejection
   checks** below.
6. **Checks each surviving finding against everything published in the
   lookback window** (`dedupe_lookback_days` in `settings.yaml`, default
   180 days), to catch exact duplicates and flag likely renewals of a
   previously-reported relationship.
7. **Publishes** the day's surviving findings as a new page on the site
   (`docs/`), highest-confidence first.
8. **Once a week** (if enabled), emails a digest of the best recent
   findings to a configured recipient list.

### AI extraction

The pipeline supports two interchangeable AI providers, chosen by
`ai.provider` in `config/settings.yaml`:

- **`bedrock`** (the current default) — calls Amazon Nova Pro via AWS
  Bedrock's Converse API. No per-day request ceiling; cost is billed
  per token. See **AWS Bedrock setup** below for how this authenticates.
- **`gemini`** — calls Google's Gemini API directly with an API key.
  Has a hard free-tier daily request cap (`max_ai_calls_per_run` in
  `settings.yaml` exists specifically so a busy news day can't silently
  exceed it mid-run).

Switching providers is a one-line change (`ai.provider`) plus whichever
credential that provider needs (see **One-time setup**).

### AWS Bedrock setup

When `ai.provider` is `bedrock`, the daily workflow authenticates to AWS
using **GitHub's OIDC identity provider** — there is no long-lived AWS
secret stored anywhere. On each run:

1. GitHub mints a short-lived OIDC token identifying this exact
   workflow run (repo, branch, and workflow path included as claims).
2. AWS's `aws-actions/configure-aws-credentials` action exchanges that
   token for temporary credentials, scoped to one IAM role
   (`role-to-assume` in `daily.yml`).
3. That role's permissions are limited to invoking the specific Bedrock
   model configured in `settings.yaml`.
4. The temporary credentials expire when the job ends.

This means **the IAM role itself must exist in an AWS account that
trusts this specific GitHub repository**, via an IAM role trust policy
condition matching this repo's `owner/repo` path (and typically branch).
If this repository changes owner (a fork, a transfer to a different
GitHub account/org) or the AWS account access changes, the existing role
ARN in `daily.yml` will stop working — AWS will reject the OIDC token
because the trust condition no longer matches. To fix that:

1. In the AWS account that should run this, create (or update) an IAM
   OIDC identity provider for `token.actions.githubusercontent.com`
   (skip if one already exists for GitHub Actions in that account).
2. Create an IAM role with a trust policy scoped to
   `repo:<owner>/<repo>:*` (or a narrower branch condition) for that
   OIDC provider.
3. Attach a permissions policy granting `bedrock:InvokeModel` (and
   `bedrock:InvokeModelWithResponseStream` if ever needed) on the
   specific model/inference-profile ARN you intend to use.
4. On the Bedrock console, in the region you're using, open **Model
   access** and enable the model referenced by `ai.bedrock_model` in
   `settings.yaml`. Amazon's own first-party models (like Nova) are
   usually instant self-service; third-party models may require a
   separate AWS Marketplace subscription approval.
5. Update `role-to-assume` in `.github/workflows/daily.yml` to the new
   role's ARN, and `ai.bedrock_region` / `ai.bedrock_model` in
   `settings.yaml` if the region or model changed.
6. Bedrock's Converse API sometimes requires a *cross-region inference
   profile ID* rather than a bare foundation-model ID (the model ID is
   prefixed with a region code, e.g. `eu.amazon.nova-pro-v1:0`) — if a
   run fails with an error about on-demand throughput not being
   supported, check the Bedrock console's cross-region inference page
   for the correct profile ID string.

No AWS setup is needed at all if using `ai.provider: "gemini"` instead —
just a `GEMINI_API_KEY` secret.

### Confidence scoring

Each finding is scored 0–10 from four weighted factors (weights live in
`confidence:` in `settings.yaml`, editable without touching code):
source tier (official/general-news/etc.), whether an explicit amount or
figure is stated, how many independent sources corroborate it, and
whether the recipient is one of the curated, actively-monitored
organizations in `config/recipients.txt`.

### Rejection checks

Before anything is published, every extracted event passes through a
series of backstop checks (implemented in `src/extraction.py`, applied
in `scripts/run_daily.py`), each guarding against a distinct way an AI
extraction can be technically well-formed but not a real, usable
finding:

- Neither side of the donation may be a government or sovereign-state
  entity (including a bare country name as the donor) — this tracker
  is scoped to *private-sector* giving only.
- The extracted donor must not itself be one of the monitored recipient
  organizations (a sign the model mixed up which side gave and which
  side received).
- Both donor and recipient must be actually named — a generic
  description ("a company", "a local charity", a bare job title like
  "CEO") doesn't count, including when the model's own stated
  assumptions admit a name is missing or uncertain.
- The recipient's country/crisis scope must be on the monitored list,
  *or* the finding must otherwise be anchored to a real, specific place
  even without a formally named recipient organization.
- There must be some actual evidence a donation happened — a stated
  amount, a quoted figure, or an in-kind description — not just two
  named entities mentioned near each other.

## Config files (no code changes needed)

Everything you're likely to want to adjust lives in `config/` as plain
text files with instructions written at the top of each one:

| File | What it controls |
|---|---|
| `config/recipients.txt` | Which UN agencies, INGOs, and NGOs to watch for incoming donations |
| `config/countries.txt` | Which countries/crises count as "in scope" geographically |
| `config/trigger_phrases.txt` | The words/phrases used to spot donation-related stories (general sources) |
| `config/mediacloud_trigger_phrases.txt` | A smaller, higher-precision subset of the above, used only for Media Cloud's query |
| `config/pr_wire_feeds.txt` | Which press-release wire feeds get checked alongside Google News |
| `config/settings.yaml` | Source toggles, confidence scoring weights, AI provider/model, email settings |

To check a config file after editing it, without running the whole
pipeline:
```bash
python src/config_loader.py
```
This either prints `OK: ...` or names the exact file and line with a
problem.

You should not need to touch anything in `src/`, `templates/`, or
`scripts/` for routine content changes.

---

## One-time setup (for whoever is standing this up fresh)

### 1. Choose an AI provider and get credentials

- **Bedrock (recommended, no daily request cap):** follow **AWS Bedrock
  setup** above to create the IAM role, then set `role-to-assume` in
  `daily.yml` to its ARN. No GitHub secret is needed for Bedrock itself
  — authentication is via OIDC, not a stored key.
- **Gemini (simpler, free tier, daily cap applies):** get a free key at
  [Google AI Studio](https://aistudio.google.com/app/apikey), add it as
  the `GEMINI_API_KEY` repository secret, and set `ai.provider: "gemini"`
  in `settings.yaml`.

### 2. Get a Media Cloud API key (optional)

Free key at [search.mediacloud.org](https://search.mediacloud.org). Add
it as the `MEDIACLOUD_API_KEY` repository secret. Leave
`search.mediacloud_enabled: false` in `settings.yaml` until the secret is
actually set — enabling it without a key fails the run immediately with
a clear error rather than failing silently.

GDELT, Google News, and the PR wire feeds need no credentials at all.

### 3. Add repository secrets

**Settings → Secrets and variables → Actions → Secrets**:
- `GEMINI_API_KEY` — only if using the Gemini provider
- `MEDIACLOUD_API_KEY` — only if Media Cloud is enabled
- `RESEND_API_KEY` — only if using Resend for the weekly email
- `SMTP_PASSWORD` — only if using SMTP (e.g. Gmail) for the weekly email

**Settings → Secrets and variables → Actions → Variables**:
- `SITE_URL` — your GitHub Pages URL once it's live (step 4), e.g.
  `https://yourorg.github.io/donation-tracker`

### 4. Turn on GitHub Pages

**Settings → Pages** → Source: "Deploy from a branch" → Branch: `main`,
folder: `/docs`. Save. The site will appear at the URL GitHub shows
there — put that URL into the `SITE_URL` variable above.

### 5. Set up the weekly email (optional)

Edit `config/settings.yaml`:
```yaml
email:
  enabled: true
  recipients: "person1@org.org, person2@org.org"
  method: "resend"          # or "smtp"
  from_address: "digest@yourdomain.org"
```
**Resend** (recommended — simple, generous free tier): sign up at
[resend.com](https://resend.com), verify a sending domain, create an API
key, add it as `RESEND_API_KEY`.

**SMTP** instead (e.g. a Gmail account): fill in `smtp_host`,
`smtp_port`, and `smtp_username` in `settings.yaml`, and add the SMTP
password — for Gmail, an *app password* (requires 2-factor
authentication enabled on the account), not the account's normal
password — as the `SMTP_PASSWORD` secret.

### 6. Uncomment the schedules and test manually first

Follow **To resume** at the top of this file. Trigger each workflow
manually from the **Actions** tab before trusting the schedule.

---

## Running it locally (development/testing)

```bash
pip install -r requirements.txt

# Check your config files are valid:
python src/config_loader.py

# Full dry run with fake data — no API calls or internet needed at all:
python scripts/run_daily.py --mock --mock-fetch

# Real fetching (all enabled sources) but fake AI extraction — good for
# checking your recipients/triggers/countries lists are actually finding
# real candidate articles, without spending AI quota/cost:
python scripts/run_daily.py --mock

# Full live run (needs whichever AI provider's credentials are configured,
# e.g. AWS credentials in the environment for Bedrock, or GEMINI_API_KEY
# for Gemini):
python scripts/run_daily.py

# Preview the weekly email without sending it:
python scripts/run_weekly_email.py --dry-run
```

After any run, open `docs/index.html` in a browser to see the site.

---

## Repository layout

| Path | Contents |
|---|---|
| `scripts/run_daily.py` | Entry point for the daily pipeline |
| `scripts/run_weekly_email.py` | Entry point for the weekly digest email |
| `scripts/wipe_today.py` | Removes only today's data, for a clean manual re-run |
| `src/sources.py`, `query_builder.py`, `gdelt_gkg.py`, `mediacloud_source.py` | The four fetch sources |
| `src/clustering.py` | Trigger-phrase filtering and near-duplicate clustering |
| `src/extraction.py` | AI prompt, response parsing, confidence scoring, rejection checks |
| `src/store.py` | Cross-run duplicate/renewal detection against publishing history |
| `src/site_builder.py` | Renders `docs/` from `templates/` and the day's findings |
| `config/` | All editable, non-code configuration (see table above) |
| `templates/` | Jinja2 HTML templates for the site |
| `data/` | Published edition JSON files and the dedupe store — the pipeline's own durable state |
| `docs/` | The generated static site (what GitHub Pages serves) |

## Design notes (for whoever maintains the code)

- **Recipient-anchored search, not a donor list.** A donor could be
  "anyone, anywhere," so every search query is built from the known,
  stable list of monitored organizations rather than a pre-built company
  list. Previously-unknown donors surface naturally because the query is
  anchored on who they gave *to*, not who they are.
- **Clustering is a cost optimization, not the source of truth on
  duplicates.** Fuzzy title matching groups same-day articles about one
  event to avoid redundant AI calls, but it's an imperfect heuristic.
  The real duplicate/renewal detection in `store.py` compares actual
  extracted facts (donor, recipient, amount, timing) against publishing
  history, which catches duplicates even when headlines look nothing
  alike, and correctly tells apart "the same donation reported twice"
  from "a second, separate donation from the same recurring donor."
- **Copyright-safe extraction.** The AI prompt requires a paraphrased
  summary plus, separately, only a short (under 15 words) verbatim
  clause for the specific figure cited — never a reproduced sentence or
  paragraph from a news source.
- **"Unspecified / global" scope is tagged explicitly, never guessed.**
  A donation to an agency's general/unearmarked fund often doesn't name
  a country at all. Rather than have the AI infer a likely country from
  context, these are tagged "Unspecified / global" and still published
  at a correspondingly lower confidence score.
- **The AI call budget is a hard constraint.** `max_ai_calls_per_run` in
  `settings.yaml` caps AI calls per run regardless of provider, so a
  high-volume news day degrades gracefully (processing the
  highest-priority candidates first) rather than failing partway through
  or running away in cost.

## Known limitations

See also the live "Sources & Limitations" page (`docs/sources.html`).

- Under-catches donations that are only announced through a channel
  none of the four sources cover — for example, an organization's own
  financial-tracking or donor-acknowledgment page, or a social-media-only
  post, rather than conventional news coverage or a press release.
- Fuzzy clustering/dedup is approximate; very occasionally the same
  event may be published twice if wording and reported facts both
  diverge enough between outlets.
- Confidence scores reflect source strength and corroboration, not legal
  or financial verification — always check the linked source before
  citing a figure externally.
