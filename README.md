# Data Engineering & Analytics — Server Setup

This repo documents how I provisioned a cloud server from scratch and configured it to run a PostgreSQL database inside Docker, with the project version-controlled through GitHub, and connected visually using DBeaver. I built this as part of my data engineering learning path, and wrote it up so I (and anyone else) can reproduce the setup from a blank Ubuntu server.

## What this covers

- Launching an AWS EC2 (Ubuntu) server and connecting to it over SSH
- Generating an SSH key and linking the server to GitHub
- Cloning a private repo onto the server and fixing real permission issues along the way
- Editing the server's files remotely from VS Code (Remote-SSH)
- Installing Docker Engine and configuring it properly (non-root usage, auto-start on boot)
- Running PostgreSQL inside a Docker container
- Connecting to PostgreSQL visually with DBeaver, and fixing a firewall issue along the way

---

## 1. Launching the server (AWS EC2)

I used **EC2** ("Elastic Compute Cloud") to spin up an Ubuntu virtual server — a rentable computer that lives in AWS's data center instead of on my desk.

- Image: **Ubuntu 24.04**
- Instance type: **t2.small**
- A **security group** (a firewall) allowing SSH, HTTP, and HTTPS traffic

Once it was running, I connected to it from my terminal:

```bash
ssh ubuntu@<server-ip-address>
```

**SSH** (Secure Shell) is what lets you remotely log into a server's command line, as if you were typing directly on it.

> **Windows users:** use **Windows Terminal** or **PowerShell** (both come built into Windows 10/11 with SSH support) instead of the Mac Terminal app. The `ssh` command itself is identical.

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
- `id_ed25519` — the **private** key (kept secret, never shared)
- `id_ed25519.pub` — the **public** key (safe to share)

On Linux/Mac these live in `~/.ssh/`. On Windows they'd live in `C:\Users\<YourUsername>\.ssh\` — but since this key was generated *on the server itself* (a Linux machine), the path is `~/.ssh/` either way.

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

This extension works identically on Windows, Mac, and Linux — install VS Code, install "Remote - SSH," and connect.

## 6. Installing Docker

Following Docker's official installation steps (run on the server, so identical regardless of your own OS):

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

`systemctl` ("system + ctrl") is Linux's tool for controlling background services — starting, stopping, enabling, or disabling them. `enable` here means "start this automatically if the server reboots." It requires `sudo` because it's a high-privilege operation that can stop critical services.

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

## 10. Connecting to PostgreSQL with DBeaver

**DBeaver** is a free, graphical tool for browsing and querying databases — instead of typing raw SQL in a terminal, you get a visual interface. It's identical across Windows, Mac, and Linux.

### Issue I hit: connection failed

My first connection attempt failed with:
```
Can't connect to database
Connection to '...' cannot be established.
```

**Cause:** my EC2 server's security group (its firewall) didn't allow incoming traffic on PostgreSQL's port, 5432, by default.

**Fix:** I went to **AWS → EC2 → Security Groups → Inbound rules → Edit inbound rules → Add rule**, set Type to "PostgreSQL" (auto-fills port 5432), and Source to "My IP". After saving, the connection worked.

### How I connected

1. Downloaded DBeaver Community Edition from dbeaver.io and installed it.
2. Opened port 5432 in my EC2 security group (see above).
3. In DBeaver, created a new PostgreSQL connection using my server's public IP as the Host, port 5432, username `postgres`, and my database password.
4. Clicked "Test Connection..." to confirm it worked.
5. Clicked Finish — the database now appears in DBeaver's sidebar, and I can browse tables and run SQL queries visually.

---

## What I learned

- **SSH keys are per-user** — a key valid for one Linux user won't automatically work for another (e.g. `ubuntu` vs `root`). This is one of the most common gotchas on a fresh server.
- **`sudo` + `chown`** are the two tools I reach for first when I hit a "permission denied" error.
- **VS Code Remote-SSH** makes remote development feel local.
- **Docker** packages an app with everything it needs to run, so it behaves the same everywhere; `docker run hello-world` is the standard sanity check after install.
- **`systemctl enable <service>`** ensures critical services restart automatically after a reboot.
- Always generate real passwords (`openssl rand -base64 32`) instead of hand-typing something simple.
- **Firewall rules (security groups) are a common blocker** when a database client "can't connect" — always check whether the required port is open before assuming it's a credentials problem.

## Stack

`AWS EC2` · `Ubuntu` · `SSH` · `Git / GitHub` · `VS Code Remote-SSH` · `Docker` · `PostgreSQL` · `DBeaver`
