# /wp-to-static

A Claude Code skill that converts any WordPress website to a static site and deploys it to Cloudflare Pages in minutes.

**What it does:** SSH into your WordPress server, mirror the rendered site, extract only the referenced assets (typically shrinks 1.5GB+ to ~25MB), fix all URLs for static hosting, self-host Google Fonts, strip WordPress cruft, and deploy to Cloudflare Pages.

## Quick Start

### 1. Install the skill

Copy the `.claude/commands/wp-to-static.md` file into your project's `.claude/commands/` directory:

```bash
mkdir -p .claude/commands
curl -o .claude/commands/wp-to-static.md \
  https://raw.githubusercontent.com/abhibavishi/wp-to-static/main/.claude/commands/wp-to-static.md
```

Or clone this repo and copy the file manually.

### 2. Set up your SSH key

Save your WordPress server's SSH private key to a file:

```bash
# Create the key file (paste your private key contents)
nano ~/.ssh/wp_mysite_key
chmod 600 ~/.ssh/wp_mysite_key
```

### 3. Set environment variables

```bash
export WP_SSH_HOST="ssh.example.com"
export WP_SSH_USER="your-ssh-username"
export WP_SSH_PORT="18765"
export WP_SSH_KEY="$HOME/.ssh/wp_mysite_key"
export WP_SSH_PASSPHRASE="your-key-passphrase"  # leave empty if none
export WP_SITE_URL="https://example.com"
export WP_SITE_NAME="mysite"
export CF_ACCOUNT_ID="your-cloudflare-account-id"
```

### 4. Run it

```bash
claude
# Then type:
/wp-to-static
```

Claude will:
1. Connect to your WordPress server via SSH
2. Mirror the rendered website
3. Download all assets (images, CSS, JS, fonts)
4. Extract only the files referenced by your pages (~98% size reduction)
5. Fix all URLs for static hosting
6. Self-host Google Fonts
7. Remove WordPress bloat (admin, xmlrpc, generators, etc.)
8. Let you preview locally before deploying
9. Create a private GitHub repo and deploy to Cloudflare Pages

## Requirements

- [Claude Code](https://claude.ai/claude-code) CLI
- `ssh`, `rsync`, `git`, `curl` (standard on macOS/Linux)
- `expect` (standard on macOS, `sudo apt install expect` on Linux)
- [GitHub CLI](https://cli.github.com/) (`gh`) - authenticated
- [Wrangler](https://developers.cloudflare.com/workers/wrangler/) (`wrangler`) - authenticated

## Security

- Credentials are passed via environment variables, never hardcoded
- SSH keys are read from local files, never committed to git
- The skill explicitly refuses to log or display credential values
- All GitHub repos are created as private by default
- The `.gitignore` excludes sensitive files

## How It Works

WordPress sites are dynamically rendered by PHP, but the output is just HTML + static assets. This skill:

1. Uses `wget --mirror` on the server to capture the fully rendered HTML with all links converted to relative paths
2. Pulls original upload files via `rsync` for full-quality images
3. Parses the HTML and CSS to build a complete list of every referenced file (images, fonts, scripts, stylesheets)
4. Copies only those files into a clean output directory
5. Rewrites all absolute URLs to relative paths
6. Downloads Google Fonts locally for self-hosting
7. Adds Cloudflare-specific caching headers and redirect rules

The result is a pixel-perfect static copy that loads faster, costs nothing to host, and has zero attack surface.

## License

MIT
