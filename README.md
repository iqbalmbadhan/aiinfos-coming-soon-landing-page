# AI Infos — Pre-launch waitlist page

![PHP](https://img.shields.io/badge/PHP-8.1%2B-777BB4?logo=php&logoColor=white)
![Dependencies](https://img.shields.io/badge/dependencies-0-brightgreen)
![Build step](https://img.shields.io/badge/build%20step-none-brightgreen)
![Data](https://img.shields.io/badge/data-CSV%2C%20no%20database-informational)

A single-page "launching soon" site with an animated hero, a live countdown, and an early-access
waitlist that writes to a CSV file. Two files, no framework, no database, no build step.

[Live site](https://aiinfos.com) · [Repository](https://github.com/iqbalmbadhan/aiinfos-coming-soon-landing-page) · [Issues](https://github.com/iqbalmbadhan/aiinfos-coming-soon-landing-page/issues)

Built for [AI Infos](https://aiinfos.com), a software firm in the making: ServiceNow solutions,
AI automation, web and mobile apps, and custom software.

## Highlights

- **Animated hero** — diagonal clip-path wipes with drifting SVG patterns, staggered type reveals.
- **Countdown** — days / hours / minutes / seconds, driven by one ISO timestamp, and it swaps itself
  for a "We are live." panel when the target passes.
- **Timezone-aware** — the date line and the hero clock both render in a configurable IANA zone
  (`America/Chicago` by default), so DST is handled by the browser, not by you.
- **No-JS-dependency front end** — plain HTML, CSS and vanilla JS in one file.
- **Waitlist with real guards** — honeypot field, duplicate detection, per-IP hourly rate limit, and
  CSV formula-injection protection.
- **Accessible** — screen-reader heading, visible focus rings, ARIA labels on the timer and icon
  links, and full `prefers-reduced-motion` support.
- **Private by default** — the CSV lives outside the public web root and is never reachable by URL.

## Files

```
.
├── public/
│   ├── index.html     # the page: hero, countdown, form (all CSS/JS inline)
│   └── signup.php     # POST endpoint, appends one row per signup to the CSV
├── data/              # created at runtime, one level ABOVE public/ (not in git)
└── README.md
```

`data/` intentionally does not exist in the repository. When deployed, the web root is `public/`
and the CSV is written to `../data/waitlist.csv`.

## Quick start (local)

Requires PHP 8.1 or newer.

```bash
git clone https://github.com/iqbalmbadhan/aiinfos-coming-soon-landing-page.git
cd aiinfos-coming-soon-landing-page
php -S localhost:8000 -t public
```

Open <http://localhost:8000>. Submitting the form writes `data/waitlist.csv` in the project root.

> Opening `public/index.html` directly with `file://` works for the countdown, but the form needs a
> server because it posts to `signup.php`.

## Deploy

Works on any nginx/Apache host with PHP-FPM. Steps below assume a CloudPanel VPS.

1. **Create the site as a PHP site.** In CloudPanel: *Sites → Add Site → Create a PHP Site*.
   A "Static" site will not execute PHP and every signup fails with `405`.
2. **Upload `public/index.html` and `public/signup.php`** into the web root, usually
   `/home/USER/htdocs/example.com/`.
3. **Create the data directory next to the web root** (not inside it), owned by the site user:
   ```bash
   mkdir -p /home/USER/htdocs/data
   chmod 750 /home/USER/htdocs/data
   chown USER:USER /home/USER/htdocs/data
   ```
   `signup.php` resolves it with `dirname(__DIR__) . '/data'`, so `/home/USER/htdocs/example.com`
   maps to `/home/USER/htdocs/data/waitlist.csv`. Set the `WAITLIST_DIR` environment variable to
   use a different location.
4. **Verify** before sharing the link:
   ```bash
   # PHP must execute the file: expect JSON, NOT the literal source code
   curl -s https://example.com/signup.php
   # {"ok":false,"error":"Method not allowed."}

   # A real signup: expect {"ok":true}
   curl -i -X POST https://example.com/signup.php \
     -H "Content-Type: application/json" \
     -d '{"email":"you@example.com","website":""}'
   ```
5. **Check the CSV is private.** Both of these must return `404`:
   ```bash
   curl -sI https://example.com/data/waitlist.csv
   curl -sI https://example.com/../data/waitlist.csv
   ```

## Configuration

### Launch time

In `public/index.html`, near the bottom of the script:

```js
var LAUNCH = '2026-12-15T10:00:00-06:00';
```

ISO 8601 with an explicit offset. `-06:00` is Chicago in winter (CST); use `-05:00` for CDT.
To display the date in another zone, change `America/Chicago` in the three `timeZone` options and
the "Chicago" label in the hero.

### Other settings

| Setting | File | Default | Purpose |
|---|---|---|---|
| `TRUSTED_IP_HEADER` | `signup.php` | `''` | Set to `'CF-Connecting-IP'` behind Cloudflare, otherwise every row records the proxy IP |
| `MAX_PER_IP_PER_HOUR` | `signup.php` | `10` | Per-IP signup cap; blocks simple floods |
| `WAITLIST_DIR` | env var | `../data` | Absolute path for the CSV directory |
| `DATA_DIR` / `CSV_FILE` | `signup.php` | derived | Override if you prefer a hard-coded path |

## API

`POST /signup.php` with `Content-Type: application/json`:

```json
{ "email": "jane@example.com", "website": "" }
```

| Status | Body | Meaning |
|---|---|---|
| `200` | `{"ok":true}` | Saved, already on the list, or honeypot tripped (bots get a fake success) |
| `422` | `{"ok":false,"error":"..."}` | Missing or invalid email |
| `429` | `{"ok":false,"error":"..."}` | Too many signups from this IP in the last hour |
| `405` | `{"ok":false,"error":"Method not allowed."}` | Not a POST (PHP is running) |
| `500` | `{"ok":false,"error":"..."}` | CSV directory or file not writable |

Every response is JSON, `Cache-Control: no-store`. Errors are logged to the PHP error log.

## What gets saved

`data/waitlist.csv`, created automatically on the first signup:

| email | signed_up_at | ip | consent |
|---|---|---|---|
| jane@example.com | 2026-12-01T09:14:02Z | 203.0.113.7 | notice-v1 |

- `signed_up_at` is UTC.
- `consent` records which version of the on-page privacy notice the visitor saw.
- Duplicate emails are ignored but still reported as success.
- Emails starting with `=`, `+`, `-` or `@` are prefixed with `'` so spreadsheets never run them as
  formulas.
- File mode is tightened to `0640` after each write.
- The CSV is the only copy: back it up.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `405 (Method Not Allowed)`, `Content-Type: text/html` | nginx rejected a POST to a static file | The site must be a PHP site; recreate it as one in CloudPanel |
| `GET /signup.php` returns `<?php declare(...)` | PHP is not executed for this vhost; the source is downloadable | Same fix as above, and check `php-fpm` is running |
| `500` / "Something went wrong" | `data/` missing or not writable by PHP-FPM | Create it beside the web root with owner `USER`, mode `750` |
| Every row has the same IP | Site sits behind Cloudflare or another proxy | Set `TRUSTED_IP_HEADER` accordingly |
| Countdown shows `00 00 00 00` | `LAUNCH` is unparsable or already in the past | Use full ISO 8601 with an offset |
| Wrong date shown | Browser locale/timezone override | The page always renders in the zone set in the `timeZone` options |

## Privacy and security notes

IP addresses are personal data under GDPR/UK GDPR and similar laws, even when collected only for
spam control. The page discloses that email and IP are stored, and how to request deletion.

Before launch:

- Add a short privacy page and link it near the form: what is kept, why, for how long, how to ask
  for deletion.
- Pick a retention period for IPs (for example 30-90 days after launch) and stick to it.
- Email only people who opted in, with an unsubscribe link in every message.
- Never move the CSV into the web root or send it around unencrypted.
- Prefer SFTP or the File Manager over public URLs when exporting the list.

## License

No license file is included yet. Add a `LICENSE` of your choice (MIT and Apache-2.0 are common for
site templates) before accepting contributions.
