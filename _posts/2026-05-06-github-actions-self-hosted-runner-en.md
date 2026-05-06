---
title: "Free CI Minutes Are Gone: Setting Up a GitHub Actions Self-Hosted Runner on Linux"
description: "When your monthly 3,000 GitHub Actions minutes run out mid-project, you have two options: upgrade your plan or turn the server you already have into a runner. I did the second one."
date: 2026-05-06 12:00:00 +0300
categories: [DevOps, GitHub Actions, CI/CD]
tags: [github-actions, self-hosted-runner, linux, systemd, ci-cd]
image:
  path: /assets/img/posts/2026-05-06-github-actions-self-hosted-runner/banner.png
  alt: "GitHub Actions Self-Hosted Runner Setup"
---

GitHub gives you 3,000 free Actions minutes per month on private repositories.

That sounds like a lot until you're running multi-step CI pipelines on every PR, every push, every little tweak. Then it's not a lot. Then you're watching the counter drop and quietly calculating how many commits you can still make this month.

The obvious fix is paying more. The other fix is already having a server sitting there.

I had a server sitting there. I set it up as a self-hosted runner. Here's what that looked like.

---

## Why Not Just Pay?

I don't have a strong principled reason. The server was running anyway, jobs were queuing, and the quota was at zero. The path of least resistance was pointing GitHub at the machine I already had.

Self-hosted runners also run on your hardware, in your network, with your dependencies already installed — which means no time spent on "install Python 3.12, install dependencies, wait for cache" every single run. The first run is slower (Poetry installs everything fresh); subsequent runs are faster because the virtualenv is already there.

---

## The Setup

### 1. Create a Dedicated User

Don't run the runner as root. Create a system user for it:

```bash
useradd -r -m -d /opt/github-runner -s /bin/bash \
  -c "GitHub Actions Runner" github-runner
```

`-r` makes it a system account (UID < 1000), `-m` creates the home directory, `-d` sets it to `/opt/github-runner`. No sudo access, no shell login by default. Exactly what you want.

### 2. Download and Verify the Runner

Find the latest version at [github.com/actions/runner/releases](https://github.com/actions/runner/releases), then:

```bash
cd /opt/github-runner

curl -o actions-runner-linux-x64-2.334.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.334.0/actions-runner-linux-x64-2.334.0.tar.gz

# Verify SHA256 (hash is in the release body)
echo "048024cd2c848eb6f14d5646d56c13a4def2ae7ee3ad12122bee960c56f3d271  actions-runner-linux-x64-2.334.0.tar.gz" | sha256sum -c

tar xzf actions-runner-linux-x64-2.334.0.tar.gz
```

SHA256 verification matters here. You're downloading an executable that will have significant access to your server.

### 3. Get a Registration Token

Go to your repository → Settings → Actions → Runners → New self-hosted runner. GitHub will show you a registration token. Copy it — it expires in about an hour.

### 4. Register the Runner

```bash
sudo -u github-runner ./config.sh \
  --url https://github.com/your-username/your-repo \
  --token YOUR_REGISTRATION_TOKEN \
  --name prod-server-01 \
  --labels "self-hosted,linux,prod" \
  --unattended
```

Run this as the `github-runner` user, not root. The `--labels` let you target this specific runner in your workflow YAML. `--unattended` skips the interactive prompts.

When it works, you'll see:

```
√ Connected to GitHub
√ Runner successfully added
√ Settings Saved
```

### 5. Install as a systemd Service

```bash
cd /opt/github-runner
sudo ./svc.sh install github-runner
sudo ./svc.sh start
```

Check it's running:

```bash
sudo systemctl status actions.runner.*.service
```

You want `active (running)`. Check the logs:

```bash
sudo journalctl -u actions.runner.*.service -n 50 --no-pager
```

The line you're looking for: `Listening for Jobs`. Once you see that, the runner is up and waiting.

![GitHub Actions Runner Setup](/assets/img/posts/2026-05-06-github-actions-self-hosted-runner/runner-setup.png)
*The runner registration flow — straightforward once you have the token.*

---

## Security Hardening

A few things worth doing before you call it done:

**File permissions.** The runner directory should be owned by the runner user only:

```bash
chown -R github-runner:github-runner /opt/github-runner
chmod 700 /opt/github-runner
```

**Limit what the runner user can do.** Don't add it to sudoers unless your jobs actually need it. If they do, scope the permissions tightly with a specific sudoers rule rather than giving full sudo access.

**Consider ephemeral runners for sensitive repos.** For public repos especially, [ephemeral runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#self-hosted-runner-security) run each job in a fresh environment and auto-deregister. For a private repo on your own hardware, persistent runners are usually fine — just be aware of the trade-offs.

---

## Updating Your Workflow Files

Change `runs-on` in every workflow YAML:

```yaml
# Before
jobs:
  test:
    runs-on: ubuntu-latest

# After
jobs:
  test:
    runs-on: [self-hosted, linux, prod]
```

The labels in `runs-on` must match what you set with `--labels` during registration. If you have multiple runners with different labels (e.g., `prod`, `staging`), you can target them precisely.

---

## The Moment It Worked

I pushed a PR, watched the workflow page, and saw:

```
Running job: Backend Lint (ruff)
```

Not on a GitHub-hosted VM spinning up somewhere in Azure. On the machine I was sitting in front of. The first run took a while — Poetry was installing everything fresh. After that, runs were noticeably faster because the virtualenv persisted between jobs.

---

## Trade-offs at a Glance

| | GitHub-Hosted | Self-Hosted |
|---|---|---|
| **Cost** | Free (up to 3k min/mo) | Your server's electricity bill |
| **Maintenance** | None | Runner updates, OS patches |
| **Speed** | Consistent (plan-dependent) | Faster after first run (local deps) |
| **Isolation** | Fresh VM every run | Shared filesystem between runs |
| **Network** | GitHub's network | Your network (good for private infra) |

Neither is universally better. If you're under the free tier, GitHub-hosted is the right default. Once you've burned through the quota, self-hosted makes sense if you have spare server capacity.

---

## Conclusion

The setup takes about 20 minutes end-to-end: create the user, download the runner, register it, install the service, update the workflow files. The tricky part is the registration token — it expires quickly, so have the `config.sh` command ready before you generate it.

Once it's running, you stop thinking about it. Jobs queue, the runner picks them up, logs stream in. The CI pipeline works exactly the same as before — just on your hardware instead of GitHub's.

The counter is back to zero. I mean, my quota is back to 3,000. The counter is on my server now.

---

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*

---

## Other Posts

- [freqtrade-mcp: Ask Claude to Check Your Trades](/posts/freqtrade-mcp-server-en/)
- [Adding Browser Automation to CLI-Anything: First MCP Backend Pattern](/posts/cli-anything-browser-automation-contribution-en/)
- [awesome-trending-repos: Modern Web Interface for GitHub Trending](/posts/awesome-trending-repos-web-en/)
