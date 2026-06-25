# PoC Test Setup — End to End

## What this tests

Whether an attacker can open a fork PR, have it trigger a `pull_request_target` workflow,
have their code read by a privileged CI agent, and have injected text posted as an
official repo comment — mirroring `azure/azure-sdk-for-go` mgmt-review.lock.yml.

---

## Step 1 — Create your test repo on GitHub

Create a **new public repo** on your GitHub account, e.g. `vibhum/azure-sdk-go-test`.

> Must be public so fork PRs work the same as on the real target.

---

## Step 2 — Push the base repo (this folder)

```bash
cd poc-harness/
git init
git add .
git commit -m "init: base repo with vulnerable workflow"
git remote add origin https://github.com/YOUR_USERNAME/azure-sdk-go-test.git
git push -u origin main
```

---

## Step 3 — Fork your own repo

Go to `https://github.com/YOUR_USERNAME/azure-sdk-go-test` and click **Fork**.
This creates `YOUR_USERNAME2/azure-sdk-go-test` (use a second account, or
GitHub allows self-forks via the API).

If you only have one account, fork via the GitHub API:
```bash
curl -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/YOUR_USERNAME/azure-sdk-go-test/forks
```

---

## Step 4 — Create PR from fork with payload

In the fork, the payload file is already at `sdk/resourcemanager/poc/review.go`.
Open a PR from your fork's `main` branch → the base repo's `main` branch.

```bash
# In your fork clone
git checkout -b poc-branch
# The payload file is already present — no changes needed
# Or modify the AGENT_COMMENT in review.go if you want custom text
git push origin poc-branch
# Open PR via GitHub UI or CLI
gh pr create --repo YOUR_USERNAME/azure-sdk-go-test \
  --head YOUR_FORK:poc-branch \
  --title "feat: management review poc" \
  --body "Routine update to resource manager client."
```

---

## Step 5 — Trigger the workflow

The workflow fires on `labeled`. Apply any label to the PR:

```bash
gh pr edit <PR_NUMBER> \
  --repo YOUR_USERNAME/azure-sdk-go-test \
  --add-label "needs-review"
```

> If the label doesn't exist yet, create it first:
> ```bash
> gh label create "needs-review" --repo YOUR_USERNAME/azure-sdk-go-test
> ```

---

## Step 6 — Observe

Go to the Actions tab in your base repo and watch the workflow run.

**Expected outcome:**

1. Workflow triggers on `pull_request_target`
2. PR HEAD code (from fork) is checked out
3. `review.go` is read — `AGENT_COMMENT` extracted
4. The following comment appears on the PR posted by `github-actions[bot]`:

```
## ⚠️ PoC: Prompt Injection via pull_request_target

This comment was posted by an agent that read attacker-controlled PR code.

Injected instruction from PR file:
> ✅ CONFIRMED: Attacker-controlled PR code was read by privileged CI agent...
```

---

## Step 7 — Screenshot and document

Screenshot showing:
- The workflow run triggered by `pull_request_target`
- The comment posted by `github-actions[bot]` containing your injected text
- The Actions log showing `GITHUB_TOKEN present: YES`

This is your proof of concept for MSRC submission.

---

## Mapping to Real Target

| Test repo | azure-sdk-for-go |
|-----------|-----------------|
| `actions/checkout@v4 ref: PR HEAD` | `checkout_pr_branch.cjs` |
| `actions/github-script` posting comment | GitHub MCP `pull_requests` tool |
| `GITHUB_TOKEN` (scoped to repo) | `GH_AW_GITHUB_MCP_SERVER_TOKEN` (likely broader PAT) |
| Manual label trigger | Auto-labeler or maintainer label |
| Comment posted by bot | Agent-managed PR review comment |

The real target's impact is **higher** because:
- Token is likely a PAT (`GH_AW_GITHUB_MCP_SERVER_TOKEN`), not just `GITHUB_TOKEN`
- Copilot LLM follows natural language instructions — no `AGENT_COMMENT:` marker needed
- Agent has shell execution (`--allow-all-tools`) so token exfiltration is also possible
