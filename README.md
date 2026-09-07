# resume.lab980.com

Jim Orr's resume site. Single static page, self-hosted fonts, no build step,
no third-party requests. The PDF is generated from the page itself, so the
two can't drift.

## Files

- `index.html` — the whole site (styles inline), including `@media print`
  rules tuned so the printed page fits on one sheet.
- `fonts/` — latin-subset woff2 for Inter, Space Grotesk, JetBrains Mono,
  plus `fonts.css` (`@font-face` declarations). No Google Fonts dependency.
- `Jim_Orr_Resume_2026.pdf` — generated from `index.html` (committed because
  the droplet serves it directly; regenerate after any content edit).

## Regenerating the PDF

Any Chromium works:

```bash
chromium --headless=new --no-sandbox --no-pdf-header-footer \
  --print-to-pdf=Jim_Orr_Resume_2026.pdf "file://$PWD/index.html"
```

Check it's still one page before committing.

## Deploy (lab980 droplet)

Serves as static files from `/var/www/resume.lab980.com` behind nginx
(standard lab980 shape — no app process, no pm2).

One-time cutover from the unversioned directory:

```bash
cd /var/www
mv resume.lab980.com resume.lab980.com.bak
git clone https://github.com/ivjames/resume resume.lab980.com
nginx -t && systemctl reload nginx   # only if the root path changed
```

After that, deploy = `git -C /var/www/resume.lab980.com pull`.
Delete the `.bak` dir once the live site checks out.

**Check the branch before trusting that pull.** `git pull` reports "Already up
to date" truthfully about whatever branch the checkout happens to be on, so a
web root left on some other branch serves a stale build while every deploy
looks clean. This is not hypothetical on this droplet — `ivjames/forest` sat on
a `claude/*` branch for three weeks doing exactly that. Confirm, then pin:

```bash
cd /var/www/resume.lab980.com
git status -sb                      # expect: ## main...origin/main
git checkout -B main origin/main    # if it isn't
git branch -u origin/main main
```

**Verify what is actually live.** A 200 only proves nginx answered, not which
build it served. Compare the served bytes against the commit you expect:

```bash
git fetch -q origin main
curl -s https://resume.lab980.com/ | git hash-object --stdin
git rev-parse origin/main:index.html
```

Identical hashes mean the deploy landed. Fetch first and compare against
`origin/main`, not local `main` — a stale clone and a stale deploy hash
identically, so the local-branch form passes in exactly the case it exists to
catch. Verified 2026-09-07: both sides read `0e112568…`, so the live site
matches `origin/main`.
