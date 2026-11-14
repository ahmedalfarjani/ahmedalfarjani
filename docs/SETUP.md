# Profile README setup

This repository renders the GitHub profile for **Ahmed Mohamed Al Farjani** (`ahmedalfarjani/ahmedalfarjani`). It is designed for GitHub dark and light modes and updates its data through GitHub Actions.

## Architecture

| Section | Source | Runtime |
|---------|--------|---------|
| Stats, top languages, repository pins | [`stats-organization/github-readme-stats-action`](https://github.com/stats-organization/github-readme-stats-action) (maintained successor to `github-readme-stats`) | Generated in Actions and committed to `profile/` |
| Contribution activity graph | [`maurodesouza/github-readme-activity-graph-action`](https://github.com/maurodesouza/github-readme-activity-graph-action) (wraps `github-readme-activity-graph`) | Generated in Actions and committed to `profile/` |
| Isometric contribution calendar | [`lowlighter/metrics`](https://github.com/lowlighter/metrics) (`plugin_isocalendar`) | Generated in Actions and committed to the root |
| Contribution snake | [`Platane/snk`](https://github.com/Platane/snk) | Generated in Actions and pushed to the `output` branch |
| Streak card | [`github-readme-streak-stats`](https://github.com/DenverCoder1/github-readme-streak-stats) | Live SVG from `streak-stats.demolab.com` |
| Trophy case | [`github-profile-trophy`](https://github.com/ryo-ma/github-profile-trophy) | Live SVG from the official volunteer mirror `trophygithubreadmelang.cybee.dpdns.org` |
| Technology icons | [`tandpfun/skill-icons`](https://github.com/tandpfun/skill-icons) | Live SVG from `skillicons.dev` |

The original public instances of `github-readme-stats`, `github-profile-trophy`, and `github-readme-activity-graph` were paused or disabled on Vercel during 2026. This repository avoids those single points of failure by generating their SVGs in GitHub Actions, except for the trophy mirror and streak endpoint, which were verified live on 14 November 2026.

## Files

```
.
├── README.md
├── github-metrics-isocalendar.svg      # generated
├── profile/
│   ├── stats-dark.svg                  # generated
│   ├── stats-light.svg                 # generated
│   ├── top-langs-dark.svg              # generated
│   ├── top-langs-light.svg             # generated
│   ├── pin-deep-learning-dark.svg      # generated
│   ├── pin-deep-learning-light.svg     # generated
│   ├── pin-inventree-dark.svg          # generated
│   ├── pin-inventree-light.svg         # generated
│   ├── activity-dark.svg               # generated
│   └── activity-light.svg              # generated
├── .github/workflows/
│   ├── metrics.yml                     # isometric calendar
│   ├── cards.yml                       # stats, languages, pins, activity
│   └── snake.yml                       # contribution snake
└── docs/SETUP.md
```

Generated files do not exist until the workflows run for the first time.

## 1. Create the profile repository

The repository must be public and named exactly like the account:

```powershell
gh auth status
gh repo create ahmedalfarjani/ahmedalfarjani --public --source=. --remote=origin --push
```

If the repository already exists, push normally:

```powershell
git init
git add .
git commit -m "feat: add profile README and automation"
git branch -M main
git remote add origin https://github.com/ahmedalfarjani/ahmedalfarjani.git
git push -u origin main
```

## 2. Create a token for private-aware stats (recommended)

The workflows fall back to `GITHUB_TOKEN`, which can read public data only. A personal access token lets `lowlighter/metrics` and the stats action access private contribution counts and private repository metadata.

1. Open GitHub **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic)**.
2. Select scopes:
   - `repo` — private repository metadata
   - `read:user` — profile and contribution data
   - `read:org` — organization membership, if needed
3. Generate the token and store it as the repository secret `METRICS_TOKEN`:

```powershell
gh secret set METRICS_TOKEN --repo ahmedalfarjani/ahmedalfarjani
```

Paste the token when prompted. Do not commit it.

Fine-grained tokens also work if they are granted read access to the account's repositories and profile data.

## 3. Enable workflow write access

1. Open **Settings → Actions → General**.
2. Under **Workflow permissions**, select **Read and write permissions**.
3. Save.

The workflows also declare `permissions: contents: write`, but the repository-level setting must allow it.

## 4. Run the workflows

The `push` trigger starts all three workflows after the first push. To run them manually:

```powershell
gh workflow run metrics.yml --repo ahmedalfarjani/ahmedalfarjani
gh workflow run cards.yml --repo ahmedalfarjani/ahmedalfarjani
gh workflow run snake.yml --repo ahmedalfarjani/ahmedalfarjani
gh run list --repo ahmedalfarjani/ahmedalfarjani --limit 5
```

Wait for the runs to finish, then confirm the generated files:

```powershell
gh api repos/ahmedalfarjani/ahmedalfarjani/contents/github-metrics-isocalendar.svg --jq '.size'
gh api repos/ahmedalfarjani/ahmedalfarjani/contents/profile/stats-dark.svg --jq '.size'
gh api repos/ahmedalfarjani/ahmedalfarjani/contents/output/github-snake-dark.svg --jq '.size'
```

The snake workflow publishes to a separate `output` branch. The raw URLs in `README.md` point there.

## 5. Private contributions

To include private contributions in the public contribution graph:

1. Open **Profile → Contributions & Activity → Contribution settings**.
2. Enable **Private contributions**.

Private repository names and code remain private. The stats cards currently run with `count_private` disabled by default. To include private counts, add `&count_private=true` to the stats card options in `.github/workflows/cards.yml` and make sure `METRICS_TOKEN` is configured.

## 6. Dark and light mode

The README uses `<picture>` with `prefers-color-scheme` so GitHub serves the correct variant:

- Local generated SVGs: `profile/*-dark.svg` and `profile/*-light.svg`
- Live widgets: separate dark and light URLs for streak, trophies, and skill icons
- `lowlighter/metrics` and the snake SVGs are theme-adapted in their own workflows

Preview both modes in a browser or in GitHub by toggling **Settings → Appearance**. GitHub may cache images; allow a few minutes after a workflow run for changes to appear.

## 7. Validation

Render the Markdown with GitHub's own renderer:

```powershell
gh api --method POST /markdown -f mode=gfm -F text=@README.md > preview.html
```

Check live widget endpoints:

```powershell
$urls = @(
  'https://streak-stats.demolab.com?user=ahmedalfarjani',
  'https://trophygithubreadmelang.cybee.dpdns.org/?username=ahmedalfarjani',
  'https://skillicons.dev/icons?i=py,cpp,matlab,postgres,tensorflow,sklearn,arduino,ts,git,github'
)
foreach ($url in $urls) {
  try {
    $response = Invoke-WebRequest -UseBasicParsing -Uri $url -TimeoutSec 30
    "$($response.StatusCode) $url"
  }
  catch {
    "ERROR $url"
  }
}
```

Expected: all URLs return `200` with `image/svg+xml`.

## 8. Maintenance

| Workflow | Schedule | Writes to |
|----------|----------|-----------|
| `metrics.yml` | Daily at 03:00 UTC | `main` (`github-metrics-isocalendar.svg`) |
| `cards.yml` | Daily at 03:30 UTC | `main` (`profile/*.svg`) |
| `snake.yml` | Daily at 04:00 UTC | `output` branch |

`metrics.yml` and `cards.yml` share the `profile-readme` concurrency group so their commits do not race. `snake.yml` uses a separate group.

To update the schedule, edit the `cron` entries. All schedules use UTC.

## 9. Troubleshooting

### A generated image is missing or broken

- Open **Actions** and inspect the most recent run.
- Confirm the workflow has `contents: write` permission.
- Re-run the workflow manually.
- Wait a few minutes for GitHub's image proxy cache to refresh.

### Metrics fails with an authentication or rate-limit error

- Confirm `METRICS_TOKEN` exists and is not expired.
- Confirm the token has `repo` and `read:user` scopes.
- Re-run the workflow.

### Snake images return 404

- Confirm the `Snake` workflow succeeded.
- Confirm an `output` branch exists and contains `github-snake.svg` and `github-snake-dark.svg`.
- Re-run the workflow; it recreates the branch output.

### Stats action fails

The action writes an error card by default. Set `fail_on_error: true` on the action steps to fail the job instead, then inspect the logs. As a temporary fallback, the maintained public instance is available at:

```
https://github-stats-extended.vercel.app/api?username=ahmedalfarjani
```

### Trophy mirror is unavailable

The README uses a volunteer mirror listed in the official `ryo-ma/github-profile-trophy` repository. Replace the host with another listed endpoint:

- `https://github-profile-trophy-unserori.vercel.app/`
- `https://trophygithubreadmelang.cybee.dpdns.org/`

Or deploy the official project to your own Vercel account.

### Activity graph is unavailable

The graph is generated locally by `maurodesouza/github-readme-activity-graph-action`. If that action breaks, temporarily point the README at the verified fallback:

```
https://github-readme-activity-graph-delta.vercel.app/graph?username=ahmedalfarjani
```

Or self-host `Ashutosh00710/github-readme-activity-graph`.

### Streak card returns 403 or does not load

Try the alternate official endpoint:

```
https://github-readme-streak-stats.herokuapp.com?user=ahmedalfarjani
```

Or self-host `DenverCoder1/github-readme-streak-stats`.

## 10. Privacy

The README publishes the email address, phone/WhatsApp number, and location supplied for the profile. Remove or mask any of them in `README.md` if they should not be public.

Private repository names are listed without links. If they should not be named publicly, remove the **Private repositories** block from `README.md`.

## 11. Widget validation status

Checked on 14 November 2026:

| Endpoint | Status |
|----------|--------|
| `streak-stats.demolab.com` | 200, `image/svg+xml` |
| `skillicons.dev` | 200, `image/svg+xml` |
| `trophygithubreadmelang.cybee.dpdns.org` | 200, `image/svg+xml` |
| `github-stats-extended.vercel.app` | 200, `image/svg+xml` |
| `github-readme-activity-graph-delta.vercel.app` | 200, `image/svg+xml` |
| `github-readme-stats.vercel.app` | 503, deployment paused (not used) |
| `github-profile-trophy.vercel.app` | 402, deployment disabled (not used) |
| `github-readme-activity-graph.vercel.app` | 402, deployment disabled (not used) |
