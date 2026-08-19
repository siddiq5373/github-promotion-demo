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
2. Package `app/`.
3. Create `build-info.json` containing the exact commit SHA and build run ID.
4. Create `release-package.zip`.
5. Create a SHA-256 checksum.
6. Upload the files as the `release-package` GitHub Actions artifact.
7. Download that exact artifact into the DEV deployment job.
8. Simulate deployment to DEV.
9. Create a GitHub deployment record for the `dev` environment.

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
unzip release-package.zip into a temporary runner directory
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
