# GitHub Promotion Demo v4

> This version uses a delta release package and staging-triggered auto-merge to main.

# GitHub Release Promotion Demo

This repository demonstrates a simple **build once, promote later** workflow:

```text
release/1.0.0
      |
      | push
      v
Build Release + DEV
      |
      +--> build one immutable artifact
      |
      +--> DEV automatically
              |
              | hours/days later
              v
          Promote Release
              |
              +--> UAT
              |
              | hours/days later
              v
          Promote Release
              |
              +--> STAGING
              |
              | hours/days later
              v
          Promote Release
              |
              +--> PRODUCTION
                       |
                       +--> tag v1.0.0
```

The promotion workflow never rebuilds the application. It downloads the `release-package` artifact from the **original Build Release + DEV workflow run** using that run's ID.

## Repository structure

```text
.github/workflows/
  ci.yml
  build-release.yml
  promote.yml
app/
  index.html
README.md
```

## 1. Create the repository on GitHub

Create an empty repository, then push this sample into it.

Example with GitHub CLI:

```bash
git init
git add .
git commit -m "Initial promotion demo"
git branch -M main
gh repo create github-promotion-demo --public --source=. --remote=origin --push
```

A public repository is the easiest way to test GitHub Environment approval rules across GitHub plans. If you use a private repository, environment/protection-rule availability depends on your GitHub plan.

## 2. Create GitHub Environments

In GitHub:

1. Open the repository.
2. Go to **Settings > Environments**.
3. Create these four environments exactly:
   - `dev`
   - `uat`
   - `staging`
   - `production`

For the first test:

- `dev`: no approval rule.
- `uat`: optionally add yourself as a Required reviewer.
- `staging`: optionally add yourself as a Required reviewer.
- `production`: optionally add yourself as a Required reviewer.

If you are testing alone, do **not** enable **Prevent self-review**, otherwise you may not be able to approve your own deployment.

No environment secrets are required because the sample only simulates deployment.

## 3. Create a release branch

From `main`:

```bash
git switch -c release/1.0.0
```

Edit `app/index.html` if you want to make the release obvious, then:

```bash
git add .
git commit -m "Prepare release 1.0.0"
git push -u origin release/1.0.0
```

## 4. Watch the automatic build and DEV deployment

The push to `release/1.0.0` triggers:

```text
.github/workflows/build-release.yml
```

It will:

1. Read version `1.0.0` from the branch name.
2. Find the latest reachable production tag (`v*`) and calculate the cumulative Git delta from that tag to the release commit.
3. Copy only added, modified, type-changed, copied, and renamed-to files under `app/` into the delta package.
4. Record deleted and renamed-away paths in `deleted-files.txt`.
5. Create `manifest.json`, `changes.json`, and `release-package.zip`.
6. Create a SHA-256 checksum.
7. Upload the files as the `release-package` GitHub Actions artifact.
8. Download that exact artifact into the DEV deployment job.
9. Simulate applying the delta to DEV.
10. Create a GitHub deployment record for the `dev` environment.

Open **Actions > Build Release + DEV > the latest run**.

Open the workflow summary and copy the value shown as:

```text
Build run ID: 12345678901
```

That number identifies the immutable build you will promote later.

## 5. Wait as long as your DEV testing requires

Nothing needs to keep running.

For example:

```text
Monday 10:00   build 1.0.0
Monday 10:05   DEV deployed
Monday-Friday  DEV testing
Friday 14:00   promote same build to UAT
```

The build artifact is retained for 90 days in this sample.

## 6. Promote the exact build to UAT

In GitHub:

1. Go to **Actions**.
2. Select **Promote Release**.
3. Click **Run workflow**.
4. Leave the workflow branch as `main`.
5. Enter the Build run ID copied earlier.
6. Select `uat`.
7. Click **Run workflow**.

The promotion workflow downloads `release-package` from that original run ID.

If the `uat` environment has Required reviewers, the job pauses until approved.

After approval, it verifies the checksum and simulates UAT deployment.

## 7. Promote the same build to STAGING

Hours or days later, run **Promote Release** again:

```text
Build run ID:      SAME ORIGINAL RUN ID
Target environment: staging
```

Do not create another build.

## 8. Promote the same build to PRODUCTION

After staging approval, run it again:

```text
Build run ID:      SAME ORIGINAL RUN ID
Target environment: production
```

After the simulated production deployment succeeds, the workflow creates:

```text
v1.0.0
```

on the exact commit recorded in the original build manifest.

## What promotion actually means

A promotion is not a Git merge and it is not another build.

It is essentially:

```text
original build run 12345678901
        |
        | contains release-package
        v
Download exact artifact
        |
Verify checksum
        |
Read version + commit from manifest
        |
Deploy artifact to selected GitHub Environment
```

The only changing value is the target environment.

## Test that the build really stays unchanged

After DEV has deployed release `1.0.0`, make another commit on the release branch:

```bash
echo "<!-- second build -->" >> app/index.html
git add app/index.html
git commit -m "Change release candidate"
git push
```

GitHub creates a **new** Build Release + DEV run with a different run ID and commit SHA.

You now have two immutable candidates:

```text
Build run 111111  -> commit AAA -> old candidate
Build run 222222  -> commit BBB -> new candidate
```

If UAT was testing build `111111`, continue promoting `111111` unless you intentionally restart testing with `222222`.

This demonstrates why the Build run ID matters.

## Real server deployment

The sample currently performs this step:

```text
unzip the delta package and display the files that would be copied/overwritten and deleted
```

Replace the `Simulate deployment to selected environment` step with your real deployment method, for example:

```text
GitHub-hosted runner
       |
       +--> SFTP / SCP --> server

or

self-hosted Windows runner installed near/on server
       |
       +--> PowerShell copy/deployment script
```

Use environment-specific GitHub secrets/variables for values such as server host, destination path, account name, or credentials.

Example concept:

```text
dev environment
  SERVER_HOST=dev.example.org
  DEPLOY_PATH=C:\\sites\\myapp-dev

uat environment
  SERVER_HOST=uat.example.org
  DEPLOY_PATH=C:\\sites\\myapp-uat

staging environment
  SERVER_HOST=staging.example.org
  DEPLOY_PATH=C:\\sites\\myapp-staging

production environment
  SERVER_HOST=prod.example.org
  DEPLOY_PATH=C:\\sites\\myapp
```

The workflow remains the same because `${{ inputs.target_environment }}` selects which GitHub Environment supplies the variables/secrets.

## Current sample limitations

This first version intentionally keeps the promotion logic easy to understand.

It does **not** enforce that UAT must have succeeded before STAGING, or that STAGING must have succeeded before PRODUCTION. A user with permission could manually choose `production` immediately.

That is the next hardening step after you verify this basic model. A production implementation can validate previous successful deployment records before allowing the next promotion.

## Artifact retention

This sample sets artifact retention to 90 days. If your release approval process can exceed artifact retention, move immutable packages to longer-lived storage such as a package registry or a release-asset repository rather than rebuilding them.


## V2: automatic merge to `main` after STAGING

This version adds the missing release-finalization behavior.

### Before you start testing the release

Create the release branch and push it:

```bash
git switch -c release/1.0.0
git push -u origin release/1.0.0
```

Then create **one pull request** and leave it open:

```text
release/1.0.0 -> main
```

You can create it in the GitHub UI or with:

```bash
gh pr create --base main --head release/1.0.0 --title "Release 1.0.0" --body "Release candidate 1.0.0"
```

Do **not** merge it yet.

### Enable repository auto-merge

In GitHub:

```text
Repository -> Settings -> General -> Pull Requests -> Allow auto-merge
```

The STAGING promotion uses `gh pr merge --auto --merge` so GitHub merges the release PR automatically when its merge requirements are satisfied.

### New promotion sequence

```text
release/1.0.0
      |
      +--> BUILD ONCE
      |       |
      |      DEV
      |       |
      |      UAT
      |       |
      |    STAGING
      |       |
      |       +--> verify open PR head == tested artifact commit
      |       |
      |       +--> enable auto-merge: release/1.0.0 -> main
      |                         |
      |                         v
      |                       main
      |                         |
      +-------------------------+
                                |
                         production promotion
                                |
                    verify tested commit is in main
                                |
                                v
                           PRODUCTION
                                |
                                v
                             v1.0.0
```

### Important safety checks

The STAGING promotion refuses to enable auto-merge if the release branch has changed since the artifact was built. It compares the PR head SHA with the SHA stored in `manifest.json`.

The PRODUCTION promotion refuses to deploy unless the exact tested release commit is already contained in `main`.

This means production cannot accidentally deploy a release that was never integrated into `main`.

### Why the workflow does not create the PR itself

For this test repository, create the release PR manually once. GitHub normally suppresses new workflow runs for events caused by the workflow's own `GITHUB_TOKEN`. Creating the PR outside the workflow ensures your normal pull-request checks can run as expected.



## V3: changed-files-only delta release package

The release artifact is now a **delta from the latest production tag** reachable from the release branch. For example:

```text
v1.0.0 (currently in production)
   |
   +-- release/1.1.0
          |
          +-- modified: handlers/User.cfc
          +-- added:    views/report.cfm
          +-- renamed:  js/old.js -> js/new.js
          +-- deleted:  views/obsolete.cfm
```

The resulting artifact contains only the deployment delta:

```text
release-package.zip
  files/
    handlers/User.cfc
    views/report.cfm
    js/new.js
  changed-files.txt
  deleted-files.txt
  changes.json
  manifest.json
```

`changed-files.txt` contains application-relative paths to copy or overwrite. `deleted-files.txt` contains application-relative paths to remove. A Git rename is represented as **copy the new path + delete the old path**.

Example manifest:

```json
{
  "version": "1.1.0",
  "commit": "abc123...",
  "sourceBranch": "release/1.1.0",
  "buildRunId": "123456789",
  "baseTag": "v1.0.0",
  "baseCommit": "def456...",
  "firstRelease": false,
  "changedFileCount": 3,
  "deletedFileCount": 2,
  "changedFiles": [
    "handlers/User.cfc",
    "views/report.cfm",
    "js/new.js"
  ],
  "deletedFiles": [
    "js/old.js",
    "views/obsolete.cfm"
  ]
}
```

### First release

If the repository has no reachable `v*` production tag yet, the workflow compares the release commit with Git's empty tree. Therefore the first release contains every tracked file under `app/`.

### Important deployment rule for delta packages

A delta must be applied to the version it was built from:

```text
server currently at v1.0.0
        +
v1.0.0 -> v1.1.0 delta
        =
valid
```

Do not apply the same delta to an unknown or older server state without first bringing that server to the expected base version. The real server deployment step should verify `manifest.baseTag` / `manifest.baseCommit` before copying or deleting files.

### Promotion is unchanged

The same immutable delta artifact still moves through the complete release path:

```text
release/1.1.0
      |
      v
BUILD DELTA ONCE
      |
      v
DEV
      |
      v
UAT
      |
      v
STAGING
      |
      +--> auto-merge release PR -> main
      |
      v
PRODUCTION
      |
      v
v1.1.0
```

No environment rebuilds the delta package.
