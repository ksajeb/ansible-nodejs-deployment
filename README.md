# Ansible Node.js Deployment

Deploy a Node.js application to a remote Ubuntu server with a single Ansible playbook. It installs the runtime, pulls your app from GitHub, installs its dependencies, and runs it as a `systemd` service that starts on boot and restarts on failure.

```bash
ansible-playbook -i inventory.ini playbook.yml
```

---

## Table of Contents

- [What It Does](#what-it-does)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [How the Playbook Works](#how-the-playbook-works)
- [Verifying the Deployment](#verifying-the-deployment)
- [Managing the Service](#managing-the-service)
- [Deploying Your Own App](#deploying-your-own-app)
- [Troubleshooting](#troubleshooting)
- [Security Notes](#security-notes)
- [Contributing](#contributing)

---

## What It Does

Running the playbook against a fresh server will:

1. Install `git` and `curl`
2. Add the official [NodeSource](https://github.com/nodesource/distributions) apt repository for your chosen Node.js version
3. Install Node.js and npm
4. Clone your application repository into `/node_app`
5. Run `npm install` for the app's dependencies
6. Verify the Node.js installation
7. Create a `node-app` systemd unit that runs `npm start`
8. Start the service and enable it on boot

By default it deploys [`rat9615/simple-nodejs-app`](https://github.com/rat9615/simple-nodejs-app) with Node.js 22, listening on port 3000.

---

## Requirements

### Control machine (where you run Ansible)

| Requirement | Notes |
|---|---|
| Ansible | Install with `pip install ansible` or your package manager |
| `community.general` collection | Provides the `npm` module. Included in the full `ansible` package; with `ansible-core` only, run `ansible-galaxy collection install community.general` |
| SSH access | Private key for the target server |

### Target server

| Requirement | Notes |
|---|---|
| Ubuntu or Debian | The playbook uses `apt`, so RPM-based distros (Amazon Linux, RHEL, CentOS) are **not** supported as-is |
| SSH user with `sudo` | The playbook uses `become: yes`. On Ubuntu EC2 images this user is `ubuntu` |
| Outbound internet | To reach NodeSource, GitHub, and the npm registry |
| Open inbound port | `22` for SSH, and `3000` (or your app's port) for the application |
| Python 3 | Required by Ansible on managed hosts; present on standard Ubuntu images |

---

## Project Structure

```
ansible-nodejs-deployment/
├── inventory.ini     # Target hosts (group: webservers)
├── playbook.yml      # Deployment playbook
└── README.md
```

---

## Quick Start

### 1. Clone this repository

```bash
git clone https://github.com/ksajeb/ansible-nodejs-deployment.git
cd ansible-nodejs-deployment
```

### 2. Edit the inventory

Open `inventory.ini` and replace the placeholders with your server details:

```ini
[webservers]
203.0.113.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/my-key.pem
```

- `203.0.113.10`: your server's public IP or hostname
- `ansible_user`: the SSH user (`ubuntu` on Ubuntu AMIs)
- `ansible_ssh_private_key_file`: path to your private key

> The repository ships with `ansible_user=ec2-user` as a placeholder. Use `ubuntu` for Ubuntu servers, since the playbook is apt-based and the service runs as `ubuntu`.

Make sure your key has safe permissions:

```bash
chmod 400 ~/.ssh/my-key.pem
```

### 3. Test connectivity

```bash
ansible -i inventory.ini webservers -m ping
```

You should see `"ping": "pong"` for each host.

### 4. Run the playbook

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Add `--check` for a dry run, or `-v` for more detail:

```bash
ansible-playbook -i inventory.ini playbook.yml --check
ansible-playbook -i inventory.ini playbook.yml -v
```

When it finishes, the last task prints:

```
SUCCESS: Node.js application deployment completed!
```

### 5. Open the app

```
http://<your-server-ip>:3000
```

---

## Configuration

Variables live in the `vars:` block at the top of `playbook.yml`.

| Variable | Default | Description |
|---|---|---|
| `github_repo` | `https://github.com/rat9615/simple-nodejs-app.git` | Git repository of the app to deploy |
| `app_dir` | `/node_app` | Directory on the server where the app is cloned |
| `nodejs_version` | `"22"` | Node.js major version installed from NodeSource |

You can override any of them at run time without editing the file:

```bash
ansible-playbook -i inventory.ini playbook.yml \
  -e github_repo=https://github.com/<you>/<your-app>.git \
  -e nodejs_version=20 \
  -e app_dir=/opt/myapp
```

Values hardcoded in the systemd unit inside the playbook:

| Setting | Value | Where to change it |
|---|---|---|
| Service name | `node-app` | `Create systemd service` task and the `Restart` handler |
| Service user | `ubuntu` | `User=` in the unit file |
| Port | `3000` | `Environment=PORT=3000` in the unit file |
| Start command | `/usr/bin/npm start` | `ExecStart=` in the unit file |

---

## How the Playbook Works

| # | Task | Module | Purpose |
|---|---|---|---|
| 1 | Installing Git And Curl | `apt` | Installs prerequisites and refreshes the apt cache |
| 2 | Configure Node.js repository | `shell` | Downloads and runs the NodeSource setup script for `nodejs_version`. Skipped if `/etc/apt/sources.list.d/nodesource.list` already exists |
| 3 | Install Node.js and npm | `apt` | Installs the `nodejs` package (npm is bundled) |
| 4 | Create application directory | `file` | Creates `app_dir` with mode `0755` |
| 5 | Clone application repository | `git` | Clones or updates the app into `app_dir` (`force: yes` discards local changes) |
| 6 | Install application dependencies | `npm` | Runs `npm install` in `app_dir` |
| 7 | Verify Node.js installation | `command` | Runs `node --version` |
| 8 | Create systemd service | `copy` | Writes `/etc/systemd/system/node-app.service` and notifies the restart handler on change |
| 9 | Start and enable service | `systemd` | Reloads systemd, starts `node-app`, and enables it on boot |
| 10 | Display success message | `debug` | Prints the completion message |

**Handler:** `Restart Node.js application` restarts the service whenever the unit file changes.

### Re-running the playbook

The playbook is safe to run repeatedly. Re-running it pulls the latest commit of the app, reinstalls dependencies, and keeps the service enabled. Note that the service itself is only restarted when the unit file changes, so after pulling new app code, restart it manually (see below) or add a restart step.

---

## Verifying the Deployment

SSH into the server and check:

```bash
# Node.js version
node --version

# Service status
sudo systemctl status node-app

# Recent logs
sudo journalctl -u node-app -n 50 --no-pager

# Is the app responding locally?
curl http://localhost:3000
```

---

## Managing the Service

```bash
sudo systemctl status node-app     # status
sudo systemctl restart node-app    # restart (e.g. after pulling new code)
sudo systemctl stop node-app       # stop
sudo systemctl disable node-app    # don't start on boot
sudo journalctl -u node-app -f     # follow logs live
```

---

## Deploying Your Own App

Point the playbook at any Node.js repository that meets these conditions:

- `package.json` exists at the **repository root**
- The app starts with `npm start` (define a `start` script)
- The app reads its port from `process.env.PORT`

```bash
ansible-playbook -i inventory.ini playbook.yml \
  -e github_repo=https://github.com/<you>/<your-app>.git
```

If your app listens on a different port, change `Environment=PORT=3000` in the playbook and open that port in your firewall or cloud security group. For private repositories, use an SSH URL with a deploy key or an HTTPS URL with a token.

---

## Troubleshooting

**`UNREACHABLE` or SSH timeout**
Check the IP, that port 22 is open in your security group, that the key path is correct, and that the key is `chmod 400`.

**`Permission denied (publickey)`**
Wrong `ansible_user` for the image. Ubuntu uses `ubuntu`; Amazon Linux uses `ec2-user` (but is not supported by this playbook).

**`apt` module fails or "No package manager found"**
The target is not Debian/Ubuntu. Use an Ubuntu image.

**`couldn't resolve module/action 'npm'`**
Install the collection: `ansible-galaxy collection install community.general`.

**App is running but the browser can't connect**
Open inbound TCP port `3000` in your cloud security group or firewall (`sudo ufw allow 3000` if UFW is enabled).

**Service fails to start**
Read the logs with `sudo journalctl -u node-app -n 100 --no-pager`. Common causes: no `start` script in `package.json`, the app expects a different entry point, or the app doesn't use the `PORT` environment variable.

**Service fails with `status=217/USER`**
The `ubuntu` user doesn't exist on the server. Change `User=` in the unit file to a valid user.

---

## Security Notes

- **Never commit private keys.** Keep `.pem` files out of the repository. Add `*.pem` to `.gitignore`.
- **Limit exposure.** Restrict SSH (port 22) in your security group to your own IP rather than `0.0.0.0/0`.
- **Production use.** Consider putting Nginx or a load balancer with HTTPS in front of the app rather than exposing port 3000 directly.
- **Review remote scripts.** The playbook runs the NodeSource setup script downloaded over HTTPS, so make sure you trust the source.

---

## Contributing

Issues and pull requests are welcome. For changes to the playbook, test against a fresh Ubuntu VM and confirm it also runs cleanly a second time (idempotency).
