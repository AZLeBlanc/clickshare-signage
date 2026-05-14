### GitHub Actions Workflow (Detailed)

**File:** `.github/workflows/update-manifests.yml`

```yaml
name: Update Image Manifests

# Trigger: Only on pushes to main that modify image files
on:
  push:
    branches:
      - main
    paths:
      - 'units/**/images/**'

# Required permission to commit back to repository
permissions:
  contents: write

jobs:
  update-manifests:
    runs-on: ubuntu-latest
    
    steps:
      # Step 1: Clone repository
      - name: Checkout repo
        uses: actions/checkout@v4

      # Step 2: Run Python script to generate manifests
      - name: Generate manifests for all units
        run: |
          python3 - <<'EOF'
          import json
          import os
          from pathlib import Path

          UNITS_DIR = Path("units")
          EXTENSIONS = {".jpg", ".jpeg", ".png", ".gif", ".webp"}

          # Discover all unit folders
          unit_folders = [
              d for d in UNITS_DIR.iterdir() 
              if d.is_dir() and (d / "images").exists()
          ]

          print(f"Found {len(unit_folders)} unit(s)")

          # Process each unit
          for unit_dir in sorted(unit_folders):
              unit_name = unit_dir.name
              images_dir = unit_dir / "images"
              output_file = unit_dir / "images.json"

              # Load existing config to preserve timing
              if output_file.exists():
                  with open(output_file) as f:
                      existing = json.load(f)
              else:
                  existing = {}

              # Scan for image files
              found = sorted([
                  f"images/{f.name}"
                  for f in images_dir.iterdir()
                  if f.is_file() 
                  and f.suffix.lower() in EXTENSIONS
                  and not f.name.startswith(".")
              ])

              # Build manifest
              config = {
                  "_note": f"Auto-generated for {unit_name}. Don't edit manually.",
                  "interval": existing.get("interval", 15000),
                  "transition": existing.get("transition", 2000),
                  "images": found
              }

              # Write to disk
              with open(output_file, "w") as f:
                  json.dump(config, f, indent=2)

              print(f"\n{unit_name}:")
              print(f"  ✓ Found {len(found)} images")
              for img in found:
                  print(f"    • {img}")
          EOF

      # Step 3: Commit changes (if any)
      - name: Commit updated manifests
        run: |
          git config user.name  "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add units/*/images.json
          
          # Only commit if manifests actually changed
          if git diff --staged --quiet; then
            echo "No changes to any manifests"
          else
            git commit -m "chore: update image manifests [skip ci]"
            git push
          fi
```

**Key Design Decisions:**

1. **Path filtering** — Only triggers on image changes to avoid unnecessary runs
2. **Preserve settings** — Loads existing `images.json` to keep custom timing
3. **Idempotent** — Running multiple times produces same result
4. **Skip CI tag** — `[skip ci]` prevents infinite loop (manifest commit doesn't re-trigger action)
5. **Batch processing** — Handles all units in single workflow run

### Slideshow HTML (Detailed)

**File:** `units/[unit-name]/index.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Digital Signage - [Unit Name]</title>
  
  <style>
    /* CSS Reset */
    *, *::before, *::after {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
    }

    /* Full viewport coverage */
    html, body {
      width: 100%;
      height: 100%;
      background: black;
      overflow: hidden;  /* Hide scrollbars */
    }

    /* Slideshow container */
    #slideshow {
      position: relative;
      width: 100%;
      height: 100%;
    }

    /* Individual slide styling */
    .slide {
      position: absolute;
      inset: 0;                    /* Shorthand for top/right/bottom/left: 0 */
      width: 100%;
      height: 100%;
      object-fit: cover;           /* Fill screen, crop to aspect ratio */
      opacity: 0;                  /* Hidden by default */
      transition-property: opacity;
      transition-timing-function: ease-in-out;
      /* transition-duration set dynamically via JS */
    }

    /* Active slide is visible */
    .slide.active {
      opacity: 1;
    }

    /* Loading indicator */
    #loading {
      position: absolute;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      color: white;
      font-family: system-ui, sans-serif;
      font-size: 24px;
    }

    /* Hide loading when slideshow has content */
    #slideshow:not(:empty) + #loading {
      display: none;
    }
  </style>
</head>
<body>
  <div id="slideshow"></div>
  <div id="loading">Loading slideshow...</div>

  <script>
    /**
     * Initialize the slideshow
     * 1. Fetch images.json from same directory
     * 2. Create <img> elements for each image
     * 3. Start rotation timer
     */
    async function init() {
      try {
        // Fetch config with cache-busting timestamp
        // This ensures updates are picked up immediately
        const config = await fetch(`images.json?t=${Date.now()}`)
          .then(r => r.json());

        const { images, interval, transition } = config;
        const container = document.getElementById('slideshow');

        // Validate config
        if (!images || images.length === 0) {
          document.getElementById('loading').textContent = 'No images configured';
          return;
        }

        // Create DOM elements for each image
        const slides = images.map((src) => {
          const img = document.createElement('img');
          img.src = src;
          img.className = 'slide';
          img.alt = '';  // Decorative image, no alt needed
          img.style.transitionDuration = `${transition}ms`;
          container.appendChild(img);
          return img;
        });

        // Show first image immediately
        let current = 0;
        slides[0].classList.add('active');

        // Start cycling if multiple images exist
        if (slides.length > 1) {
          setInterval(() => {
            // Fade out current
            slides[current].classList.remove('active');
            
            // Move to next (with wraparound)
            current = (current + 1) % slides.length;
            
            // Fade in next
            slides[current].classList.add('active');
          }, interval);
        }

        // Log success to browser console (for debugging)
        console.log(`Slideshow initialized: ${slides.length} images, ${interval}ms interval`);
        
      } catch (err) {
        console.error('Failed to initialize slideshow:', err);
        document.getElementById('loading').textContent = 'Error loading slideshow';
      }
    }

    // Start on page load
    init();
  </script>
</body>
</html>
```

**Performance Characteristics:**

- **Memory usage:** ~10-20MB per image (depends on resolution)
- **CPU usage:** Minimal (CSS transitions are GPU-accelerated)
- **Network usage:** Initial load only (images cached by browser)
- **Rendering:** 60fps smooth transitions via CSS

---

## Deployment Process

### Initial Setup (One-Time)

#### 1. Repository Creation

```bash
# If using git CLI
git init clickshare-signage
cd clickshare-signage
git checkout -b main

# Create initial structure
mkdir -p .github/workflows
mkdir -p units/conference-room-a/images
mkdir -p units/lobby/images

# Create files
touch .nojekyll
touch .github/workflows/update-manifests.yml
touch units/conference-room-a/index.html
touch units/conference-room-a/images/.gitkeep

# Initial commit
git add .
git commit -m "Initial repository setup"
git remote add origin https://github.com/AZLeBlanc/clickshare-signage.git
git push -u origin main
```

Or via GitHub Web UI (recommended for non-technical users):
1. Create new repository
2. Use "Add file" → "Create new file" for each path
3. Commit directly to main

#### 2. GitHub Pages Configuration

1. Navigate to **Settings** → **Pages**
2. Under **Source**, select **Deploy from a branch**
3. Branch: `main`, Folder: `/ (root)`
4. Click **Save**
5. Wait 1-2 minutes for initial deployment
6. Verify at: `https://azleblanc.github.io/clickshare-signage/`

#### 3. Workflow Permissions

1. Navigate to **Settings** → **Actions** → **General**
2. Scroll to **Workflow permissions**
3. Select **Read and write permissions**
4. Click **Save**

This allows the GitHub Action to commit generated manifests back to the repository.

### Adding a New Unit

#### Via GitHub Web UI:

1. Navigate to `units/` folder
2. Click **Add file** → **Create new file**
3. Enter path: `new-unit-name/images/.gitkeep`
4. Commit with message: "Add [unit name] unit"
5. Navigate to new `units/new-unit-name/` folder
6. Click **Add file** → **Create new file**
7. Name: `index.html`
8. Copy HTML template from existing unit
9. Update `<title>` tag with unit name
10. Commit

#### Via git CLI:

```bash
# Create unit structure
mkdir -p units/new-unit-name/images
touch units/new-unit-name/images/.gitkeep

# Copy template
cp units/conference-room-a/index.html units/new-unit-name/index.html

# Edit title in index.html
sed -i 's/Conference Room A/New Unit Name/' units/new-unit-name/index.html

# Commit
git add units/new-unit-name
git commit -m "Add new-unit-name unit"
git push
```

### Configuring a ClickShare Base Unit

#### Prerequisites:

- Base Unit IP address or hostname
- Admin credentials (default: `admin` / device-specific password)
- Network access to Base Unit (port 4003)
- ClickShare firmware with Digital Signage support

#### Configuration Methods:

**Option 1: PowerShell (Windows)**

```powershell
# Set variables
$baseUnitIP = "192.168.1.100"
$unitName = "conference-room-a"
$username = "admin"
$password = "your_password_here"

# Build URL
$signageURL = "https://azleblanc.github.io/clickshare-signage/units/$unitName/"

# Create request
$headers = @{ "Content-Type" = "application/json" }
$body = @{
    enabled = $true
    url = $signageURL
} | ConvertTo-Json

# Create credentials
$securePassword = ConvertTo-SecureString $password -AsPlainText -Force
$credentials = New-Object System.Management.Automation.PSCredential($username, $securePassword)

# Send request
Invoke-RestMethod `
    -Uri "https://${baseUnitIP}:4003/v2/configuration/features/digital-signage" `
    -Method Patch `
    -Headers $headers `
    -Body $body `
    -Credential $credentials `
    -SkipCertificateCheck

Write-Host "Digital Signage configured successfully for $unitName"
```

**Option 2: curl (Linux/macOS/WSL)**

```bash
#!/bin/bash

BASE_UNIT_IP="192.168.1.100"
UNIT_NAME="conference-room-a"
USERNAME="admin"
PASSWORD="your_password_here"

SIGNAGE_URL="https://azleblanc.github.io/clickshare-signage/units/${UNIT_NAME}/"

curl -X PATCH \
  -u "${USERNAME}:${PASSWORD}" \
  -H "Content-Type: application/json" \
  -d "{\"enabled\": true, \"url\": \"${SIGNAGE_URL}\"}" \
  "https://${BASE_UNIT_IP}:4003/v2/configuration/features/digital-signage" \
  --insecure

echo "Digital Signage configured for ${UNIT_NAME}"
```

**Option 3: Web Configurator (GUI)**

1. Navigate to `https://[base-unit-ip]`
2. Log in with admin credentials
3. Go to **Applications** → **Digital Signage**
4. Enable Digital Signage
5. Enter URL: `https://azleblanc.github.io/clickshare-signage/units/[unit-name]/`
6. Click **Apply**

#### Verification:

1. Display should show slideshow within 10-30 seconds
2. If "Loading slideshow..." persists:
   - Verify Base Unit has internet access
   - Check URL is exactly correct (trailing slash matters)
   - Test URL in a standard web browser first

### Content Updates (Marketing Workflow)

#### Adding Images:

1. Navigate to `units/[unit-name]/images/` on GitHub
2. Click **Add file** → **Upload files**
3. Drag and drop image files
4. Optionally add commit message describing images
5. Click **Commit changes**
6. Wait 1-2 minutes for deployment

**Behind the scenes:**
1. GitHub receives commit (t=0s)
2. GitHub Actions triggered (t=5s)
3. Workflow executes, generates manifest (t=30s)
4. Manifest committed back to repo (t=35s)
5. GitHub Pages deployment triggered (t=40s)
6. CDN updated (t=90s)
7. ClickShare fetches new manifest on next refresh

#### Removing Images:

1. Navigate to `units/[unit-name]/images/`
2. Click on image file to delete
3. Click trash icon (🗑️) in top right
4. Confirm deletion
5. Commit with message

#### Reordering Images:

Images display in alphabetical order by filename. To reorder:

1. Download images locally
2. Rename with numeric prefixes:
   - `01-first.jpg`
   - `02-second.jpg`
   - `03-third.jpg`
3. Delete old files from GitHub
4. Upload renamed files

**Pro tip:** Use two-digit prefixes (01, 02, ..., 10, 11) to maintain sort order beyond 9 items.

#### Adjusting Timing (Per-Unit):

Each unit's `images.json` can have custom timing:

1. Navigate to `units/[unit-name]/images.json`
2. Click edit (pencil icon)
3. Modify values:
```json
   {
     "interval": 10000,    // milliseconds per slide (10 seconds)
     "transition": 1500,   // crossfade duration (1.5 seconds)
     "images": [...]       // DO NOT edit this array
   }
```
4. Commit changes

**Note:** Only edit `interval` and `transition`. The `images` array is auto-generated.

**Recommended values:**
- High-traffic areas (lobby): `interval: 5000-8000`
- Conference rooms: `interval: 12000-15000`
- Executive areas: `interval: 20000-30000`
- Transition: `1000-3000` (faster = more dynamic, slower = more elegant)

---

## Security Considerations

### Repository Access Control

**Current State:** Public repository (required for free GitHub Pages)

**Implications:**
- All images are publicly accessible via direct URL
- Anyone can view the repository contents
- Only authorized collaborators can commit changes

**Recommendations:**
- ✅ Use for general marketing materials, company branding, public information
- ❌ Avoid for confidential data, unreleased products, internal-only content
- ⚠️ Consider private repository ($4/month GitHub Pro) if content sensitivity increases

**Access Management:**
1. Navigate to **Settings** → **Collaborators**
2. Add marketing team members with **Write** access
3. IT maintains **Admin** access
4. Use branch protection rules to require pull request reviews (optional)

### ClickShare API Security

**Authentication:**
- HTTP Basic Authentication (username + password)
- Credentials transmitted over HTTPS
- No token expiration (session-based)

**Best Practices:**
- ✅ Change default admin password on all Base Units
- ✅ Use unique passwords per unit (or password manager)
- ✅ Store credentials in secure location (not in scripts committed to git)
- ✅ Use environment variables or credential managers for automation
- ❌ Never commit credentials to repository
- ❌ Never share credentials via email or chat

**Network Security:**
- Base Units should be on management VLAN
- Restrict API access (port 4003) to IT subnet
- Consider firewall rules for production environment

### HTTPS Certificate Handling

**GitHub Pages:**
- Automatic Let's Encrypt certificate
- No configuration required
- Renewal handled by GitHub

**ClickShare Base Units:**
- Self-signed certificate by default (causes browser warnings)
- Can upload custom certificate via API
- For production, consider replacing with CA-signed certificate

**API Calls:**
- `--insecure` flag required when using self-signed certificate
- This disables certificate verification (acceptable for internal network)
- For automation, consider certificate pinning or custom CA trust

### Content Validation

**Current State:** No automated validation

**Risks:**
- Malicious file uploads (if repository access compromised)
- Oversized images causing performance issues
- Invalid file types

**Mitigation Options (Future Enhancement):**
- GitHub Actions workflow to validate:
  - File size limits (e.g., max 5MB per image)
  - Image dimensions (e.g., max 4K resolution)
  - File type verification (magic number check, not just extension)
- Pre-commit hooks for local validation
- Required pull request reviews before merge

### Secrets Management

**GitHub Secrets:**
- Used for storing sensitive configuration (if needed)
- Encrypted at rest
- Never exposed in logs
- Access via `${{ secrets.SECRET_NAME }}` in workflows

**Current Usage:** Not currently storing secrets (not needed for public repo approach)

**Future Use Cases:**
- API credentials for external services
- Webhook signing keys
- Private repository access tokens

---

## Maintenance & Operations

### Routine Maintenance Tasks

#### Weekly:
- ✅ None required (fully automated)

#### Monthly:
- Review GitHub Actions usage (check against free tier limits: 2000 minutes/month)
- Verify all units are displaying correctly
- Check for outdated/unused images across units

#### Quarterly:
- Audit repository access (remove departed employees)
- Review and archive old content
- Update documentation if workflows change

### Monitoring & Observability

**GitHub Actions:**
- **Location:** Repository → Actions tab
- **Visibility:** All workflow runs, success/failure status, execution logs
- **Notifications:** Email on workflow failure (configurable per user)

**Key Metrics:**
- Workflow success rate (should be 100%)
- Execution time (typical: 15-30 seconds)
- Actions minutes consumed (track against 2000/month limit)

**GitHub Pages:**
- **Location:** Repository → Settings → Pages
- **Visibility:** Deployment status, last deployment time, site URL
- **Metrics:** Not provided by GitHub (consider Google Analytics if needed)

**ClickShare Base Units:**
- **Monitoring:** Via ClickShare web configurator or Barco XMS (if deployed)
- **Health Checks:** Periodic API calls to verify configuration
- **Display Verification:** Physical inspection or remote desktop (if available)

### Troubleshooting Procedures

#### Issue: Images not updating on display

**Symptoms:** New images uploaded to GitHub but not showing on ClickShare

**Diagnostic Steps:**
1. Check GitHub Actions tab for failed workflows
2. Verify `images.json` was updated with new image paths
3. Check GitHub Pages deployment status
4. Test URL in web browser (hard refresh: Ctrl+Shift+R)
5. Check ClickShare network connectivity
6. Verify Digital Signage URL configuration on Base Unit

**Common Causes:**
- **Browser caching:** ClickShare caching old `images.json`
  - **Solution:** Wait 5 minutes for cache expiration, or reboot Base Unit
- **Workflow failure:** Python script error, permission issue
  - **Solution:** Review workflow logs, check file permissions
- **Incorrect filename:** Hidden file, wrong extension
  - **Solution:** Verify filename matches allowed extensions

#### Issue: GitHub Action failing

**Symptoms:** Workflow shows red X, email notification of failure

**Diagnostic Steps:**
1. Open failed workflow run
2. Expand failed step
3. Read error message and traceback
4. Check recently committed files for issues

**Common Causes:**
- **Permission denied:** Workflow lacks write permission
  - **Solution:** Enable write permissions in repo settings
- **Syntax error:** Malformed `images.json` manually edited
  - **Solution:** Delete `images.json`, let workflow regenerate
- **File not found:** Image deleted but referenced elsewhere
  - **Solution:** Clear repo, restart from clean state

#### Issue: Slow performance

**Symptoms:** Slideshow laggy, transitions stuttering, high CPU usage

**Diagnostic Steps:**
1. Check image file sizes
2. Check image dimensions
3. Test on different ClickShare model

**Common Causes:**
- **Oversized images:** 10MB+ files
  - **Solution:** Compress images to <2MB (online tools or Photoshop)
- **High resolution:** 8K images on 1080p display
  - **Solution:** Resize to native display resolution (1920×1080 or 3840×2160)
- **Too many images:** 50+ images in single unit
  - **Solution:** Limit to 20-30 images, rotate content seasonally

#### Issue: Display shows blank/black screen

**Symptoms:** No wallpaper, no slideshow, black screen when idle

**Diagnostic Steps:**
1. Check ClickShare power management settings
2. Verify Digital Signage is enabled via API or web UI
3. Check HDMI connection to display
4. Test URL in web browser on computer

**Common Causes:**
- **Digital Signage disabled:** Feature turned off in settings
  - **Solution:** Re-enable via API or web configurator
- **Invalid URL:** Typo, missing trailing slash, wrong unit name
  - **Solution:** Verify URL exactly matches: `https://azleblanc.github.io/clickshare-signage/units/[unit-name]/`
- **Network issue:** No internet access, DNS failure
  - **Solution:** Check Base Unit network configuration

### Backup & Recovery

**Repository Backup:**
- **Automatic:** GitHub maintains backups (enterprise-grade infrastructure)
- **Manual:** Clone repository locally for offline backup
```bash
  git clone https://github.com/AZLeBlanc/clickshare-signage.git
  cd clickshare-signage
  git pull --all  # Fetch all branches
```
- **Frequency:** Recommended monthly for critical content

**Recovery Procedures:**

**Scenario 1: Accidental file deletion**
```bash
# View commit history
git log --oneline -- path/to/deleted/file

# Restore file from previous commit
git checkout <commit-hash> -- path/to/deleted/file

# Commit restoration
git commit -m "Restore accidentally deleted file"
git push
```

**Scenario 2: Bad commit (wrong images uploaded)**
```bash
# Revert last commit (creates new commit that undoes changes)
git revert HEAD

# Or reset to previous state (rewrites history, use with caution)
git reset --hard HEAD~1
git push --force
```

**Scenario 3: Corrupted repository**
1. Delete local repository
2. Re-clone from GitHub
3. If GitHub repo is corrupted (extremely rare):
   - Restore from local backup
   - Or create new repository and push backup

**Base Unit Configuration Backup:**
- Export configuration via ClickShare web UI
- Or query via API and save JSON:
```bash
  curl -u admin:password \
    https://base-unit-ip:4003/v2/configuration/features/digital-signage \
    --insecure > unit-config-backup.json
```

### Change Management

**Adding New Features:**
1. Create feature branch: `git checkout -b feature/new-capability`
2. Implement and test changes
3. Create pull request for review
4. Merge to main after approval
5. Deploy via standard process

**Breaking Changes:**
1. Document change in README and this document
2. Test in isolated unit first
3. Communicate to stakeholders
4. Schedule deployment window
5. Monitor for issues after deployment
6. Have rollback plan ready

**Version Control Best Practices:**
- Use meaningful commit messages: "Add lobby unit images" not "update"
- Reference issue numbers if using GitHub Issues: "Fixes #42"
- Tag major releases: `git tag v1.0.0`
- Use pull requests for significant changes (optional for small teams)

---

## Cost Analysis

### Current Costs

| Component | Cost | Notes |
|-----------|------|-------|
| GitHub Repository | $0 | Free for public repositories |
| GitHub Pages | $0 | Free hosting (100GB/month bandwidth) |
| GitHub Actions | $0 | 2000 minutes/month free tier |
| ClickShare Hardware | Existing | Already deployed infrastructure |
| ClickShare Licenses | Existing | No additional licensing required |
| IT Labor (setup) | ~2-4 hours | One-time implementation |
| IT Labor (monthly) | ~0-1 hours | Minimal ongoing maintenance |

**Total Monthly Cost:** $0

### Cost Projections

**Scenario: Scaling to 50 Base Units**

| Component | Free Tier | Usage @ 50 Units | Cost if Exceeded |
|-----------|-----------|------------------|------------------|
| GitHub Pages Bandwidth | 100 GB/month | ~2 GB/month | N/A (under limit) |
| GitHub Actions Minutes | 2000 min/month | ~250 min/month | $0.008/minute |
| Repository Storage | 1 GB | ~100 MB | N/A (under limit) |

**Assumptions:**
- Average 10 images per unit @ 500KB each = 5MB per unit
- 50 units × 5MB = 250MB repository size
- Each unit updated 2x/month = 100 workflow runs/month
- Each workflow run = 30 seconds = 50 minutes total

**Projected Monthly Cost @ 50 Units:** $0

**Break-even Analysis:**
- Free tier supports up to ~4000 image uploads/month
- Or ~240 units with 2 updates/month each
- Exceeding limits: GitHub Team ($4/user/month) increases all limits 10x

### Cost Comparison: Alternative Solutions

| Solution | Setup Cost | Monthly Cost | Limitations |
|----------|------------|--------------|-------------|
| **This Solution** | $0 | $0 | Public repo |
| AWS S3 + CloudFront | $0 | $1-5 | Requires AWS account, config complexity |
| Azure Blob + CDN | $0 | $1-5 | Requires Azure account, config complexity |
| On-premises Web Server | $500-2000 | $20-50 | Hardware, power, maintenance |
| Barco XMS Cloud | $0 (if licensed) | Varies | Requires XMS deployment |
| Digital Signage CMS | $500-5000 | $50-500 | Per-screen licensing, vendor lock-in |

**Value Proposition:**
- Zero infrastructure investment
- Zero recurring costs
- Enterprise-grade hosting (GitHub's CDN)
- Version control and audit trail included
- No vendor lock-in (standard web technologies)

### Hidden Cost Considerations

**Potential Cost Drivers:**

1. **Private Repository:** $4/month (GitHub Team, first user)
   - Required if content must be confidential
   - Includes additional collaboration features

2. **Additional Collaborators:** $4/user/month (GitHub Team)
   - Only if using private repository
   - Public repos support unlimited collaborators

3. **Training Time:** 1-2 hours per marketing team member
   - One-time investment
   - Minimal learning curve for web-based UI

4. **Content Creation:** Variable
   - Photography/design work for signage images
   - Existing workflow, not unique to this solution

5. **Network Bandwidth:** Negligible
   - ClickShare fetches updates over existing corporate internet
   - Each unit: <1MB/day typical usage

**Total Cost of Ownership (3 years):**
- Implementation: 4 hours × IT hourly rate
- Maintenance: 12 hours/year × 3 years × IT hourly rate
- Training: 10 users × 1 hour × average hourly rate
- Infrastructure: $0
- Software licenses: $0

**ROI Calculation:**
- **Savings:** Eliminated manual configuration time
  - Previous: ~30 minutes per update per unit
  - New: ~2 minutes per update (upload only)
  - Time saved: 28 minutes per update
  - At 2 updates/month × 10 units = 560 minutes/month saved
  - Annual savings: 112 hours of labor

---

## Scaling & Future Enhancements

### Current Capacity Limits

| Resource | Limit | Current Usage | Headroom |
|----------|-------|---------------|----------|
| Repository Size | 1 GB (soft) | ~10 MB | 99% |
| File Size | 100 MB (hard) | <5 MB | Safe |
| GitHub Pages Bandwidth | 100 GB/month | <1 GB/month | 99% |
| GitHub Actions Minutes | 2000/month | ~50/month | 97.5% |
| Base Units Supported | ~500+ | 4 (planned) | 99% |

**Scalability Assessment:** Solution can easily support 100+ Base Units without modifications or additional costs.

### Phase 2 Enhancements (Potential)

#### 1. Content Scheduling

**Feature:** Schedule images to appear only during specific dates/times

**Implementation:**
- Add metadata to `images.json`:
```json
  {
    "images": [
      {
        "path": "images/holiday-promo.jpg",
        "startDate": "2024-12-01",
        "endDate": "2024-12-31"
      }
    ]
  }
```
- Modify slideshow JavaScript to filter by current date
- Marketing uploads all content upfront, system handles scheduling

**Use Cases:**
- Holiday promotions
- Event-specific signage
- Seasonal content rotation

#### 2. Centralized Configuration Management

**Feature:** Set timing/transition globally or per-unit group

**Implementation:**
- Add `config.json` in repository root:
```json
  {
    "defaults": {
      "interval": 15000,
      "transition": 2000
    },
    "groups": {
      "high-traffic": {
        "interval": 8000,
        "units": ["lobby", "cafeteria"]
      }
    }
  }
```
- GitHub Action reads config and applies to unit manifests

**Use Cases:**
- Consistent branding across locations
- Different pacing for different unit types
- Quick global adjustments

#### 3. Content Approval Workflow

**Feature:** Require approval before content goes live

**Implementation:**
- Marketing uploads to draft branch
- Automated preview deployment to staging URL
- Reviewer approves via pull request
- Merge to main deploys to production

**Use Cases:**
- Brand compliance review
- Legal approval for regulated industries
- Executive sign-off for executive areas

#### 4. Analytics & Reporting

**Feature:** Track which images are displaying, for how long, on which units

**Implementation:**
- Add lightweight analytics beacon (e.g., Google Analytics)
- Slideshow JavaScript sends page view events per image
- Dashboard shows:
  - Images displayed per unit
  - Average display time per image
  - Most/least shown content

**Use Cases:**
- Content effectiveness measurement
- Compliance audit trail
- Optimization of rotation timing

#### 5. Dynamic Content (API Integration)

**Feature:** Display live data (weather, news, calendar)

**Implementation:**
- Create API proxy service (Cloudflare Workers, AWS Lambda)
- Fetch external data and render to image
- Cache and serve via standard slideshow mechanism

**Use Cases:**
- Weather forecasts
- Company stock price
- Meeting room schedules
- RSS feed headlines

#### 6. Multi-Region Support

**Feature:** Deploy to multiple GitHub Pages sites for geographic distribution

**Implementation:**
- Create separate repositories per region
- Use GitHub Actions to sync content
- Point regional Base Units to regional URLs

**Use Cases:**
- Global enterprise with regional content
- Reduce latency for international offices
- Regulatory compliance (data sovereignty)

#### 7. Automated Image Optimization

**Feature:** Automatically compress/resize uploaded images

**Implementation:**
- GitHub Action workflow on image upload
- Use ImageMagick or similar to:
  - Compress to <1MB
  - Resize to 1920×1080
  - Convert to optimal format
- Commit optimized version

**Use Cases:**
- Prevent performance issues from oversized images
- Reduce bandwidth usage
- Enforce consistent quality

#### 8. Hybrid Auto-Refresh (Content Watcher)

> **Status: Future planning — not currently implemented in the repo.**

**Problem:** Base Units load the page once and hold it indefinitely. New images committed to GitHub become live on the CDN within ~2 minutes, but a running display never sees them until the unit reboots or IT intervenes manually. This defeats the goal of marketing being able to update content without IT involvement.

**Why a server-push approach won't work:** GitHub Actions runs in the cloud; Base Units are on an internal corporate network. There is no inbound path from GitHub to the Base Unit's browser after a push. The content watcher must be client-side — the page has to pull.

**Feature:** The slideshow polls `images.json` on a short interval and reloads itself when the image list has changed.

**Design: poll the JSON, reload only on change**

Two timers run independently after page load:

1. **Rotation timer** — existing `setInterval` that cycles through slides (unchanged)
2. **Watcher timer** — new `setInterval` that fetches `images.json` every 2 minutes

```
page loads
  ├── init(): fetch images.json → build slides → start rotation timer
  └── startWatcher(loadedImages): start watcher timer
        ↓
        every 2 minutes:
          fetch images.json?t=<timestamp>  (cache-busted)
          compare fresh.images to loadedImages
          ├── same → do nothing, no interruption to display
          └── different → location.reload()
```

**Comparison logic:**

Images are always written in sorted order by the GitHub Actions workflow, so serializing both arrays to a string is a stable, reliable diff:

```js
JSON.stringify(fresh.images) !== JSON.stringify(loadedImages)
```

`loadedImages` is captured at init time and passed into the watcher — the watcher closes over it and always compares against the state the page originally loaded with.

**What happens on reload:**

`location.reload()` triggers a full page reload. The browser re-fetches `index.html` and `images.json` (cache-busted). Image files themselves are already cached from the first load, so the reload is fast — the browser does not re-download the JPEGs, just re-renders the slide DOM. The display goes black briefly (~1 second), then resumes with the new content.

**Failure handling:**

If the poll fetch fails (network blip, GitHub outage), the watcher catches the error, logs it, and does nothing. The current slides keep running. It retries on the next interval. The display never goes dark due to a failed poll.

**End-to-end timing:**

| Event | Time |
|---|---|
| Marketing uploads image to GitHub | t=0 |
| GitHub Actions generates new `images.json` | t~2 min |
| GitHub Pages CDN updated | t~2 min |
| Watcher picks up change (worst case) | t~4 min |
| Display reloads with new content | t~4 min |

**Scope:**

The watcher compares image lists only. Changes to `interval` or `transition` in `images.json` do not trigger a reload on their own — they take effect the next time a reload occurs for any reason. This is an acceptable tradeoff for the simplicity it buys.

**Implementation:** One new function `startWatcher(loadedImages)` called at the end of `init()`. No new files, no external dependencies — self-contained in `index.html`.

### Migration Paths

#### Option A: OneDrive Integration

**Scenario:** Marketing prefers OneDrive over GitHub

**Implementation:**
- GitHub Action fetches OneDrive folder via Microsoft Graph API
- Generates `images.json` with OneDrive download URLs
- Schedule: Run every 15 minutes to refresh expiring URLs

**Tradeoffs:**
- ➕ Familiar interface for marketing
- ➕ Mobile app upload support
- ➖ Complex Azure AD setup
- ➖ URL expiration requires frequent refresh
- ➖ Additional external dependency

See earlier discussion for detailed implementation.

#### Option B: Private Repository

**Scenario:** Content must be confidential

**Migration Steps:**
1. Purchase GitHub Team ($4/month)
2. Change repository visibility to Private
3. Deploy via GitHub Actions (not automatic Pages)
4. All functionality remains identical

**Tradeoffs:**
- ➕ Content not publicly accessible
- ➕ Advanced collaboration features
- ➖ $4/month ongoing cost
- ➖ Slightly more complex deployment setup

#### Option C: Self-Hosted GitLab

**Scenario:** Corporate policy requires on-premises git hosting

**Migration Steps:**
1. Set up GitLab server
2. Mirror repository structure
3. Configure GitLab CI/CD (equivalent to GitHub Actions)
4. Host pages via GitLab Pages or nginx

**Tradeoffs:**
- ➕ Complete control and ownership
- ➕ Integration with corporate SSO
- ➖ Server hardware/VM required
- ➖ Ongoing maintenance burden
- ➖ Initial setup complexity

#### Option D: Dedicated Digital Signage CMS

**Scenario:** Feature requirements exceed static site capabilities

**Evaluation Criteria:**
- Need real-time data integration (stock tickers, weather)
- Need video playback support
- Need interactive touch functionality
- Need enterprise SLA guarantees

**Recommended Vendors:**
- Barco XMS (native integration)
- ScreenCloud
- Yodeck
- Rise Vision

**Cost:** Typically $10-50/screen/month

### Technical Debt Considerations

**Current Technical Debt:**
- Minimal (greenfield implementation with modern practices)

**Potential Future Debt:**
- **Image size validation:** Not currently enforced
- **File naming conventions:** Assumed but not validated
- **Error handling:** Basic error messages, could be more user-friendly
- **Testing:** No automated tests (acceptable for current scale)

**Mitigation Strategy:**
- Address validation as requirements arise
- Implement testing if team size grows beyond 2-3 contributors
- Refactor to TypeScript if JavaScript codebase becomes complex

---

## Appendices

### Appendix A: ClickShare REST API Reference

**Base URL:** `https://[base-unit-ip]:4003`

**API Version:** v2

**Full Documentation:** Available at `https://[base-unit-ip]:4003/api-docs`

**Relevant Endpoints:**

#### GET /v2/configuration/features/digital-signage

Retrieve current Digital Signage configuration.

**Request:**
```http
GET /v2/configuration/features/digital-signage HTTP/1.1
Host: 192.168.1.100:4003
Authorization: Basic YWRtaW46cGFzc3dvcmQ=
```

**Response (200 OK):**
```json
{
  "enabled": true,
  "url": "https://azleblanc.github.io/clickshare-signage/units/lobby/"
}
```

#### PATCH /v2/configuration/features/digital-signage

Update Digital Signage configuration.

**Request:**
```http
PATCH /v2/configuration/features/digital-signage HTTP/1.1
Host: 192.168.1.100:4003
Authorization: Basic YWRtaW46cGFzc3dvcmQ=
Content-Type: application/json

{
  "enabled": true,
  "url": "https://azleblanc.github.io/clickshare-signage/units/lobby/"
}
```

**Response (200 OK):**
```json
{
  "enabled": true,
  "url": "https://azleblanc.github.io/clickshare-signage/units/lobby/"
}
```

**Error Responses:**
- `400 Bad Request` — Invalid URL format or malformed JSON
- `401 Unauthorized` — Invalid credentials
- `404 Not Found` — Digital Signage not supported on this model

#### Other Useful Endpoints

**GET /v2/configuration/system/status**
```json
{
  "currentUptime": 3600,
  "inUse": true,
  "sharing": false,
  "errorCode": "Ok"
}
```

**POST /v2/operations/reboot**
- Reboots the Base Unit
- Use to force display refresh if content not updating

### Appendix B: Git Workflow Cheat Sheet

**For Marketing Team (GitHub Web UI):**
