# Lab 3 — Secure Git: Signed Commits, Secret Scanning, and History Hygiene

![difficulty](https://img.shields.io/badge/difficulty-beginner-success)
![topic](https://img.shields.io/badge/topic-Secure%20Git-blue)
![points](https://img.shields.io/badge/points-10%2B2-orange)
![tech](https://img.shields.io/badge/tech-Git%20%2B%20gitleaks%20%2B%20filter--repo-informational)

> **Goal:** Configure SSH commit signing, wire gitleaks into a pre-commit hook, and (bonus) rewrite history to purge a planted secret.
> **Deliverable:** A PR from `feature/lab3` with `submissions/lab3.md` plus `.pre-commit-config.yaml`. Submit PR link via Moodle.

---

## Overview

In this lab you will practice:
- **SSH commit signing** (Git ≥ 2.34, GitHub verification since Aug 2022) — the STRIDE-R control from Lecture 2
- **Pre-commit hooks** + **gitleaks** — catching secrets before they leave your laptop
- **History rewriting with `git filter-repo`** — the cleanup tool when prevention failed

> Don't skim this lab. The 5-year Toyota leak (Lecture 3, slide 1) was prevented by exactly the controls you'll wire up today.

---

## Project State

**You should have from Labs 1-2:**
- A fork of the course repo + working `feature/labN` PR workflow
- The PR template auto-filling
- A threat model surfacing STRIDE-R (Repudiation) as one of the risks Threagile flagged

**This lab adds:**
- Every future commit on your fork will be cryptographically signed and show "Verified" on GitHub
- A pre-commit hook that blocks accidental secret commits
- (Bonus) Practical experience rewriting history with `git filter-repo`

---

## Setup

You need:
- **Git ≥ 2.34** (`git --version` — needed for native SSH signing)
- **An SSH key** (`ls ~/.ssh/id_*.pub` — generate with `ssh-keygen -t ed25519` if missing)
- **Python 3.10+** + **pip** (for the `pre-commit` framework + `git-filter-repo`)
- **`gitleaks`** CLI — `brew install gitleaks` or download from [GitHub releases](https://github.com/gitleaks/gitleaks/releases). Course pins **gitleaks v8.x**.

```bash
# Branch off main
git switch main && git pull
git switch -c feature/lab3

# Install pre-commit (Python tool, works for any repo type)
pip install pre-commit          # or: pipx install pre-commit
pip install git-filter-repo     # for the bonus task
```

---

## Task 1 — SSH Commit Signing (6 pts)

**Objective:** Configure your Git client to sign every commit with your SSH key, upload the public key to GitHub as a **Signing Key**, and demonstrate verification works locally and on GitHub.

### 3.1: Configure local signing

```bash
# Tell Git to use SSH (not GPG) for signing
git config --global gpg.format ssh

# Point to your public key
git config --global user.signingkey ~/.ssh/id_ed25519.pub
# Use id_rsa.pub if you don't have ed25519

# Sign every commit + tag by default
git config --global commit.gpgsign true
git config --global tag.gpgsign true

# Set up local verification (so `git log --show-signature` works offline)
mkdir -p ~/.config/git
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers

# Add yourself to the allowed signers file
echo "$(git config --global user.email) namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" \
  >> ~/.config/git/allowed_signers
```

### 3.2: Upload the public key to GitHub as a Signing Key

1. Go to **GitHub → Settings → SSH and GPG keys → New SSH key**
2. **Key type: Signing Key** (this is the trap — if you only have it under "Authentication Key", commits show as Unverified)
3. Paste `cat ~/.ssh/id_ed25519.pub` output

> **The same key bytes can be uploaded under both roles** (Authentication + Signing). If you already use the key for `git push`, you still need to add it again under Signing.

### 3.3: Test signing

```bash
# Make a signed commit
echo "lab3 signing test" > submissions/lab3.md
git add submissions/lab3.md
git commit -m "test: first signed commit"

# Verify locally
git log --show-signature -1
# Should see: "Good \"git\" signature for <your-email>"
```

Push the branch and verify on GitHub:

```bash
git push -u origin feature/lab3
```

Open your fork on GitHub → Commits tab → your commit should show a green **Verified** badge.

### 3.4: Document in `submissions/lab3.md`

```markdown
# Lab 3 — Submission

## Task 1: SSH Commit Signing

### Local configuration
- `git config --global gpg.format` → <output>
- `git config --global user.signingkey` → <output>
- `git config --global commit.gpgsign` → <output>

### Local verification
Output of `git log --show-signature -1`:
```
<paste — should include "Good \"git\" signature for <email>">
```

### GitHub verification
- Direct link to your most recent commit on GitHub: <URL>
- Screenshot of the Verified badge: <inline image OR link to image file in PR>

### One-paragraph reflection (2-3 sentences)
What STRIDE-R (Repudiation) scenario would a forged-author commit enable in a real
team's codebase? How does the Verified badge make that attack visible?
```

---

## Task 2 — Pre-commit + gitleaks (4 pts)

> ⏭️ Optional. Skipping won't affect future labs. But this is the control that prevented Toyota's 5-year T-Connect key leak — worth getting right.

**Objective:** Install the `pre-commit` framework, wire `gitleaks` into it, prove it catches a planted secret, and document the tune-out workflow.

### 3.5: Create the pre-commit config

```bash
# YOUR TASK: create .pre-commit-config.yaml at repo root
```

Required content (see [pre-commit docs](https://pre-commit.com/) for syntax):
- Repo: `https://github.com/gitleaks/gitleaks`
- Rev: a real **v8.x** release tag (check [gitleaks releases](https://github.com/gitleaks/gitleaks/releases) for the latest)
- Hook id: `gitleaks`
- At least one additional hook from `pre-commit/pre-commit-hooks` — recommended: `detect-private-key` and `check-added-large-files`

### 3.6: Install the hook in your local repo

```bash
pre-commit install
# Should print: "pre-commit installed at .git/hooks/pre-commit"

# Sanity check: run all hooks on every file once
pre-commit run --all-files
```

### 3.7: Plant a fake secret and observe the block

```bash
# Create a file with a fake-but-realistic GH PAT
# (AKIAIOSFODNN7EXAMPLE / wJalr... are gitleaks-allowlisted as canonical example values;
#  using a GitHub-PAT-style string actually triggers the gitleaks v8 detector)
cat > /tmp/leak-test.txt <<EOF
# This is a deliberate fake secret for Lab 3 testing
GH_PAT=ghp_16C7e42F292c6912E7710c838347Ae178B4a
EOF
cp /tmp/leak-test.txt submissions/leak-attempt.txt
git add submissions/leak-attempt.txt
git commit -m "test: should be blocked by gitleaks"
# Should ABORT with gitleaks reporting the AWS-key pattern
```

Then **unstage** the test file — don't commit the planted secret even with `--no-verify`:

```bash
git restore --staged submissions/leak-attempt.txt
rm submissions/leak-attempt.txt /tmp/leak-test.txt
```

### 3.8: Document in `submissions/lab3.md`

```markdown
## Task 2: Pre-commit + gitleaks

### `.pre-commit-config.yaml` (paste the full content)
```
<paste your YAML>
```

### `pre-commit install` output
```
<paste — should say "pre-commit installed at .git/hooks/pre-commit">
```

### The blocked commit
Output of the `git commit` that gitleaks blocked (the failing hook output):
```
<paste the error block — gitleaks usually shows the rule ID + redacted finding>
```

### Tune-out exercise
Suppose a teammate insists they need to commit `AKIA*` strings because they're documentation examples in `docs/`. Briefly describe two approaches:
1. **Inline allowlist** — `[allowlist]` block in `.gitleaks.toml`. When is this OK?
2. **Path exclusion** — `paths: [docs/]` in `.gitleaks.toml`. When is this risky?
(2-3 sentences each. No correct answer; both have tradeoffs.)
```

---

## Bonus Task — History Rewrite with `git filter-repo` (2 pts)

> 🌟 **Practical & genuinely tricky.** This is the tool you'll use the day a real secret slips past your hooks. Doing it once on a sandbox repo means you'll know the workflow when it counts.

**Objective:** Plant a fake secret in a fresh sandbox repo, push it, then **rewrite history** with `git filter-repo` to purge the secret across all commits and force-push. Document the gotchas.

### B.1: Set up the sandbox

```bash
# Work outside your course fork — this is a throwaway repo
cd /tmp
mkdir lab3-bonus && cd lab3-bonus
git init
git commit --allow-empty -m "init"

# Plant a secret across three commits to make rewriting non-trivial
echo "API_KEY=ghp_AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJ" > config.txt
git add config.txt && git commit -m "feat: add config"

echo "log file" > app.log
git add app.log && git commit -m "feat: empty log"

echo "API_KEY=ghp_AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJ" >> README.md
git add README.md && git commit -m "docs: add usage notes"

# Verify the secret is in history
git log -p | grep -c 'ghp_AAAA'
# Should print: 2 (two commits contain the secret)
```

### B.2: Rewrite history

```bash
# Use a replacement file to swap the secret everywhere
echo 'ghp_AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJ==>[REDACTED]' > /tmp/replace.txt

# YOUR TASK: run git filter-repo --replace-text /tmp/replace.txt
# Hint: filter-repo refuses to run if there's an existing remote (this is a sandbox so no issue)

# Verify the rewrite
git log -p | grep -c 'ghp_AAAA'
# Should now print: 0

git log -p | grep -c 'REDACTED'
# Should print: 2 (the secret is gone; the marker remains)
```

### B.3: Document in `submissions/lab3.md`

```markdown
## Bonus: History Rewrite

### Before
```
<paste output of: git log --oneline before rewrite>
```
Output of `git log -p | grep -c 'ghp_'`: **2**

### After
```
<paste output of: git log --oneline after rewrite>
```
Output of `git log -p | grep -c 'ghp_'`: **0**
Output of `git log -p | grep -c 'REDACTED'`: **2**

### The two-step pattern in real life
1. `git filter-repo --replace-text replacements.txt` — rewrite locally
2. **<answer here>** — what's the MANDATORY second step in a real incident?
   (Hint: Lecture 3 slide 12 has this — it's the difference between cleanup and remediation.)

### Two real-world gotchas you discovered (2 sentences each)
1. <something the lab actually surprised you with — e.g., "filter-repo refused to run because there were existing remotes; I had to remove origin">
2. <another>
```

> ⚠️ **Do NOT commit `/tmp/lab3-bonus`** to your course fork. The bonus is on a sandbox repo; the submission paste-in is the evidence.

---

## How to Submit

```bash
git add .pre-commit-config.yaml          # Task 2
git add submissions/lab3.md
git commit -m "feat(lab3): SSH signing + gitleaks pre-commit + history rewrite practice"
# This commit must be signed — verify with: git log --show-signature -1
git push -u origin feature/lab3
```

PR checklist body:

```text
- [x] Task 1 — SSH signing configured + Verified badge on commit
- [ ] Task 2 — .pre-commit-config.yaml + gitleaks demonstrably blocking
- [ ] Bonus — filter-repo rewrite practice documented
```

---

## Acceptance Criteria

### Task 1 (6 pts)
- ✅ `git config --global gpg.format` returns `ssh`
- ✅ `git log --show-signature -1` shows "Good \"git\" signature"
- ✅ On the submitted PR, **every commit by the student** shows the green Verified badge on GitHub
- ✅ Submission includes screenshot/link of Verified badge + STRIDE-R reflection (2-3 sentences, substantive)

### Task 2 (4 pts)
- ✅ `.pre-commit-config.yaml` exists with gitleaks (v8.x tag) + at least one other hook
- ✅ Submission includes actual gitleaks-blocked commit output (rule ID + redacted finding visible)
- ✅ Tune-out exercise answered for both inline allowlist AND path exclusion

### Bonus Task (2 pts)
- ✅ Submission shows before/after `git log -p | grep -c` outputs (2 → 0)
- ✅ The MANDATORY second step is correctly identified (it's **rotation**, not just rewrite)
- ✅ Two genuine gotchas the student hit during their dry-run (not generic — must be specific)

---

## Rubric

| Task | Points | Criteria |
|------|-------:|----------|
| **Task 1** — SSH signing | **6** | Local + GitHub verification both working + STRIDE-R reflection |
| **Task 2** — Pre-commit + gitleaks | **4** | Config + install + blocked-commit evidence + tune-out tradeoffs |
| **Bonus Task** — History rewrite | **2** | Before/after greps + correct mandatory-second-step + 2 specific gotchas |
| **Total** | **12** | 10 main + 2 bonus |

---

## Resources

<details>
<summary>📚 Documentation</summary>

- [Pro Git, Chapter 7: Signing Your Work](https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work) — Free book chapter
- [GitHub: About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification) — Official docs
- [pre-commit framework](https://pre-commit.com/) — Tool homepage
- [gitleaks docs](https://github.com/gitleaks/gitleaks/blob/master/README.md) — Including config syntax
- [git-filter-repo manual](https://htmlpreview.github.io/?https://github.com/newren/git-filter-repo/blob/docs/html/git-filter-repo.html)

</details>

<details>
<summary>⚠️ Common Pitfalls</summary>

- 🚨 **Commits still show "Unverified" on GitHub** — most likely cause: you uploaded the SSH key under **Authentication** but not **Signing**. Add it again under the Signing Key role. The same key bytes, two slots.
- 🚨 **"error: gpg failed to sign the data"** — Git < 2.34 doesn't support SSH signing natively. Upgrade or use GPG instead (slightly more setup; same end result).
- 🚨 **`pre-commit run --all-files` exits 1 on files you didn't touch** — pre-commit checks the WHOLE repo on first run. Either fix the legacy issues OR scope by adding `default_stages: [commit]` and only running on staged files.
- 🚨 **gitleaks doesn't catch your planted secret** — gitleaks v8 changed how `allowlist` blocks work, and the canonical `AKIAIOSFODNN7EXAMPLE` value is in the built-in allowlist (gitleaks treats it as documentation). Use the `ghp_16C7e42F292c6912E7710c838347Ae178B4a`-style fake PAT from the lab — that triggers the `github-pat` rule.
- 🚨 **`gitleaks protect --staged` returns "command not found"** — gitleaks v8.21+ dropped the `protect` subcommand. Use `gitleaks dir --staged` for the same behavior (the pre-commit framework's hook config handles this for you).
- 🚨 **`git filter-repo: error: this does not look like a fresh clone`** — by design. Run `git filter-repo --force` only if you understand the implications, or work on a fresh clone (the lab's bonus uses a fresh `git init` sandbox, so no issue).
- 💡 **`git log --show-signature` works offline thanks to the `allowedSignersFile` config** in step 3.1. Verify the file path is correct: `cat ~/.config/git/allowed_signers` should print your email + key.

</details>

<details>
<summary>🪜 Looking ahead</summary>

What you set up here propagates:
- **Lab 4** (CI/CD) — your CI workflow will require signed commits via branch protection
- **Lab 8** (Supply Chain) — the same SSH key signs your Cosign attestations
- **Lab 10** (DefectDojo) — secret-leak findings from gitleaks get imported alongside other scanners' findings

Branch protection rule on `main`: "Require signed commits". Enable it in your fork's settings (Settings → Branches → Branch protection rules). Your future self will thank you when an unsigned commit gets rejected at push time.

</details>
