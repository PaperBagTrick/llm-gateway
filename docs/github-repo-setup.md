# Setting Up the llm-gateway GitHub Repo

How the `llm-gateway` repo was set up, step by step, with the commands used, what happened, and the problems we ran into.

- **Repo:** `PaperBagTrick/llm-gateway` (private) at https://github.com/PaperBagTrick/llm-gateway
- **Local path:** `~/llm-gateway`
- **Tools:** Git Bash (UCRT64) on Windows, GitHub CLI (`gh`) over HTTPS, VS Code
- **Approach:** one shared repo for the whole team, one folder per role, one branch per feature

## Status

| Step | What | Status |
|---|---|---|
| 1 | Sign in to GitHub CLI | Done |
| 2 | Create and clone the repo | Done |
| 3 | Folder skeleton with placeholder READMEs | Done |
| 4 | `.gitignore` | In progress (first line added, rest to paste in) |
| 5 | `.env.example` | In progress (file created) |
| 6 | `docs/api-contract.md` and root `README.md` | Contract copied, needs two small fixes; root README to write |
| 7 | Commit and push | Not yet done |
| 8 | Invite teammates | Not yet done |
| 9 | Protect `main` | Not yet done |
| 10 | First branches | Not yet done |

## 1. Sign in to GitHub CLI

Running `gh repo create` before signing in fails with:

```
To get started with GitHub CLI, please run:  gh auth login
```

So sign in first:

```bash
gh auth login
```

Answers used at the prompts:

- Where do you use GitHub? **GitHub.com**
- How would you like to authenticate? **Login with a web browser**
- Preferred protocol for Git operations? **HTTPS**
- Authenticate Git with your GitHub credentials? **Yes**

The terminal prints a one-time code and copies it to the clipboard. Press Enter to open `https://github.com/login/device`, paste the code, and click **Authorize**.

**The authorization screen lists an organization (UTDNebula).** That has nothing to do with this project. GitHub lists every organization the account belongs to. The CLI gets no permissions beyond what the account already has. To keep this side project separate from Nebula work, click the organization row and uncheck it before authorizing. This setup works either way.

Verify the login:

```bash
gh auth status
```

Expected output: logged in as `PaperBagTrick`, Git protocol `https`, token scopes `gist`, `read:org`, `repo`, `workflow`.

## 2. Create and clone the repo

From the home directory:

```bash
gh repo create llm-gateway --private --clone
cd llm-gateway
```

Result: `Created repository PaperBagTrick/llm-gateway on github.com`.

Notes:

- `--clone` already clones the repo into a new `llm-gateway` folder in the directory where you ran the command, so there is no separate `git clone` step.
- Don't run the create command a second time. GitHub will say the repository already exists.
- Private is the safe choice while building. It can be made public later, which is also good for a resume, as long as nothing secret was ever committed.
- Check the result with `git remote -v` (should show `origin` pointing to the repo) and `git status` (an empty repo with no commits is normal).

## 3. Create the folder skeleton

Git doesn't track empty folders, so each folder gets a placeholder README:

```bash
mkdir -p gateway guardrails dashboard bench docs
for d in gateway guardrails dashboard bench docs; do echo "# $d" > $d/README.md; done
code .
```

| Folder | Owner |
|---|---|
| `gateway/` | Backend (Go gateway and admin API) |
| `guardrails/` | AI engineer (Python `/check` service) |
| `dashboard/` | Fullstack (React dashboard) |
| `bench/` | AI engineer (benchmark script and safety test set) |
| `docs/` | Shared (API contract) |

In VS Code, `bench` and `docs` show different icons than the other folders. That is only VS Code's file-icon theme. It doesn't affect Git.

## 4. Add `.gitignore`

Git does not create this file for you, and `gh repo create` only adds one if you pass `--gitignore`. Create `.gitignore` in the repo root (not inside a subfolder). If a folder is highlighted in the VS Code explorer when you click New File, click empty space first so the file lands in the root.

```
.env
.env.*
!.env.example
node_modules/
dist/
__pycache__/
*.pyc
.venv/
data/
.DS_Store
.idea/
.vscode/
```

What each line does:

| Line | Purpose |
|---|---|
| `.env` | Keeps your real secrets (admin token, Hugging Face token) out of the repo |
| `.env.*` | Same, for variants like `.env.local` and `.env.production` |
| `!.env.example` | Exception: the `!` means "do track this one", so the placeholder template stays in the repo |
| `node_modules/`, `dist/` | Dashboard dependencies and build output, which are huge and regenerable |
| `__pycache__/`, `*.pyc`, `.venv/` | Python compiled files and virtual environment |
| `data/` | Local database files or benchmark outputs |
| `.DS_Store` | macOS clutter, in case a teammate uses a Mac |
| `.idea/`, `.vscode/` | Editor settings, which differ per person |

**`.env` vs `.env.*`:** `.env` matches only a file named exactly `.env`. In `.env.*` the `*` stands for any text, so it matches `.env.local`, `.env.dev`, and so on. It also matches `.env.example`, which is why the `!.env.example` line must come **after** it. Order matters. If the `!` line came first, `.env.*` would ignore the file again.

**Test that it works:**

```bash
echo test > .env
git status
```

`.env` should not appear in the list. Then clean up with `rm .env`. To check specific names:

```bash
git check-ignore -v .env.local .env.example
```

`.env.local` should be listed as ignored by the `.env.*` rule, and `.env.example` should print nothing.

**Gotchas:**

- Windows can quietly add a `.txt` extension. Make sure the file is named exactly `.gitignore`.
- An ignore rule doesn't remove a file that was already committed. If a `.env` was ever committed, untrack it with `git rm --cached .env`. Nothing had been committed at this point in the setup, so this wasn't needed.

## 5. Add `.env.example`

This template shows teammates which variables exist, without exposing real values:

```
ADMIN_TOKEN=change-me
MONGO_URL=mongodb://mongo:27017/gateway
UPSTREAM_URL=http://model:8000
GUARDRAILS_URL=http://guardrails:8001
HF_TOKEN=your-huggingface-token
```

- The hostnames (`mongo`, `model`, `guardrails`) must match the service names in `docker-compose.yml`, which gets written on Day 1 of the build.
- A real `.env` file is not needed for the GitHub setup. When the gateway runs for the first time, copy `.env.example` to `.env` and fill in real values. Each teammate makes their own, and `.gitignore` keeps it out of Git.
- Don't put real tokens in `.env` until `git status` confirms the file is ignored.

## 6. Add the API contract and a root README

**`docs/api-contract.md`**

Copy the API contract from the project plan doc into this file. It becomes the single source of truth. Changing it requires all three teammates to approve the pull request. Two fixes after pasting:

1. Remove the stray backslash the copy added in the usage heading, so `key\_id` becomes `key_id`.
2. Add a title and intro at the top, since the paste starts at "Conventions":

```markdown
# API contract (v0.1)

Freeze this on day 1. After that, any change needs all three of you to agree. The fullstack engineer builds against mocks generated from this document.
```

The JSON examples with `//` comments (key creation and `/check`) are not valid JSON, because JSON doesn't allow comments. They are fine as documentation. Delete the comment lines before copying them into a parser or a mock file.

**Root `README.md`**

Short for now:

- Project name and a one-paragraph description (a self-hosted LLM gateway with auth, rate limiting, usage logging, and guardrails)
- The folder list with who owns each (see the table in step 3)
- A note that setup instructions come later

## 7. Check, commit, and push

```bash
git status
```

Confirm that you see the five folders, `.gitignore`, `.env.example`, `README.md`, and `docs/api-contract.md`, and that **no `.env`** appears. If `.env` appears in the list, stop and fix the `.gitignore` first. Then:

```bash
git add .
git commit -m "Initial project skeleton"
git push -u origin main
```

Do this push before turning on branch protection in step 9, otherwise the protection will block it.

## 8. Invite teammates

On GitHub: repo, then **Settings**, then **Collaborators**, then **Add people**. Each friend has to accept the email invite before they can clone.

## 9. Protect `main`

Do this after the first push.

1. Open the repo on GitHub and click **Settings**.
2. In the left sidebar, click **Rules**, then **Rulesets**. On some accounts the path is **Branches** instead.
3. Click **New ruleset**, then **New branch ruleset**.
4. Name it `protect-main` and set **Enforcement status** to **Active**.
5. Under **Target branches**, click **Add target** and choose **Include default branch**.
6. Under **Rules**, tick **Require a pull request before merging**. Set **Required approvals** to **0** for this week, so an offline teammate can't block merges. Still open pull requests, since the history and diff are what matter.
7. Leave **Block force pushes** ticked if it is on by default.
8. Click **Create**.

To test it, try pushing a change directly from `main`. It should be rejected. Then use the normal flow:

```bash
git switch -c backend/test-branch
git push -u origin backend/test-branch
```

GitHub shows a banner offering to open a pull request.

**If the option is greyed out:** private repos on a free GitHub plan often can't use branch protection or rulesets. Either agree as a team that nobody pushes straight to `main`, or make the repo public (Settings, General, Danger Zone) after confirming nothing secret was ever committed. Check with `git log --stat` that no `.env` is in the history.

## 10. First branches

Teammates clone the repo and work on branches named after their area:

```bash
git clone https://github.com/PaperBagTrick/llm-gateway.git
cd llm-gateway
git switch -c ai/vllm-setup
```

Use the prefixes `backend/...`, `ai/...`, and `dashboard/...` so it's obvious who owns what.

Working rules:

- Each person works in their own folder, so merge conflicts stay rare.
- `docker-compose.yml`, `.env.example`, and the README are shared files and the likeliest conflict spots. Keep changes to them small and merge them quickly.
- Commit small and often, especially when using AI assistants, so a bad generation is easy to revert.
- Never commit keys, tokens, or `.env`.

## Optional: track the plan as issues

Create one GitHub issue per day per role from the build guides, and put them on a GitHub Project board with Todo, Doing, and Done columns. It takes about 20 minutes and shows at a glance who is falling behind.

## Troubleshooting

| Problem | Fix |
|---|---|
| `gh repo create` says to run `gh auth login` | Sign in first (step 1) |
| "Repository already exists" | It was already created. Don't rerun it; just `cd llm-gateway` |
| `cd llm-gateway` says the folder doesn't exist | The create command hasn't run yet, or you are in a different directory |
| `.env` shows up in `git status` | Check the `.gitignore` line for typos or spaces, and that the file isn't named `.gitignore.txt` |
| `.env.example` is being ignored | Make sure `!.env.example` comes after `.env.*` |
| Branch protection is greyed out | Free-plan private repos can't use it; see step 9 |
| Teammate can't clone | They haven't accepted the collaborator invite email yet |

## Final layout

```
llm-gateway/
  gateway/README.md
  guardrails/README.md
  dashboard/README.md
  bench/README.md
  docs/
    README.md
    api-contract.md
  .gitignore
  .env.example
  README.md
```
