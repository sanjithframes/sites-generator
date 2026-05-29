# Engine — Full Documentation

## What is this?

**Engine** is a browser-based site generator tool. You fill in business details, pick a template, and it generates a complete website (HTML + CSS + images). You can either download it as a zip or deploy it live to a subdomain instantly.

Live at: `http://engine.smart-forms.in`

---

## System Overview

```
You (browser) ──── engine.smart-forms.in ──── your site generator
                                                      │
                                                      ▼ Deploy Live button
                                               deployguy API (:9000)
                                                      │
                                                      ▼
                                               deploy-domain script
                                                      │
                                                      ▼
                                          http://sitename.smart-forms.in
```

---

## Part 1 — Engine (Site Generator)

**What it is:** Single HTML file. No backend, no framework. Everything runs in the browser.

**Source:** `C:\Users\sanji\engine\index.html`

**Live URL:** `http://engine.smart-forms.in`

**Features:**
- 17+ industry templates (Restaurant, Dental, Real Estate, Hotel, etc.)
- Live preview as you type
- Brand color picker
- Image upload with auto-compression (JPEG 0.85 quality, no visible loss)
- SEO baked in — meta description, Open Graph, Schema.org JSON-LD per template
- Generate & download as zip
- Deploy Live directly to your VPS

**How to use locally:**
- Open `index.html` directly in Chrome (double-click)
- The Deploy Live button works from local file (no mixed content issue)
- Vercel/hosted version: Deploy button won't work (HTTPS → HTTP blocked by browser)

---

## Part 2 — deployguy (Go Backend)

**What it is:** Lightweight Go HTTP API running on the VPS. Receives a zip file from the site generator and deploys it as a live subdomain.

**Source:** `C:\Users\sanji\deployguy\main.go`

**Running on VPS:** `root@72.61.253.142` on port `9000`

**Binary location on server:** `/opt/deployguy/deployguy`

**Config on server:** `/opt/deployguy/.env`

```env
API_KEY=<your-api-key>        # keep this secret — matches X-API-Key header
DOMAIN=smart-forms.in
PORT=9000
```

### API Endpoints

**POST /deploy** — Deploy a site
```
Header:  X-API-Key: <api-key>
Body:    multipart/form-data
         site_name = "my-business"   (lowercase, letters/numbers/hyphens, min 3 chars)
         file      = upload.zip
```

Responses:
| Code | Meaning |
|------|---------|
| 200 | `{"status":"ok","url":"http://sitename.smart-forms.in"}` |
| 401 | Wrong API key |
| 409 | Name already taken — choose a different name |
| 500 | Deploy script failed |

**GET /health** — Check if running
```
curl http://72.61.253.142:9000/health
→ {"status":"ok"}
```

**Rule:** A site name that was ever deployed can never be reused. Server checks if `/var/www/sites/<name>` exists before deploying.

### Rebuild & Redeploy deployguy

When you change `main.go`:
```powershell
# On Windows — cross-compile for Linux
cd C:\Users\sanji\deployguy
$env:GOOS="linux"; $env:GOARCH="amd64"; go build -o deployguy

# Stop, upload, restart on VPS
! ssh root@72.61.253.142 "systemctl stop deployguy"
! scp C:/Users/sanji/deployguy/deployguy root@72.61.253.142:/opt/deployguy/deployguy
! ssh root@72.61.253.142 "systemctl start deployguy"
```

### Service Management (on VPS)
```bash
systemctl status deployguy       # check running
systemctl restart deployguy      # restart
journalctl -u deployguy -f       # live logs
```

---

## Part 3 — CI/CD Pipeline

**What it does:** Every time you push to the `main` branch on GitHub, the site generator (`index.html`) is automatically deployed to `http://engine.smart-forms.in` — no manual steps needed.

**Flow:**
```
git push → GitHub → GitHub Actions triggers → SCP index.html → VPS → site updated
```

**Time:** Under 30 seconds from push to live.

### Workflow File

Located at: `.github/workflows/deploy.yml`

```yaml
name: Deploy to engine.smart-forms.in

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Upload index.html to VPS
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.VPS_HOST }}
          username: root
          password: ${{ secrets.VPS_PASSWORD }}
          source: "index.html"
          target: "/var/www/sites/engine/"
```

### GitHub Secrets (set once in repo Settings → Secrets → Actions)

| Secret Name | Value |
|-------------|-------|
| `VPS_HOST` | `72.61.253.142` |
| `VPS_PASSWORD` | stored in your password manager |

### Manual Deploy (bat file)

`deploy-generator.bat` in this folder — double-click to manually push without going through git. Uses PuTTY's `pscp` to copy directly to VPS.

---

## Part 4 — Server Architecture

**VPS:** `root@72.61.253.142`

**Domain:** `smart-forms.in` (wildcard DNS `*.smart-forms.in` points to this server)

**Nginx configs location:** `/etc/nginx/sites-available/`

| Config file | Handles | Serves from |
|-------------|---------|-------------|
| `sanju-api` | `*.smart-forms.in` (wildcard) | `/var/www/sites/<subdomain>/` |
| `erp` | `erp.smart-forms.in` | proxies to port 8001 |

**How client sites work:**
```
browser → engine.smart-forms.in → deploy-domain mysite
       → creates /var/www/sites/mysite/
       → nginx wildcard serves it at http://mysite.smart-forms.in
```

**deploy-domain script** (on VPS):
- Takes a site name as argument
- Reads zip from `/var/deploy/workspace/<name>/upload.zip`
- Extracts to `/var/www/sites/<name>/`
- Backs up to `/var/backups/sites/<name>/`
- Site is immediately live via nginx wildcard

**SSL note:** Self-signed cert exists at `/etc/letsencrypt/live/smart-forms.in/` to satisfy nginx config. Sites are HTTP only for now. Let's Encrypt real cert can be added later.

---

## Part 5 — Go Language (why Go for deployguy)

Go was chosen over Python/Node for deployguy because:
- Compiles to a **single binary** — no runtime, no dependencies to install on server
- Cross-compile from Windows to Linux in one command
- Very low memory (1.1MB RAM for deployguy vs ~30MB for a Python service)
- Fast startup — systemd restarts it in milliseconds
- Standard library handles HTTP, file I/O, exec — no external packages needed

The entire deployguy service is **one file** (`main.go`, ~130 lines).

---

## Part 6 — API Key

The API key is the only authentication between the site generator (browser) and deployguy (server). It is:
- Set in `/opt/deployguy/.env` on the server
- Stored in browser `localStorage` when you enter it in the ⚙ API Settings modal
- Sent as `X-API-Key` header on every deploy request
- Never stored in the HTML file or any public file

To change it: edit `.env` on the server and restart deployguy.

---

## Quick Reference

| Task | Command / Action |
|------|-----------------|
| Push and auto-deploy engine | `git push` |
| Manual deploy engine | Double-click `deploy-generator.bat` |
| Deploy a client site | Click Deploy Live ⚡ in engine |
| Check deployguy health | `curl http://72.61.253.142:9000/health` |
| Check deployguy logs | `ssh root@72.61.253.142 "journalctl -u deployguy -f"` |
| Rebuild deployguy | See Part 2 above |
| View live GitHub Actions | github.com/sanjithframes/sites-generator/actions |

---

## Folder Structure

```
C:\Users\sanji\
├── engine\                    ← site generator (this repo)
│   ├── index.html             ← the entire app
│   ├── deploy-generator.bat   ← manual deploy script
│   ├── git-steps.txt          ← git workflow reference
│   ├── DOCS.md                ← this file
│   └── .github\
│       └── workflows\
│           └── deploy.yml     ← CI/CD pipeline
│
└── deployguy\                 ← Go deploy backend
    ├── main.go
    ├── go.mod
    ├── .env.example
    ├── .gitignore
    └── deployguy.service      ← systemd unit file

B:\deployguy\
└── README.md                  ← deployguy-specific docs
```
