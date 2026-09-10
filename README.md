# Data Engineering & Analytics — Server Setup

This repo documents how I provisioned a cloud server from scratch and configured it to run a PostgreSQL database inside Docker, with the project version-controlled through GitHub. I built this as part of my data engineering learning path, and wrote it up so I (and anyone else) can reproduce the setup from a blank Ubuntu server.

## What this covers

- Launching an AWS EC2 (Ubuntu) server and connecting to it over SSH
- Generating an SSH key and linking the server to GitHub
- Cloning a private repo onto the server and fixing real permission issues along the way
- Editing the server's files remotely from VS Code (Remote-SSH)
- Installing Docker Engine and configuring it properly (non-root usage, auto-start on boot)
- Running PostgreSQL inside a Docker container

---

## 1. Launching the server (AWS EC2)

I used **EC2** (Amazon's "Elastic Compute Cloud") to spin up an Ubuntu virtual server — essentially a rentable computer that lives in AWS's data center instead of on my desk.

Once it was running, I connected to it from my terminal with:

```bash
ssh ubuntu@<server-ip-address>
```

**SSH** (Secure Shell) is what lets you remotely log into a server's command line, as if you were typing directly on it.

## 2. Setting up the project folder

```bash
cd /opt
sudo mkdir services
```

`/opt` is the standard Linux location for installed services/applications. Creating a folder here requires `sudo` (administrator privileges) — I hit a `Permission denied` error on the first try without it, which is expected.

## 3. Connecting the server to GitHub (SSH keys)

To let the server pull code privately from GitHub, I generated an SSH key pair on it:

```bash
ssh-keygen -t ed25519
```

This creates:
- `~/.ssh/id_ed*******` — the **private** key (kept secret, never shared)
- `~/.ssh/id_ed******.pub` — the **public** key (safe to share)

I printed the public key and copied it:

```bash
cat ~/.ssh/id_ed25519.pub
```

...then added it under **GitHub → Settings → SSH and GPG keys → New SSH key**, and verified the connection:

```bash
ssh -T git@github.com
```

## 4. Cloning the repo (and fixing permission issues)

```bash
cd /opt/services
git clone git@github.com:<your-username>/<your-repo>.git
```

**Issue #1 — `sudo git clone` failed with `Permission denied (publickey)`.**
SSH keys belong to a specific Linux user. My key was set up for the `ubuntu` user, but `sudo` runs as `root`, which had no key of its own. I fixed this by copying the same key files into `root`'s `.ssh` folder so both users could authenticate.

**Issue #2 — cloned files were owned by `root`, so I couldn't edit them as `ubuntu`.**
Fixed with:

```bash
sudo chown -R ubuntu:ubuntu services/
```

`chown` = "change owner"; `-R` applies it recursively to every file and folder inside.

## 5. Editing remotely with VS Code (Remote-SSH)

Instead of editing files only through the terminal, I connected VS Code directly to the server using its **Remote-SSH** extension. This lets me use a normal code editor (file explorer, syntax highlighting, etc.) while the files stay on the remote server. I set up an SSH config file so I can connect with one click instead of retyping the full SSH command each time.

## 6. Installing Docker

Following Docker's official installation steps:

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
# add Docker's repository to apt sources, then:
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

Verified the install with:

```bash
sudo docker run hello-world
```

## 7. Running Docker without `sudo`

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world   # now works without sudo
```

## 8. Making Docker start on boot

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

`systemctl` ("system + ctrl") is Linux's tool for controlling background services — starting, stopping, enabling, or disabling them. `enable` here means "start this automatically if the server reboots."

## 9. Running PostgreSQL in Docker

I generated a secure random password first:

```bash
openssl rand -base64 32
```

Then started the database container:

```bash
docker run --rm --name postgres-dev \
  -e POSTGRES_PASSWORD=<your-generated-password> \
  -p 5432:5432 \
  -d postgres:18
```

| Flag | Purpose |
|---|---|
| `--rm` | Automatically removes the container when it stops |
| `--name` | Gives the container a memorable name |
| `-e POSTGRES_PASSWORD=...` | Sets the database's admin password |
| `-p 5432:5432` | Maps the host port to the container's port so it's reachable |
| `-d` | Runs the container in the background |

---

## What I learned

- **SSH keys are per-user** — a key valid for one Linux user won't automatically work for another (e.g. `ubuntu` vs `root`). This is one of the most common gotchas on a fresh server.
- **`sudo` + `chown`** are the two tools I reach for first when I hit a "permission denied" error.
- **VS Code Remote-SSH** makes remote development feel local.
- **Docker** packages an app with everything it needs to run, so it behaves the same everywhere; `docker run hello-world` is the standard sanity check after install.
- **`systemctl enable <service>`** ensures critical services restart automatically after a reboot.
- Always generate real passwords (`openssl rand -base64 32`) instead of hand-typing something simple.

## Stack

`AWS EC2` · `Ubuntu` · `SSH` · `Git / GitHub` · `VS Code Remote-SSH` · `Docker` · `PostgreSQL`
