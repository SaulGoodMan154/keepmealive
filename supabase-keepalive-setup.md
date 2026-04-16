# Supabase Keepalive — GitHub Actions Automation
## Technical Implementation Document

---

## Overview

This automation prevents Supabase free-tier projects from pausing due to inactivity.
It uses a GitHub Actions scheduled workflow that calls the Supabase Management API
every 3 days to restore all projects — whether paused or active (the restore call is idempotent).

- **Total projects:** 3
- **Supabase accounts involved:** 2
- **Infrastructure required:** GitHub (free tier) only
- **Secrets required:** 2 Supabase Personal Access Tokens (PATs)

---

## Project Reference IDs

| Account | Project Ref |
|---------|-------------|
| Account A (primary Gmail) | dalknkrliiogkycwptbh |
| Account B (secondary Gmail) | desxlxesgbnsiwuphqok |
| Account B (secondary Gmail) | nzrmpmwwbuwytpaqwodp |

---

## Step 1 — Create a Private GitHub Repository

1. Go to https://github.com/new
2. Set repository name to: supabase-keepalive
3. Set visibility to: Private
4. Do NOT initialize with README or any other files
5. Click "Create repository"

---

## Step 2 — Create the Workflow File

Inside the repository, create the following file at this exact path:

    .github/workflows/keepalive.yml

File contents:

```yaml
name: Supabase Keepalive

on:
  schedule:
    - cron: '0 6 */3 * *'  # Runs every 3 days at 6AM UTC
  workflow_dispatch:         # Enables manual trigger from GitHub UI

jobs:
  restore-all:
    runs-on: ubuntu-latest

    steps:
      - name: Restore all Supabase projects
        env:
          TOKEN_ACCOUNT_A: ${{ secrets.SUPABASE_TOKEN_ACCOUNT_A }}
          TOKEN_ACCOUNT_B: ${{ secrets.SUPABASE_TOKEN_ACCOUNT_B }}
        run: |
          restore() {
            local token=$1
            local ref=$2
            local name=$3
            echo "Restoring: $name ($ref)"
            response=$(curl -s -o /dev/null -w "%{http_code}" \
              -X POST "https://api.supabase.com/v1/projects/$ref/restore" \
              -H "Authorization: Bearer $token" \
              -H "Content-Type: application/json")
            echo "$name -> HTTP $response"
          }

          # Account A — 1 project
          restore "$TOKEN_ACCOUNT_A" "dalknkrliiogkycwptbh" "Project-AccountA"

          # Account B — 2 projects
          restore "$TOKEN_ACCOUNT_B" "desxlxesgbnsiwuphqok" "Project-AccountB-1"
          restore "$TOKEN_ACCOUNT_B" "nzrmpmwwbuwytpaqwodp" "Project-AccountB-2"

          echo "All projects restore triggered."
```

---

## Step 3 — Generate Supabase Personal Access Tokens

You need to do this step manually in a browser. Generate one token per account.

For Account A:
1. Log into the Supabase account linked to your primary Gmail
2. Go to: https://supabase.com/dashboard/account/tokens
3. Click "Generate new token"
4. Name it: github-keepalive
5. Copy the token immediately — it will not be shown again
6. Store it temporarily in a local notepad (not online)

For Account B:
1. Log out and log into the Supabase account linked to your secondary Gmail
2. Go to: https://supabase.com/dashboard/account/tokens
3. Click "Generate new token"
4. Name it: github-keepalive
5. Copy and store this token as well

---

## Step 4 — Add Secrets to the GitHub Repository

1. Go to your supabase-keepalive repository on GitHub
2. Click "Settings" (top navigation tab)
3. In the left sidebar, click "Secrets and variables" -> "Actions"
4. Click "New repository secret" and add the following two secrets:

   Secret 1:
   - Name:  SUPABASE_TOKEN_ACCOUNT_A
   - Value: (paste the PAT generated from Account A)

   Secret 2:
   - Name:  SUPABASE_TOKEN_ACCOUNT_B
   - Value: (paste the PAT generated from Account B)

5. After saving, you should see both secrets listed (values will be hidden)

---

## Step 5 — Test the Workflow Manually

1. Go to your repository on GitHub
2. Click the "Actions" tab
3. In the left sidebar, click "Supabase Keepalive"
4. Click the "Run workflow" dropdown on the right
5. Select branch: main
6. Click "Run workflow"
7. Wait ~10 seconds and refresh the page
8. Click the running job to view logs

Expected log output:

    Restoring: Project-AccountA (dalknkrliiogkycwptbh)
    Project-AccountA -> HTTP 201
    Restoring: Project-AccountB-1 (desxlxesgbnsiwuphqok)
    Project-AccountB-1 -> HTTP 201
    Restoring: Project-AccountB-2 (nzrmpmwwbuwytpaqwodp)
    Project-AccountB-2 -> HTTP 201
    All projects restore triggered.

HTTP Response Codes:
- 201 = Project was paused, now resumed
- 200 = Project was already active, no change needed
- 401 = Token is wrong or expired — re-check the secret value
- 404 = Project ref is wrong — re-check the ref ID

---

## Step 6 — Verify Automatic Schedule

After the manual test succeeds, the workflow will run automatically every 3 days.
No further action is needed.

To confirm the schedule is active:
1. Go to Actions tab in the repository
2. You should see the last run listed with a green checkmark
3. GitHub will email you at your GitHub account email if any future run fails

---

## Emergency Manual Trigger (Use When a Recruiter Reports an Issue)

If someone reports a project is down:
1. Open GitHub on your phone or browser
2. Go to: https://github.com/YOUR_USERNAME/supabase-keepalive/actions
3. Click "Supabase Keepalive" -> "Run workflow" -> "Run workflow"
4. Projects will be back online within 30-60 seconds

---

## File Structure of the Repository

    supabase-keepalive/
    └── .github/
        └── workflows/
            └── keepalive.yml

That is the only file needed in this repository.

---

## Notes

- Do NOT commit your PATs anywhere in the codebase. Always use GitHub Secrets.
- The workflow_dispatch trigger keeps the workflow alive even if GitHub detects
  no recent pushes (GitHub disables scheduled workflows after 60 days of repo inactivity —
  make a dummy commit or manual run once every 2 months to prevent this).
- The restore API endpoint is safe to call on active projects — it does nothing if
  the project is already running.
- Supabase Management API reference: https://api.supabase.com/api/v1

---

End of Document
