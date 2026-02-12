---
name: wp-to-static
description: Convert a WordPress website to a static site and deploy to Cloudflare Pages. Mirrors the rendered HTML via SSH, extracts only referenced assets (shrinks 1.5GB+ to ~25MB), fixes URLs, self-hosts fonts, strips WordPress cruft, and deploys. Use when migrating a WordPress site to static hosting.
disable-model-invocation: true
argument-hint: "[site-url]"
allowed-tools: Bash, Read, Write, Edit, Grep, Glob, Task, WebFetch
metadata: {"openclaw":{"requires":{"bins":["ssh","rsync","curl","git","gh","wrangler","expect"],"env":["WP_SSH_HOST","WP_SSH_USER","WP_SSH_PORT","WP_SSH_KEY","WP_SITE_URL","WP_SITE_NAME"]},"emoji":"🔄","os":["darwin","linux"]}}
---

# WordPress to Static Site (Cloudflare Pages)

Convert a WordPress website to a pixel-perfect static site and deploy it to Cloudflare Pages. Zero attack surface, zero hosting cost, instant load times.

## Environment Variables

**Required** (stop and ask if missing):
- `WP_SSH_HOST` — SSH hostname (e.g., `ssh.example.com`)
- `WP_SSH_USER` — SSH username
- `WP_SSH_PORT` — SSH port (e.g., `18765`)
- `WP_SSH_KEY` — Path to SSH private key file (e.g., `~/.ssh/wp_key`)
- `WP_SITE_URL` — WordPress site URL (e.g., `https://example.com`)
- `WP_SITE_NAME` — Short project name (e.g., `mysite`)

**Optional:**
- `WP_SSH_PASSPHRASE` — SSH key passphrase (leave empty if none)
- `CF_ACCOUNT_ID` — Cloudflare account ID for Pages deployment
- `GH_REPO_VISIBILITY` — `private` (default) or `public`

**SECURITY: NEVER log, echo, or display credential values.**

## Step 0: Validate

Check all required env vars are set. Verify required binaries exist (`ssh`, `rsync`, `curl`, `git`, `gh`, `wrangler`, `expect`). Stop if anything is missing.

## Step 1: Test SSH Connection

```bash
expect -c "
spawn ssh -i \$WP_SSH_KEY -p \$WP_SSH_PORT -o StrictHostKeyChecking=no \$WP_SSH_USER@\$WP_SSH_HOST \"echo connected\"
expect {
    \"passphrase\" { send \"\$WP_SSH_PASSPHRASE\r\"; exp_continue }
    \"password\" { send \"\$WP_SSH_PASSPHRASE\r\"; exp_continue }
    eof
}
"
```

If connection fails, stop and report the error.

## Step 2: Locate WordPress Installation

SSH in and find the `public_html` directory. Common locations:
- `~/www/DOMAIN/public_html/`
- `~/public_html/`
- `~/htdocs/`
- `/var/www/html/`

Confirm by finding `wp-config.php`. Store path as `WP_ROOT`.

## Step 3: Mirror with wget (ON THE SERVER)

Run `wget --mirror` **on the server** (not locally):

```bash
cd /tmp && rm -rf static_mirror && mkdir -p static_mirror && cd static_mirror && \
wget --mirror --convert-links --adjust-extension --page-requisites --no-parent \
  --restrict-file-names=windows -e robots=off --timeout=30 --tries=3 --wait=0.5 \
  --user-agent="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)" \
  $WP_SITE_URL/ 2>&1 | tail -30
```

If `wget` is not on the server, use `curl` locally for the rendered HTML instead.

## Step 4: Rsync to Local

Create `./build/site` (NEVER use the project root as temp dir):

```bash
rsync -avz server:/tmp/static_mirror/DOMAIN/ ./build/site/
rsync -avz server:$WP_ROOT/wp-content/uploads/ ./build/site/wp-content/uploads/
rsync -avz server:$WP_ROOT/wp-content/themes/ ./build/site/wp-content/themes/
rsync -avz server:$WP_ROOT/wp-content/plugins/ ./build/site/wp-content/plugins/
rsync -avz server:$WP_ROOT/wp-includes/ ./build/site/wp-includes/
```

## Step 5: Extract Only Referenced Assets

**This is the key step.** Parse all HTML and CSS files to find every referenced local file:

**From HTML:** `src=`, `href=`, `data-src=`, `data-srcset=`, `srcset=`, inline `background-image: url()`

**From CSS:** All `url()` references — resolve relative paths from CSS file location to site root.

Write the list to `./build/referenced-files.txt`, then copy only those files to `./public/` preserving directory structure. This typically shrinks 1.5GB+ down to ~25MB.

## Step 6: Fix Absolute URLs

In `index.html` and ALL CSS files:

1. Replace `$WP_SITE_URL/` → empty string (relative paths)
2. Replace any staging/dev domain URLs → local paths
3. Self-host Google Fonts:
   - Download each `.ttf` to `./public/fonts/`
   - Update `@font-face src:` to `fonts/filename.ttf`
4. Remove `<link rel="preconnect">` for Google Fonts domains

**CSS path resolution is critical.** If CSS is at `wp-content/uploads/cache/file.css`:
- `wp-content/uploads/` → `../../`
- `wp-content/themes/` → `../../themes/`
- `wp-includes/` → `../../../wp-includes/`

## Step 7: Strip WordPress Cruft

**Remove:**
- `<meta name="generator" ...>` (WordPress, WPBakery, Slider Revolution)
- `<link rel="EditURI"...>`, `<link rel="alternate"...>` (RSS, oEmbed)
- `<link rel="https://api.w.org/"...>`, `<link rel="shortlink"...>`
- `<link rel="profile" href="gmpg.org/xfn/11">`
- `<link rel="dns-prefetch"...>` for fonts.googleapis.com
- W3 Total Cache HTML comments
- `wp-json` root references in inline JSON

**Keep:** Email addresses, `<link rel="canonical">` (update to `/`)

## Step 8: Cloudflare Pages Config

Create `./public/_headers` with aggressive caching for `/fonts/*`, `/wp-content/*`, `/wp-includes/*`.

Create `./public/_redirects` redirecting `/wp-admin/*`, `/wp-login.php`, `/xmlrpc.php`, `/feed/*` → `/` (302).

## Step 9: Verify Locally

1. Start `python3 -m http.server` from `./public/`
2. Test key assets return HTTP 200 (CSS, JS, logo, fonts, images)
3. Tell user to open the URL and visually verify
4. **Wait for user confirmation before deploying**

## Step 10: Deploy

1. `git init`, commit `./public/` and `.gitignore`
2. `git config http.postBuffer 524288000` (for binary assets)
3. `gh repo create $WP_SITE_NAME --private --source=. --push`
4. `CLOUDFLARE_ACCOUNT_ID=$CF_ACCOUNT_ID wrangler pages project create $WP_SITE_NAME --production-branch main`
5. `CLOUDFLARE_ACCOUNT_ID=$CF_ACCOUNT_ID wrangler pages deploy ./public --project-name $WP_SITE_NAME`
6. Verify deployment, report live URL, remind about custom domain setup

## Safety Rules

- NEVER display or log credentials (SSH keys, passphrases, tokens)
- NEVER commit credentials to git
- NEVER delete the current working directory (breaks the shell CWD)
- NEVER force-push or use destructive git commands
- Use `./build/` for temp files, `./public/` for output
- Clean up `./build/` only AFTER successful deploy
- Stop and report on any failure — do NOT retry blindly
