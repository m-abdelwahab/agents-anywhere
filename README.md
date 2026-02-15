# Agents Anywhere

Run [Claude Code](https://docs.anthropic.com/en/docs/claude-code) in a long-running container on Railway. SSH in from any device and start coding. One-click deploy, persistent storage, and pre-installed [Railway skills](https://docs.railway.com/ai/agent-skills).

- Runs 24/7. Reconnect from anywhere and pick up where you left off.
- Repos, settings, and session history are stored on a [volume](https://docs.railway.com/volumes) at `/data` and persist across redeploys.
- SSH sessions auto-attach to a [`tmux`](https://github.com/tmux/tmux) session — if your connection drops, reconnect and pick up where you left off. To keep processes running, close your terminal or press `Ctrl+B, D` to detach. Typing `exit` kills the session.

---

## Quick Start

### 1. Deploy

You can deploy this project on Railway in a few clicks using a [Railway template](https://docs.railway.com/reference/templates). To get started, click the button below.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/agents-anywhere?referralCode=thisismahmoud&utm_medium=integration&utm_source=template&utm_campaign=generic)

When configuring the template, you'll find the following [environment variables](https://docs.railway.com/variables):

| Variable | Default | Description |
|---|---|---|
| `SSH_PASSWORD` | Auto-generated | Password for SSH access. Generated on deploy, but you can set your own. The SSH port is exposed to the internet — use a strong password (16+ characters) or key-based auth. |
| `SSH_PUBLIC_KEY` | — | Your SSH public key for key-based auth. Use `SSH_PUBLIC_KEY_LAPTOP`, `SSH_PUBLIC_KEY_PHONE`, etc. to register multiple devices. |

---

### 2. Connect

Find your SSH connection details in the Railway dashboard under **Settings → Networking → [TCP Proxy](https://docs.railway.com/networking/tcp-proxy)**. You'll see a hostname and port like `roundhouse.proxy.rlwy.net:12345`.

> **Important:** Use the Railway-assigned port (e.g. `12345`), **not** port 22. Railway's [TCP proxy](https://docs.railway.com/networking/tcp-proxy) maps that external port to your container's internal port 22. Connecting on port 22 will reach the proxy itself and immediately close.

#### From your phone or tablet

Use any SSH app:
- [Termius](https://termius.com) (iOS, Android)
- [Echo](https://replay.software/echo) (iOS)
- [Blink Shell](https://blink.sh) (iOS)

Enter the hostname, Railway-assigned port, username `user`, and the `SSH_PASSWORD` from your service's **[Variables](https://docs.railway.com/variables)** tab.

#### From your laptop

```bash
ssh user@<hostname> -p <port>
```

Enter the `SSH_PASSWORD` when prompted. The hostname and port stay the same across redeploys.

<details>
<summary><strong>Using SSH key authentication</strong></summary>

To use key-based auth instead of a password, paste your public key into `SSH_PUBLIC_KEY`:

1. Run:
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
2. If you get "No such file", generate a key first:
   ```bash
   ssh-keygen -t ed25519
   cat ~/.ssh/id_ed25519.pub
   ```
3. Copy the output (starts with `ssh-ed25519 ...`) and paste it into the `SSH_PUBLIC_KEY` field on Railway.

You can set both a password and a key. Useful if you want key auth from your computer and password auth from your phone.

**Multiple devices:** Any environment variable starting with `SSH_PUBLIC_KEY` is collected as an SSH key. Add one per device:

| Variable | Value |
|---|---|
| `SSH_PUBLIC_KEY_LAPTOP` | `ssh-ed25519 AAAA... user@laptop` |
| `SSH_PUBLIC_KEY_PHONE` | `ssh-ed25519 AAAA... user@phone` |

If you're in a team setting, deploy one container per developer. Each person gets their own persistent environment, tmux session, and agent logins.

</details>

---

### 3. Set Up

Once connected, log in to your services. Each is a one-time setup that persists across redeploys.

```bash
gh auth login         # GitHub — clone, push, create PRs
railway login         # Railway — manage services and deployments
```

Follow the device code prompts (you'll open a URL on any browser and enter a code). Claude Code will also prompt you to log in on first run.

---

### 4. Code

Clone a repo and start an agent:

```bash
cd /data
git clone https://github.com/youruser/yourrepo.git
cd yourrepo
claude
```

Start Claude Code with the `claude` command.

---

## Reset

To wipe persistent storage and start fresh:

1. Go to your service on Railway → **Settings → [Volume](https://docs.railway.com/volumes)**.
2. Click **Wipe Volume**.
3. Redeploy. The entrypoint reinitialises everything automatically.

<details>
<summary><strong>Selective reset</strong></summary>

SSH in and remove specific directories instead of wiping the full volume:

```bash
# Agent settings only (keeps repos)
rm -rf /data/.claude /data/.config

# SSH host keys (clients will need to accept the new key)
rm -rf /data/.ssh_host_keys

# Re-sync skills from the image without restarting
cp -r /opt/default-skills/* ~/.claude/skills/
```

</details>

---

## How It Works

1. Build — The Dockerfile builds an Ubuntu image with Claude Code, GitHub CLI, Railway CLI, and SSH server pre-installed.
2. Deploy — Railway runs the container and mounts a persistent [volume](https://docs.railway.com/volumes) at `/data` for repos, settings, and session history.
3. Connect — Railway's [TCP proxy](https://docs.railway.com/networking/tcp-proxy) exposes the container's SSH port to the internet on a stable hostname and port.
4. Persist — The entrypoint symlinks config directories (`~/.claude`, `~/.config`, etc.) to `/data` so everything survives redeploys.

---

## Security

- SSH hardening — root login disabled, max 3 auth attempts per connection, only `user` can connect. When password auth is active, `fail2ban` bans IPs after 5 failed attempts. Prefer SSH keys over passwords. The SSH port is exposed via Railway's [TCP proxy](https://docs.railway.com/networking/tcp-proxy).
- OAuth sessions — `gh auth login`, `railway login`, and Claude Code login store sessions in `~/.config/` (persisted on the [volume](https://docs.railway.com/volumes)). Run `gh auth logout` or `railway logout` to revoke. Set spending caps on your Anthropic account.
- Agent install scripts — the Dockerfile installs Claude Code via the [native installer](https://docs.anthropic.com/en/docs/claude-code) at build time.

---

## Troubleshooting

<details>
<summary><strong>Authentication succeeds but connection closes immediately</strong></summary>

If your SSH client shows "Authentication succeeded" followed by "Connection closed with error: end of file", you're connecting on the **wrong port**.

Railway's [TCP proxy](https://docs.railway.com/networking/tcp-proxy) assigns a specific port (e.g. `25377`) that maps to your container's internal port 22. If you connect on port 22 instead, you'll reach the proxy's own SSH server — it authenticates you, but has nothing to forward to.

**Fix:** Use the port shown in **Settings → Networking → TCP Proxy** (e.g. `gondola.proxy.rlwy.net:25377`), not port 22.

</details>

<details>
<summary><strong>Connection refused</strong></summary>

- The container may still be starting. Wait 30–60 seconds after deploy and try again.
- Verify you're using the correct hostname and port from **Settings → Networking → [TCP Proxy](https://docs.railway.com/networking/tcp-proxy)** on Railway.

</details>

<details>
<summary><strong>Permission denied (publickey)</strong></summary>

- Make sure you set the **public** key (`~/.ssh/id_ed25519.pub`), not the private key, in your [`SSH_PUBLIC_KEY*` variable](https://docs.railway.com/variables).
- Check that the key was pasted correctly with no extra whitespace or line breaks.
- Verify locally: `ssh-keygen -l -f ~/.ssh/id_ed25519.pub` should print the key fingerprint without errors.
- Try connecting with verbose output: `ssh -v user@<hostname> -p <port>` to see which keys are being tried.

</details>

<details>
<summary><strong>Permission denied (password)</strong></summary>

- Double-check the `SSH_PASSWORD` value in your Railway [environment variables](https://docs.railway.com/variables).
- Password auth is only enabled when `SSH_PASSWORD` is set. If you cleared it, redeploy.

</details>

<details>
<summary><strong>Host key verification failed</strong></summary>

This happens when the container got a new host key after redeployment. Remove the old key:

```bash
ssh-keygen -R "[<hostname>]:<port>"
```

Then connect again. Host keys are normally persisted across redeploys, so this should be rare.

</details>