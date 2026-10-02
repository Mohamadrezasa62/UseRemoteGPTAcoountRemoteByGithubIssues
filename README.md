# UseRemoteGPTAcoountRemoteByGithubIssues

Use GitHub Issues as a request queue for Codex running on your always-on Windows PC. The PC signs in to your ChatGPT Pro account once. From another PC, you use **GitHub only**: create an issue, apply `codex-ready`, then read Codex's answer and pull request on GitHub. The watcher never merges a PR automatically.

This repository publishes the example watcher and guide. It is **public**. Issues, replies, branches, commits, and PRs made here are public too. For private code and requests, copy the watcher into a **private repository** and point it there. Never put Pro credentials, `auth.json`, API keys, logs, or private code in GitHub.

OpenAI's built-in `@codex` mention on GitHub Issues starts a **cloud** task. This watcher is for work that must run on your PC. A GitHub plugin alone does not install a listener on that PC.

## Complete example

You create issue **#12** in `Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues` from your laptop:

> **Title:** Add a health check command
> **Body:** Add a `-HealthCheck` option that verifies `gh`, `git`, and `codex` are available. Do not run an issue. Show a useful error if a command is missing. Update the README and run the PowerShell parser check.

You add `codex-ready`. Within five minutes, the always-on PC checks that the author is `Mohamadrezasa62`, changes the label to `codex-running`, and creates `codex/issue-12` locally. It runs Codex in a separate worktree. If files change, the wrapper commits and pushes the branch, creates a PR, comments on issue #12 with the answer and PR URL, and changes the label to `codex-done`. You review and merge the PR yourself. The issue and PR numbers are illustrative; this guide does not create them.

## 1. Prepare GitHub

This guide's public repository is `https://github.com/Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues`. For private work, create a private repository such as `Mohamadrezasa62/my-private-codex-project` and substitute its owner/name in all commands. That private name is an example, not an existing repository.

Enable **Issues** and protect `main` so changes require a PR. On a public repository, limit issue labeling to trusted maintainers. The watcher also checks the issue author's exact GitHub login. Give the PC worker a separate GitHub identity with write access only to the selected repository, or another repository-scoped credential. It needs to push branches, create PRs, edit labels, and comment on issues.

Create the labels from a PC where GitHub CLI is authenticated:

```powershell
gh label create codex-ready   --repo Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues --color 1D76DB
gh label create codex-running --repo Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues --color FBCA04
gh label create codex-done    --repo Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues --color 0E8A16
gh label create codex-failed  --repo Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues --color D93F0B
```

For a private repo, replace the `--repo` value in all four commands.

## 2. Prepare the always-on Windows PC

Create a dedicated **standard Windows user** for this worker. It should not have access to personal documents or unrelated repositories. Install Git, GitHub CLI (`gh`), and Codex CLI for that user. In PowerShell, verify them:

```powershell
git --version
gh --version
codex --version
```

Sign in to GitHub as the worker identity, clone the selected repository, and set the commit identity. For this public example:

```powershell
gh auth login
gh repo clone Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues C:\CodexWorker\repo
git -C C:\CodexWorker\repo config user.name "Codex Issue Worker"
git -C C:\CodexWorker\repo config user.email "worker@example.com"
gh auth status
```

Replace `worker@example.com` with an email associated with the worker's GitHub identity. For a private project, clone that private repository to the same path. The watcher uses `gh` directly; no GitHub plugin is needed on the host.

Create an isolated Codex home and sign in to Pro **on this PC only**:

```powershell
$env:CODEX_HOME = 'C:\CodexWorker\codex-home'
New-Item -ItemType Directory -Force $env:CODEX_HOME | Out-Null
Set-Content -Path (Join-Path $env:CODEX_HOME 'config.toml') -Value "[features]`nmemories = false`n"
Remove-Item Env:OPENAI_API_KEY,Env:CODEX_API_KEY -ErrorAction SilentlyContinue
codex login
codex login status
```

The status should show ChatGPT authentication. Never copy `C:\CodexWorker\codex-home` or its `auth.json` to GitHub or another PC. The watcher also passes `--ignore-user-config`, `--ephemeral`, and `features.memories=false` on every run. Keep the PC awake and online. It only needs outbound connections to GitHub and OpenAI; no same-LAN connection or inbound public port is needed.

**Privacy example:** If your usual Windows account has `C:\Users\You\Documents\PrivateProject`, the worker user should not be able to open that folder. Its checkout is only `C:\CodexWorker\repo`. Codex CLI uses a separate local memory store from ChatGPT web memory, and memory use is disabled here. Windows account permissions and PR review remain essential; a prompt or filename filter alone does not enforce isolation.

## 3. Install and test the watcher

Copy [watch-issues.ps1](watch-issues.ps1) to `C:\CodexWorker\watch-issues.ps1`. As the worker user, run one manual check. This public repository requires explicit `-AllowPublicRepo`:

```powershell
powershell.exe -NoProfile -File C:\CodexWorker\watch-issues.ps1 `
  -Repo Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues `
  -Checkout C:\CodexWorker\repo `
  -AllowedAuthor Mohamadrezasa62 `
  -CodexHome C:\CodexWorker\codex-home `
  -BaseBranch main `
  -AllowPublicRepo
```

The watcher refuses public repositories without `-AllowPublicRepo`. For private work, substitute the private `-Repo` and omit that switch. With no `codex-ready` issues, the command exits without running Codex. It checks that the checkout's `origin` matches `-Repo`.

## 4. Schedule the PC worker

In **Windows Task Scheduler**, select **Create Task**:

1. **General:** choose the dedicated Windows worker user. Leave **Run with highest privileges** off. Try **Run whether user is logged on or not**.
2. **Triggers:** add a daily trigger that repeats every **5 minutes** indefinitely.
3. **Actions:** choose `powershell.exe` as the program. For the public example, paste these arguments on one line:

   ```text
   -NoProfile -File C:\CodexWorker\watch-issues.ps1 -Repo Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues -Checkout C:\CodexWorker\repo -AllowedAuthor Mohamadrezasa62 -CodexHome C:\CodexWorker\codex-home -BaseBranch main -AllowPublicRepo
   ```

4. **Settings:** choose **Do not start a new instance** if already running.
5. Save, run manually, and test after a restart. If a logged-out task cannot access GitHub or Codex authentication, use **Run only when user is logged on** and keep that session active.

For a private repository, replace the `-Repo` value and omit `-AllowPublicRepo` in the scheduled arguments.

## 5. Send a task from another PC

Sign in to **GitHub** using `Mohamadrezasa62` on the other PC. No Pro sign-in is needed there. Open a new issue in the selected repository. Here is a copyable example:

```markdown
Title: Add a health check command

Goal:
Add a -HealthCheck option to watch-issues.ps1.

Acceptance criteria:
- Verify gh, git, and codex are available before reading issues.
- Print a clear error naming any missing command.
- Do not process an issue during a health check.
- Update README.md with a usage example.

Verification:
- Run a PowerShell parser check on watch-issues.ps1.
- Run the health check once with all tools installed and once with a missing tool.
```

Add `codex-ready` when ready. The issue author must match `-AllowedAuthor`. On this public example repository, everyone can read the issue and answer; use a private repository for private code or instructions.

**Example result:** For issue #12, the worker creates `codex/issue-12` in `C:\CodexWorker\codex-issue-worktrees\issue-12` and writes logs under `C:\CodexWorker\codex-home\issue-logs`. If Codex changes files, it opens a PR and comments on the issue, for example:

> Codex finished on the always-on PC and opened `https://github.com/Mohamadrezasa62/UseRemoteGPTAcoountRemoteByGithubIssues/pull/13`. Please review the PR before merging.
> Added `-HealthCheck`, updated the guide, and verified the parser check.

The PR URL above is **illustrative**. The PR body contains `Closes #12`, so GitHub closes the issue when you merge the PR.

## 6. Handle failures and follow-ups

If issue #12 fails, the watcher changes its label to `codex-failed` and leaves a short comment. On the host, inspect `C:\CodexWorker\codex-home\issue-logs\issue-12.log` and run `git -C C:\CodexWorker\repo worktree list`. A partial branch or worktree may remain for inspection. The watcher refuses to overwrite it.

For a new request, create a **new issue**. To retry the same issue, inspect and clean up its partial worktree and branch on the host, remove `codex-failed`, then apply `codex-ready` again. Do not blindly delete worktrees or branches that contain changes you want to keep.

## 7. Boundaries and verification

- This setup is for **your own GitHub identity across your computers**. It is not a way to provide other people access to your Pro subscription.
- Codex receives the issue text and repository files. Treat public issue content as untrusted. The watcher only runs issues authored by the configured login and requires explicit public-repo opt-in.
- The worker uses `workspace-write` and never merges. Review diffs and CI before merging.
- The wrapper refuses to commit common secret-file names, but cannot detect every secret. Never put credentials in repositories or issues.
- The PowerShell script has been syntax checked. A live GitHub-to-PR run still requires your GitHub and Codex logins on the host and a real issue; those steps were not performed while preparing this public example.

## References

- [Codex non-interactive mode and saved ChatGPT authentication](https://learn.chatgpt.com/docs/non-interactive-mode)
- [Codex CLI login and command reference](https://learn.chatgpt.com/docs/developer-commands)
- [Codex local memories](https://learn.chatgpt.com/docs/customization/memories)
- [GitHub Issues CLI](https://cli.github.com/manual/gh_issue_list) and [PR creation](https://cli.github.com/manual/gh_pr_create)
- [OpenAI changelog: `@codex` on GitHub Issues](https://learn.chatgpt.com/docs/changelog)
