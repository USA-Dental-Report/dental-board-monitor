# Dental Board Newsroom Monitor

Checks state dental board news pages once a week and emails you (via SMTP) when one changes. Built for USA Dental Report.

## How it works

1. `data/sites.json` lists the pages to watch: 34 entries covering 31 states plus the District of Columbia, Guam and the Northern Mariana Islands.
2. Every Monday at 13:00 UTC, a GitHub Action (`.github/workflows/check.yml`) runs `scripts/check_sites.py`. GitHub often starts scheduled runs a few hours late. The script:
   - Fetches each page (with a browser-like user agent, since several state sites block obvious bots)
   - Removes scripts, styles, nav, header, footer and forms, keeping just the readable text
   - Hashes that text and compares it to the stored hash in `data/state/<slug>.json` (the text itself is saved in `data/state/<slug>.txt`)
   - If the hash changed, pulls out up to 15 added lines as a lightweight diff
   - If the fetch fails, keeps the last good snapshot and counts the failure
3. If any page changed, or any site has failed **3 runs in a row**, it sends **one summary email** listing every changed site (with its new lines) and every site that is no longer being checked. If neither happened, no email is sent.
4. The updated state files are committed back to `main` ("Update site state [skip ci]") so the next run has something to compare against.

The first successful fetch of a site only saves a baseline (no email). You'll be alerted on later runs when something changes.

## Setup

In the repo's **Settings → Secrets and variables → Actions**, add these repository secrets:

| Secret | Required | Notes |
|---|---|---|
| `SMTP_HOST` | yes | SMTP server hostname, e.g. `smtp.gmail.com` |
| `SMTP_FROM_EMAIL` | yes | "From" address; must be allowed by your SMTP provider |
| `ALERT_TO_EMAIL` | yes | Where alerts go; separate several addresses with commas |
| `SMTP_USERNAME` | usually | SMTP login. Login is skipped if this or the password is missing |
| `SMTP_PASSWORD` | usually | SMTP password (for Gmail, use an App Password) |
| `SMTP_PORT` | no | Defaults to `587` (STARTTLS); use `465` for implicit SSL |
| `SMTP_FROM_NAME` | no | Defaults to "USA Dental Report Board Monitor" |

If any required secret is missing, the job still checks every site and saves the results, but it skips the email. The run log then says `Not sending email: ... is not set`.

To test it, open the **Actions** tab, choose **Check dental board sites**, and click **Run workflow**. An email only goes out if something changed or a site has been failing, so a quiet week sends nothing.

## Adding/editing sites

Edit `data/sites.json` directly. Each entry looks like:

```json
{
  "state": "Alabama",
  "board_name": "Board of Dental Examiners of Alabama",
  "url": "https://dentalboard.org/news-and-announcements/",
  "slug": "alabama"
}
```

`slug` must be unique and filesystem-safe (it's used for the state file names). A new site's first run is a silent baseline.

Optional per-site keys:

- `headers`: a dict of extra request headers for that site only.
- `allow_unverified_tls`: `true` lets the script retry without certificate checking, but only when the normal request fails with an SSL error. This is meant for boards whose servers send an incomplete certificate chain (currently North Carolina). The run log marks these fetches `[INSECURE]`.
- `note`: free text for your own reference; the script ignores it.

## Not monitored

These 19 states have no entry in `data/sites.json`: Alaska, Delaware, Idaho, Indiana, Kansas, Kentucky, Louisiana, Maine, Missouri, Montana, Nevada, Ohio, Oklahoma, Rhode Island, South Carolina, South Dakota, Texas, Vermont, Wyoming. Add them once you find a page worth watching.

## Current site problems (as of November 15, 2026)

**Failing to load (not being monitored):**

- Massachusetts: blocked (HTTP 403) on every run since it was added; it has never been checked.
- New York: certificate verification fails (`unable to get local issuer certificate`). Probably the same incomplete-chain issue as North Carolina.
- New Hampshire, Arizona, Tennessee: blocked (HTTP 403) by bot protection.
- Alabama, Florida, Pennsylvania, Wisconsin: timed out on the latest run. Probably temporary.

**Loading, but not capturing real news:**

- Minnesota: the saved text is a CAPTCHA page.
- Maryland, New Jersey, West Virginia: the saved text is empty, likely because the page builds its content with JavaScript.
- Georgia: the page only shows `No news for topic ""`.
- Oregon: the saved text is mostly an "out-of-date browser" banner.
- Pennsylvania: the URL is `pacast.com`, which only captures a mailing address. It needs a better URL.
- Nebraska: the URL is a 2022 PDF newsletter, so it will never change. It needs a news page instead.

You can check any site yourself by opening its `data/state/<slug>.txt`. Its `.json` file shows `consecutive_failures` and `last_error`.

## Known limitations

- **JavaScript-rendered pages**: the script uses plain `requests`, not a browser, so pages that load their content with JavaScript come back empty or partial. Fixing those needs a headless-browser fetch (e.g. Playwright) for just those sites.
- **Bot blocking**: some state sites (Akamai/Cloudflare/Radware-protected) reject automated requests with a 403 or a CAPTCHA. A failure is logged and counted, but it doesn't stop the other sites from being checked.
- **Layout-only changes**: if a page is redesigned without new announcements, you may get a "changed" alert with no clear added lines. Treat it as "go look," not necessarily as news.
- **Weekly granularity**: a page that changes and then changes back within the same week won't be noticed. The warning for an unreachable site takes 3 runs, so up to 3 weeks.
