---
author: ["Sanyam Jain"]
title: "Pushing to Bitbucket from Claude Code Cloud Sessions"
date: "2026-11-03"
summary: "Claude Code's cloud sessions are built around GitHub. Here's how I got one to clone, commit and push to a Bitbucket Cloud repo using a repository access token and a network secret."
description: "A step-by-step guide to letting Claude Code cloud sessions push to Bitbucket Cloud over HTTPS, without the token ever entering the session VM."
tags: ["Claude Code", "Bitbucket", "Git", "AI Agents"]
ShowToc: true
TocOpen: true
draft: true
---

Claude Code's cloud sessions (claude.ai/code, the Code tab in the Claude app, or `claude --cloud` from a terminal) are great for handing off a task and checking on it from your phone. The catch is that they're built around GitHub. Most of my client work lives on Bitbucket.

Anthropic's docs are clear about the official position: cloning and PR creation require GitHub. You can upload a Bitbucket repo to a session as a local bundle, but the session can't push back to that remote.

I wanted Claude to push a real branch to Bitbucket. It works, with a few gotchas.

## TL;DR

- Create a Bitbucket **repository access token** with Repositories: Read and Write.
- Add it to your Claude cloud environment as a **network secret**: host `bitbucket.org`, type username and password, username `x-token-auth`, password = the token.
- In the session, clone over **HTTPS with no credentials in the URL**. The proxy adds auth on the way out.
- SSH won't work, and setup scripts don't get the secret.

## Why it doesn't just work

Every cloud session runs in an isolated VM, and all outbound traffic goes through an HTTP/HTTPS security proxy. GitHub gets special treatment: a dedicated GitHub proxy holds your real credentials and swaps them in for a scoped credential the VM uses. Nothing like that exists for Bitbucket.

Two things make Bitbucket possible anyway:

1. `bitbucket.org` and `api.bitbucket.org` are on the default **Trusted** network allowlist, so the session can reach Bitbucket over HTTPS out of the box.
2. Cloud environments support **network secrets**.

## Network secrets in one paragraph

A network secret is a token you store on a cloud environment. Anthropic's agent proxy attaches it to outgoing requests for the hosts you list, *after* the request leaves the VM. The token never appears in the session: not in environment variables, not on disk. Claude can use it but can't read it.

The alternative is an environment variable, which anyone who uses the environment can read, including the agent. For an agent with a shell, that difference matters. A prompt injection hidden in some README can't leak a token the agent never had.

## Step 1: Create a repository access token

In Bitbucket, go to **Repository settings → Security → Access tokens → Create access token**.

- **Scopes:** Repositories → Read and Write. Add Pull requests → Write if you plan to open PRs through the API later.
- Copy the token. You won't see it again.
- Also note the token's bot email, `{botid}@bots.bitbucket.org`. The commits will use it as the author email.

Why a repository access token rather than a personal API token?

- **Blast radius:** it only reaches one repository.
- **Attribution:** Bitbucket shows the access token as its own "user", named after the token. Every push from the agent is labelled as the agent's, not mine.

If you'd rather use a personal API token, that works too:

- Username: your Bitbucket username (case-sensitive) or the static `x-bitbucket-api-token-auth`.
- Scopes: `read:repository:bitbucket` and `write:repository:bitbucket`.
- Don't use app passwords. Atlassian deprecated them in 2025.

## Step 2: Add the network secret

On [claude.ai/code](https://claude.ai/code), open the environment selector under the message box. Hover over your environment (or create a new one), click the settings icon, then open **Network secrets → Add secret**.

| Field | Value |
| --- | --- |
| Name | `Bitbucket – <repo>` |
| Host / allowed websites | `bitbucket.org` |
| Secret type | Username and password |
| Username | `x-token-auth` |
| Password | your repository access token |

Save it. You can't view the value again. To rotate it, delete the secret and add it again.

A few notes:

- **Network access level:** leave it on **Trusted**. Hosts listed on a network secret are reachable regardless of the allowlist anyway.
- **Who can add secrets:** on Pro, Max and Team you can add them to your own environments. On Enterprise you need an organization admin role. Team members without that role get a shorter form that takes a single host with no wildcards, which is all this needs.

## Step 3: Clone, commit and push from the session

Start a cloud session in that environment. Your Bitbucket repo won't be attached the way a GitHub repo would be. Claude clones it itself:

```bash
# Fail fast instead of hanging on a password prompt if auth isn't injected
export GIT_TERMINAL_PROMPT=0

# Sanity check: can we see the repo at all?
git ls-remote https://bitbucket.org/<workspace>/<repo>.git

git clone https://bitbucket.org/<workspace>/<repo>.git
cd <repo>

git config user.name  "Claude Code (cloud)"
git config user.email "<botid>@bots.bitbucket.org"

git checkout -b test-claude-code
echo "Claude in Cloud Works" >> README.md
git commit -am "test: verify push from Claude cloud session"
git push -u origin test-claude-code
```

Notice there's no username or token in any URL. The proxy sees a request to `bitbucket.org` and adds the Basic auth header itself.

## The test prompt I used

I didn't run those commands myself. I gave Claude a test with a clear pass/fail, so a failure would show up as an error report rather than Claude quietly improvising:

```text
Goal: verify that this cloud session can push to a Bitbucket Cloud repo.
This is a connectivity test, not real work.

Context:
- Repo: https://bitbucket.org/<workspace>/<repo>.git
- This environment has a network secret for bitbucket.org. The agent proxy
  injects Basic auth into HTTPS requests to bitbucket.org after they leave
  the VM. You will NOT see the token and shouldn't need it.
- Do NOT put credentials or a username in any URL, do NOT configure a
  credential helper, and do NOT use SSH (HTTPS only).

Steps:
1. export GIT_TERMINAL_PROMPT=0
2. git ls-remote <repo URL>. If this fails, stop and report.
3. Clone, then set user.name and user.email (<botid>@bots.bitbucket.org).
4. git checkout -b test-claude-code
5. Append the line "Claude in Cloud Works" to README.md. Change nothing else;
   confirm with git diff --stat.
6. Commit, then git push -u origin test-claude-code.
7. Verify with git ls-remote --heads origin test-claude-code.

Report PASS/FAIL, the exact push and verify output, and the commit hash.
If anything fails, report the exact error and step. Don't try other auth
methods. Never print or commit any token.
```

It came back PASS, and the branch showed up in Bitbucket with the README change.

## Gotchas

**Strip the username from Bitbucket's clone URL.** The Clone button gives you `https://yourname@bitbucket.org/...`. With a username in the URL, git may try to handle auth itself instead of leaving it to the proxy. Use the bare `https://bitbucket.org/<workspace>/<repo>.git`.

**SSH won't work.** The session's proxy only handles HTTP and HTTPS, so `git@bitbucket.org:...` remotes are out. Stick to HTTPS.

**Setup scripts don't get the secret.** The agent proxy starts after the environment's setup script runs, so you can't clone a private Bitbucket repo in the setup script. Clone at the start of the session instead.

**GitHub-only features stay GitHub-only.** The session's diff view, "create PR" flow and auto-fix all assume GitHub. Open the PR in Bitbucket yourself. Claude will push the branch, but the review loop happens in Bitbucket.

**Clean up test branches.** If you're testing against a shared repo, delete `test-claude-code` afterwards so it doesn't confuse your teammates.

## Fallback: an environment variable

If you can't add network secrets (an Enterprise account without the admin role, for example), put the token in an environment variable such as `BB_TOKEN` and use it in the remote URL:

```bash
git remote set-url origin "https://x-token-auth:${BB_TOKEN}@bitbucket.org/<workspace>/<repo>.git"
```

It works, but the token is now inside the VM: readable by the agent and by anyone who uses that environment. If you go this way, scope the token to one repo and give it a short expiry.

## Going further

**Opening PRs from the session.** The Bitbucket REST API lives on `api.bitbucket.org`, a separate host from `bitbucket.org`. A second network secret for that host should let Claude open PRs with `curl`. Access tokens authenticate to the API with a `Bearer` header, so that secret would be a Bearer type, not username and password. I haven't tried this yet.

**Teach every session the convention.** The setup script can write a personal `~/.claude/CLAUDE.md`, which Claude Code loads as user instructions. That way you don't have to repeat the rules in every prompt:

```bash
#!/bin/bash
mkdir -p ~/.claude
cat > ~/.claude/CLAUDE.md <<'EOF'
Bitbucket repos: clone over HTTPS with no credentials in the URL,
e.g. https://bitbucket.org/<workspace>/<repo>.git. Auth is injected by the
network proxy. Never use SSH. Run `export GIT_TERMINAL_PROMPT=0` before git
commands so auth failures surface immediately.
EOF
```

## References

- [Use Claude Code in the cloud](https://code.claude.com/docs/en/claude-code-on-the-web): limitations for non-GitHub repos
- [Configure cloud environments](https://code.claude.com/docs/en/cloud-environments): network secrets, access levels, the default allowlist
- [Using access tokens for a repository](https://support.atlassian.com/bitbucket-cloud/docs/using-access-tokens) (Atlassian)
- [Using API tokens](https://support.atlassian.com/bitbucket-cloud/docs/using-api-tokens/) (Atlassian)
