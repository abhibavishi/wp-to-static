# WordPress Export

**Turn your WordPress site into a blazing-fast static site. Free hosting, zero maintenance, one command.**

---

## The Problem With WordPress

WordPress is a great way to build a site — but a painful way to run one.

Every month there are plugin updates to apply, security patches to install, and a hosting bill to pay. Your site takes 3-5 seconds to load because every visitor triggers a database query and PHP execution. And it's a constant target for bots, brute-force attacks, and malware injections.

Here's the thing: if your site is mostly content — blog posts, service pages, a portfolio — it doesn't need any of that. It doesn't need a database. It doesn't need PHP. It's just HTML, images, and CSS.

This skill does one thing: it takes your WordPress site and converts it into exactly that.

---

## What Changes

| | Before | After |
|---|---|---|
| **Hosting cost** | $20–50/month | $0/month |
| **File size** | 1.5 GB+ | ~25 MB |
| **Load time** | 3–5 seconds | Under 1 second |
| **Security** | Constant vulnerabilities | Zero attack surface |
| **Maintenance** | Monthly updates | None |
| **Looks identical?** | — | Yes, pixel-perfect |

---

## How It Works

1. **Snapshot** — It connects to your WordPress server and takes a complete, rendered snapshot of every page on your site.

2. **Strip** — It removes the WordPress engine entirely, keeping only what your visitors actually see: the HTML, images, fonts, and styles.

3. **Deploy** — It publishes the result to Cloudflare's global network, where your site loads instantly from a server near every visitor — for free.

Your original WordPress site is untouched. You can keep it running or shut it down — up to you.

---

## Quick Start

### 1. Install the skill

```bash
mkdir -p .claude/skills/wp-to-static
curl -o .claude/skills/wp-to-static/SKILL.md \
  https://raw.githubusercontent.com/abhibavishi/wp-to-static/main/skills/wp-to-static/SKILL.md
```

### 2. Set your site details

```bash
export WP_SSH_HOST="ssh.yourhost.com"
export WP_SSH_USER="your-username"
export WP_SSH_PORT="22"
export WP_SSH_KEY="$HOME/.ssh/your_wp_key"
export WP_SITE_URL="https://yoursite.com"
export WP_SITE_NAME="yoursite"
export CF_ACCOUNT_ID="your-cloudflare-account-id"
```

### 3. Run it

```bash
claude
# Then type:
/wp-to-static
```

Claude will walk you through the rest and ask for confirmation before deploying anything.

---

## What You'll Need

- [Claude Code](https://claude.ai/claude-code) installed and running
- SSH access to your WordPress server
- A free [Cloudflare account](https://cloudflare.com)
- [GitHub CLI](https://cli.github.com/) installed and logged in (`gh auth login`)
- [Wrangler](https://developers.cloudflare.com/workers/wrangler/) installed and logged in (`wrangler login`)

> **Not sure about SSH access?** Check with whoever set up your WordPress hosting. Most managed hosts (SiteGround, Kinsta, WP Engine, Hostinger) provide SSH access in their dashboard.

---

## Security

Your credentials are never stored, logged, or committed to git. SSH keys are loaded via `ssh-agent` and all credentials are passed through environment variables. The GitHub repo created for your static site is private by default.

<details>
<summary>How it works under the hood</summary>

WordPress sites are dynamically rendered by PHP — but the output is just HTML. This skill:

1. Runs `wget --mirror` on your server to capture the fully rendered HTML with all links converted to relative paths
2. Pulls original upload files via `rsync` for full-quality images
3. Parses every HTML and CSS file to build a complete list of every referenced asset
4. Copies only those files into a clean output directory (this is what shrinks 1.5 GB to ~25 MB)
5. Rewrites all absolute URLs to relative paths
6. Downloads Google Fonts locally so your site has no external dependencies
7. Removes WordPress metadata, RSS links, admin endpoints, and other WordPress-specific cruft
8. Adds Cloudflare caching headers and redirects for `/wp-admin`, `/wp-login.php`, and `/xmlrpc.php`
9. Lets you preview locally before anything is deployed

</details>

---

## License

MIT
