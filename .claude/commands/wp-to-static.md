# WordPress to Static Site (Cloudflare Pages)

Convert a WordPress website to a static site and deploy it to Cloudflare Pages.

## Required Environment Variables

Before running, the user MUST have these set:

- `WP_SSH_HOST` - SSH hostname (e.g., `ssh.example.com`)
- `WP_SSH_USER` - SSH username
- `WP_SSH_PORT` - SSH port (e.g., `18765`)
- `WP_SSH_KEY` - Path to SSH private key file (e.g., `~/.ssh/wp_key`)
- `WP_SSH_PASSPHRASE` - SSH key passphrase (if applicable, otherwise leave empty)
- `WP_SITE_URL` - The WordPress site URL (e.g., `https://example.com`)
- `WP_SITE_NAME` - Short name for the project (e.g., `mysite`) - used for GitHub repo and Cloudflare project
- `CF_ACCOUNT_ID` - Cloudflare account ID for Pages deployment

Optional:
- `WP_PAGES_TO_MIRROR` - Comma-separated list of specific pages to mirror (default: homepage only)
- `GH_REPO_VISIBILITY` - `private` or `public` (default: `private`)

## Instructions

Follow these steps exactly. Do NOT skip any step.

### Step 0: Validate Environment

Check that ALL required environment variables are set. If any are missing, stop and tell the user which ones they need to set. Do NOT proceed without them.

Verify tools are available: `ssh`, `rsync`, `curl`, `git`, `gh`, `wrangler`. If any are missing, tell the user to install them.

IMPORTANT: Never log, echo, or display the values of credential environment variables (`WP_SSH_PASSPHRASE`, SSH key contents, etc.).

### Step 1: Test SSH Connection

Use `expect` to handle the passphrase if `WP_SSH_PASSPHRASE` is set:

```
expect -c "
spawn ssh -i $WP_SSH_KEY -p $WP_SSH_PORT -o StrictHostKeyChecking=no $WP_SSH_USER@$WP_SSH_HOST \"echo connected\"
expect {
    \"passphrase\" { send \"$WP_SSH_PASSPHRASE\r\"; exp_continue }
    \"password\" { send \"$WP_SSH_PASSPHRASE\r\"; exp_continue }
    eof
}
"
```

If connection fails, stop and report the error. Do NOT retry with different credentials.

### Step 2: Locate WordPress Installation

SSH in and find the WordPress `public_html` or `htdocs` directory. Common locations:
- `~/www/DOMAIN/public_html/`
- `~/public_html/`
- `~/htdocs/`
- `/var/www/html/`

Look for `wp-config.php` as confirmation. Store the path as `WP_ROOT` for subsequent steps.

### Step 3: Mirror the Site with wget (on server)

Run `wget --mirror` ON THE SERVER (not locally) since the server likely has wget:

```
ssh into server and run:
cd /tmp && rm -rf static_mirror && mkdir -p static_mirror && cd static_mirror && \
wget --mirror --convert-links --adjust-extension --page-requisites --no-parent \
  --restrict-file-names=windows -e robots=off --timeout=30 --tries=3 --wait=0.5 \
  --user-agent="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)" \
  $WP_SITE_URL/ 2>&1 | tail -30
```

If `wget` is not on the server, fall back to running `curl` locally to download the rendered HTML, then pull assets via rsync.

### Step 4: Rsync Everything Locally

Create a local working directory (NOT inside the final output dir):

```
mkdir -p ./build/site
```

Rsync the wget mirror:
```
rsync -avz from server:/tmp/static_mirror/DOMAIN/ ./build/site/
```

Then rsync original assets for full quality (these are large, be patient):
```
rsync -avz from server:WP_ROOT/wp-content/uploads/ ./build/site/wp-content/uploads/
rsync -avz from server:WP_ROOT/wp-content/themes/ ./build/site/wp-content/themes/
rsync -avz from server:WP_ROOT/wp-content/plugins/ ./build/site/wp-content/plugins/
rsync -avz from server:WP_ROOT/wp-includes/ ./build/site/wp-includes/
```

### Step 5: Extract Referenced Assets

This is CRITICAL for keeping the output small. Parse `index.html` (and any other mirrored HTML pages) plus all CSS files to extract EVERY referenced local file path:

From HTML:
- `src=`, `href=`, `data-src=`, `data-srcset=`, `srcset=` attributes
- `background-image: url()` in inline styles

From CSS files:
- All `url()` references
- IMPORTANT: Resolve relative paths from the CSS file's location to the site root

Write the complete list to `./build/referenced-files.txt`.

Then create the clean output:
```
mkdir -p ./public
cp index.html ./public/
# Copy each referenced file, preserving directory structure
```

### Step 6: Fix Absolute URLs

In `index.html` and ALL CSS files:

1. Replace `$WP_SITE_URL/` with empty string (makes URLs relative)
2. Replace any staging/dev domain URLs with local paths
3. Replace Google Fonts URLs with local copies:
   - Download each font file to `./public/fonts/`
   - Update `@font-face` `src:` URLs to `fonts/filename.ttf`
4. Remove `<link rel="preconnect">` hints for Google Fonts domains

IMPORTANT for CSS files: When replacing absolute URLs, account for the CSS file's location:
- If CSS is at `wp-content/uploads/cache/file.css`:
  - References to `wp-content/uploads/` become `../../`
  - References to `wp-content/themes/` become `../../themes/`
  - References to `wp-includes/` become `../../../wp-includes/`

### Step 7: Clean WordPress Cruft

Remove from HTML:
- `<meta name="generator" content="WordPress...">` tags
- `<meta name="generator" content="Powered by WPBakery...">`
- `<meta name="generator" content="Powered by Slider Revolution...">`
- `<link rel="EditURI"...>` (xmlrpc)
- `<link rel="alternate"...>` for RSS feeds, oEmbed JSON/XML
- `<link rel="https://api.w.org/"...>`
- `<link rel="shortlink"...>`
- `<link rel="profile" href="https://gmpg.org/xfn/11">`
- `<link rel="dns-prefetch"...>` for fonts.googleapis.com
- W3 Total Cache HTML comments (usually last line)
- Any `wp-json` root references in inline JSON

Do NOT remove:
- Email addresses containing the domain
- The `<link rel="canonical">` tag (update it to just `/` or `index.html`)

### Step 8: Add Cloudflare Pages Configuration

Create `./public/_headers`:
```
/fonts/*
  Cache-Control: public, max-age=31536000, immutable
  Access-Control-Allow-Origin: *

/wp-content/uploads/*
  Cache-Control: public, max-age=31536000, immutable

/wp-content/themes/*
  Cache-Control: public, max-age=31536000, immutable

/wp-content/plugins/*
  Cache-Control: public, max-age=31536000, immutable

/wp-includes/*
  Cache-Control: public, max-age=31536000, immutable
```

Create `./public/_redirects`:
```
/wp-admin/*  /  302
/wp-login.php  /  302
/xmlrpc.php  /  302
/feed/*  /  302
```

### Step 9: Verify

1. Start a local HTTP server: `python3 -m http.server 8765` from `./public/`
2. Test that key assets return 200: CSS, JS, jQuery, logo, fonts, sample images
3. Tell the user to open `http://localhost:PORT` to visually verify
4. Wait for user confirmation before proceeding to deploy

### Step 10: Deploy

1. Initialize git repo, commit all files in `./public/` and `.gitignore`
2. Set `git config http.postBuffer 524288000` (needed for binary assets)
3. Create GitHub repo: `gh repo create $WP_SITE_NAME --$GH_REPO_VISIBILITY --source=. --push`
4. If push fails, retry with the postBuffer config
5. Create Cloudflare Pages project: `CLOUDFLARE_ACCOUNT_ID=$CF_ACCOUNT_ID wrangler pages project create $WP_SITE_NAME --production-branch main`
6. Deploy: `CLOUDFLARE_ACCOUNT_ID=$CF_ACCOUNT_ID wrangler pages deploy ./public --project-name $WP_SITE_NAME`
7. Verify deployment returns 200
8. Tell user the live URL and remind them to add custom domain in Cloudflare dashboard

### IMPORTANT Safety Rules

- NEVER display or log SSH keys, passphrases, or other credentials
- NEVER commit credentials to git
- NEVER delete the current working directory (it breaks the shell) - use a separate `./build/` directory for temp work
- NEVER force-push or use destructive git commands
- Always use `./build/` for intermediate files and `./public/` for the final output
- Clean up `./build/` only AFTER everything is deployed
- If any step fails, stop and report the error - do NOT retry blindly
