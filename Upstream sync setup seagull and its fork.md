# Upstream sync setup: r5-glnpo-seabird and its fork

Oct 6, 2026 · @Dustin

## Overview

Every push to `r5-glnpo-seabird`'s dev branch opens or updates a pull request in the fork. A person still reviews and merges that PR by hand.

1. A push lands on `dev` in `r5-glnpo-seabird` (the upstream).
2. The `notify-fork.yml` workflow in `r5-glnpo-seabird` sends a `repository_dispatch` event named `upstream-updated` to the fork.
3. The `sync-upstream.yml` workflow in the fork starts, fetches upstream, and force-pushes it to a branch named `sync/upstream`.
4. If no sync PR is open, the workflow creates one from `sync/upstream` into the fork's default branch. If one is open, the push updates it.

Both workflows authenticate with a GitHub App owned by the organization.


## Step 1: Create and install the GitHub App

Create the app from the organization's settings

1. Org Settings → Developer settings → GitHub Apps → **New GitHub App**.
2. Give it a name and put the org home page or any URL in the homepage field.
3. Under Webhook, untick Active.
4. Under Repository permissions, set **Contents**, **Pull requests** and **Workflows** to Read and write.
5. Click **Create GitHub App**.
6. Copy the **Client ID** from the app's General page.
7. On the same page, click **Generate a private key**. A `.pem` file downloads.
8. Open **Install App** in the app's sidebar and click **Install** next to the organization. The page calls this an "account"; an organization counts as one.
9. Choose **Only select repositories**, pick `r5-glnpo-seabird` and `r5-glnpo-seabird-rewrite`, and confirm.

To change the repo selection later: Org Settings → GitHub Apps (under Third-party Access) → **Configure** → Repository access.

## Step 2: Store the app credentials

1. Go to Org Settings → Secrets and variables → Actions.
2. On the Variables tab, click New organization variable, name it SYNC_APP_CLIENT_ID, and paste the Client ID.
3. Under Repository access, choose Selected repositories and pick `r5-glnpo-seabird` and `r5-glnpo-seabird-rewrite`.
4. Open the .pem file in a plain text editor and copy all of it.
5. On the Secrets tab, click New organization secret, name it SYNC_APP_PRIVATE_KEY, and paste the key.
6. Set Repository access to the same two repos and click Add secret.

## Step 3: Add the two workflows

Each repo gets one file under `.github/workflows/` on its default branch.

## Pitfalls

### Problems we hit during setup

| Symptom | Cause | Fix |
| --- | --- | --- |
| `gh: Not Found (HTTP 404)` in the sync workflow | The built-in `github.token` can only see the repo it runs in. GitHub answers 404, not 403, for a private repo a token can't see. | Use a token that can read the upstream: the GitHub App token here, or a personal access token. |
| Token value is empty in the workflow | It was saved as a variable but read with `secrets.`, or the other way round. | Store it under the tab that matches how the workflow reads it (see Step 3). |
| Install App page asks for an "account" and the org isn't listed | The app was created under a personal account and limited to that account. | Recreate the app from the org's settings, or transfer it: app settings → Advanced → Transfer ownership. |
| Org is listed but has no Install button | You aren't an org owner. | Ask an owner to install it. |
| The fork doesn't appear in the app's repository picker | The fork lives under a personal account, or the picker needs a search. | Type the fork's name in the picker. If it is under a personal account, re-fork into the org (Step 1). |
| "Cannot fork because forking is disabled" | Private org repos have forking off by default, at two levels. |

### Problems that may come up later

| Symptom | Cause | Fix |
| --- | --- | --- |
| The same sync PR reopens after every merge, often with conflicts | The PR was squash- or rebase-merged. The workflow checks whether upstream's latest commit is in the fork's history, and those merge styles rewrite it. | Always merge the sync PR with a regular merge commit. |
| Push to `sync/upstream` is rejected when upstream changes a workflow file | The token lacks permission to write workflow files. | Give the app the **Workflows** permission (Step 2). With a personal token, add the `workflow` scope. |
| App token step succeeds but upstream calls return 404 | The `repositories:` input is missing, so the token covers only the current repo, or the app isn't installed on that repo. | Keep both repo names under `repositories:` and check the app's repository access. |
| New app permission has no effect | Permission changes made after install need approval. | An org owner approves the request under Org Settings → GitHub Apps → Configure. |
| Token step fails with a private key or JWT error | The `.pem` contents were pasted incompletely. | Paste the whole file again, including the BEGIN and END lines. |
| Notify succeeds but nothing starts in the fork | `repository_dispatch` and `schedule` only run the workflow file on the fork's default branch, or `FORK_REPO` is out of date. | Merge `sync-upstream.yml` to the default branch and check `FORK_REPO`. |
| Sync stops after a repo is renamed | Names are hardcoded: `FORK_REPO`, `UPSTREAM_REPO`, and the `if:` check in `notify-fork.yml`. | Update all three. |
| Commits pushed to `sync/upstream` disappear | The workflow force-pushes that branch on every run. | Never commit to `sync/upstream`. Resolve conflicts in the merge, or on a separate branch. |
| Sync PR shows merge conflicts | The fork changed the same lines as upstream. | Resolve them in the PR by hand. The workflow never merges on its own. |
| Scheduled runs are late or use up Actions minutes | Scheduled runs can be delayed, the shortest interval is 5 minutes, and each run on a private repo is billed as at least 1 minute. | Rely on the dispatch trigger and keep the schedule as a slow fallback. |
