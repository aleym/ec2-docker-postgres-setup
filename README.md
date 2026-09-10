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



# Class 1 (Part 3) — Detailed Beginner's Guide

This is the continuation of the same class — picking up right after Docker and PostgreSQL were installed. This part is where things get **database-and-code focused**: writing Python to talk to the database, handling real server hiccups, and starting to actually design the database tables (which is the "Database Design, Normalisation, PostgreSQL and SQL" part promised on the title slide).

I'll go slow and explain every concept, since you asked for real detail.

---

## Part A — Writing Python code that talks to PostgreSQL

### Why do this at all?

Up to now, you've only talked to the database through DBeaver's visual interface. But in real projects, your *application code* (Python, in this case) needs to read and write data too. That means your code needs a way to connect to the database — and that's what this section builds.

### Step 1: Create a test database

Before touching any code, a fresh database was created purely for practice, using plain SQL typed into DBeaver's SQL editor:

```sql
create database nadibaby_test;
```

**What this does:** PostgreSQL can hold many separate databases on one server (like separate filing cabinets). This command creates a new empty one called `nadibaby_test`, so practice work doesn't touch the real `nadibaby_production` database.

### Step 2: Store your database credentials safely, using a `.env` file

Instead of typing your database password directly into your Python code (which is a security risk — especially if you push that code to GitHub!), the class used a **`.env` file** — a small text file that holds "environment variables."

`.env` file contents:
```
PGHOST=localhost
PGPORT=5432
PGDATABASE=nadibaby_production
PGUSER=myuser
PGPASSWORD=mypassword
```

And a `.env.example` file — a *template* version with placeholder values, safe to commit to GitHub, so anyone cloning the project knows what variables they need to fill in themselves:
```
PGHOST=localhost
PGPORT=5432
PGDATABASE=mydb
PGUSER=myuser
PGPASSWORD=mypassword
```

> ⚠️ **Critical rule taught in the class:** *"Never commit the real `.env` file to version control (add it to `.gitignore`)."* Your `.gitignore` file tells Git which files to never track — your real `.env` (with your actual password) should always be listed there. Only `.env.example` (fake placeholder values) goes to GitHub.

### Step 3: Install the Python libraries needed

Two libraries were installed:
- **`psycopg2`** — the library that lets Python actually talk to PostgreSQL
- **`python-dotenv`** — the library that reads your `.env` file and loads those values into Python

```bash
pip install psycopg2-binary python-dotenv
```

(`psycopg2-binary` is used instead of plain `psycopg2` because it comes pre-compiled — easier to install without extra system dependencies.)

### Step 4: Write the connection script (`db_connect.py`)

Here's the script built in class, with an explanation under each part:

```python
import os
import psycopg2
from dotenv import load_dotenv

# Loads variables from a .env file into the process environment.
load_dotenv()

DB_CONFIG = {
    "host": os.environ.get("PGHOST", "localhost"),
    "port": os.environ.get("PGPORT", "5432"),
    "dbname": os.environ.get("PGDATABASE", "postgres"),
    "user": os.environ.get("PGUSER", "postgres"),
    "password": os.environ.get("PGPASSWORD", ""),
}

def get_connection():
    """Open and return a new database connection."""
    return psycopg2.connect(**DB_CONFIG)

def main():
    conn = get_connection()
    try:
        with conn.cursor() as cur:
            cur.execute("SELECT version();")
            row = cur.fetchone()
            print("Connected to:", row[0])
    finally:
        conn.close()

if __name__ == "__main__":
    main()
```

**Line-by-line, in plain English:**

| Code | What it means |
|---|---|
| `load_dotenv()` | Reads your `.env` file and makes those values available to Python |
| `os.environ.get("PGHOST", "localhost")` | "Get the PGHOST variable; if it doesn't exist, default to 'localhost'" |
| `DB_CONFIG = {...}` | A dictionary (a labeled bundle of settings) holding all your connection details |
| `get_connection()` | A reusable function — call it whenever you need a fresh connection to the database |
| `psycopg2.connect(**DB_CONFIG)` | Actually opens the connection, using all the settings from `DB_CONFIG` |
| `conn.cursor()` | A "cursor" is your tool for running SQL commands and reading results |
| `cur.execute("SELECT version();")` | Runs a SQL command asking Postgres what version it's running |
| `cur.fetchone()` | Grabs the first (and here, only) row of the result |
| `conn.close()` | Always close the connection when you're done, to free up resources |

### Step 5: The `if __name__ == "__main__":` pattern (a very common beginner question)

The instructor actually used an AI coding assistant in VS Code to explain this exact line, since it confuses a lot of beginners. Here's the explanation, plain and simple:

- If you just call `main()` directly at the bottom of the file with no `if` check, then **any time this file gets imported by another file**, `main()` runs automatically too — which is usually *not* what you want.
- Wrapping it like this:
  ```python
  if __name__ == "__main__":
      main()
  ```
  means: **"Only run `main()` if this file was run directly (e.g. `python db_connect.py`) — not if it was imported into some other script."**
- This is the standard Python pattern for script entry points, and you'll see it in almost every real Python project.

**Simple example to cement it:**
```python
def main():
    print("Hello")

main()  # ← this line runs immediately, even if this file is just imported elsewhere
```
vs.
```python
def main():
    print("Hello")

if __name__ == "__main__":
    main()  # ← this only runs when you execute this file directly
```

### Step 6: `requirements.txt` — listing your project's dependencies

```
psycopg2-binary
python-dotenv
```

**Why this file matters:** instead of telling someone "you need to `pip install` these two libraries," you just list them in `requirements.txt`, and anyone (including future-you, on a new computer) can install everything in one go:

```bash
pip install -r requirements.txt
```

### Step 7: Real troubleshooting — "No module named pip"

Running the install command above initially failed with:
```
/usr/bin/python3: No module named pip
```

**What this means:** the server's Python installation didn't have `pip` (Python's package installer) set up.

**The fix:**
```bash
sudo apt update
sudo apt install python3-pip
```
Then confirm it worked:
```bash
python3 -m pip --version
```

### Step 8: Python virtual environments (`venv`) — and fixing a broken one

A **virtual environment** is an isolated, self-contained folder that holds its own copy of Python and its own installed libraries — separate from the rest of the system. This means different projects can use different (even conflicting) library versions without interfering with each other.

In the class, the first attempt at creating one didn't work properly — the folder was missing key pieces (`pip`, `activate` script, etc.), because the `venv` package itself wasn't installed. The fix:

```bash
sudo apt update
sudo apt install python3-venv python3-pip

cd /opt/services/nadibaby-data-analytics
rm -rf .venv                          # delete the broken one
python3 -m venv .venv                 # create a fresh virtual environment
source .venv/bin/activate             # "activate" it (start using it)

python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

> **Windows note:** the activation command is different on Windows since it's not a Bash shell:
> ```powershell
> .venv\Scripts\activate
> ```
> (But again — since this venv lives *on the Linux server*, and you're connecting to that server, you'd actually still type the Linux version above, inside your SSH/VS Code Remote-SSH terminal.)

Then testing the connection script:
```bash
python db_connect.py
```

---

## Part B — A real-world EC2 gotcha: your server's IP address can change!

Partway through, the instructor **stopped and restarted the EC2 instance** (from the AWS Console → Instance state → Stop, then Start again). This is a genuinely important lesson:

> **By default, a "stopped and started" EC2 instance gets a brand-new public IP address.** The old one (`47.129.150.225` in this class) was replaced with a new one (`52.221.214.18`).

This broke two things that had the old IP address hardcoded:
1. **VS Code's Remote-SSH connection** — had to be reconnected using the new IP
2. **DBeaver's saved database connection** — had to be edited (Connection settings → Host field) to point at the new IP

**Why this matters for you:** if you ever stop/start your own EC2 instance and suddenly "can't connect" to anything, the very first thing to check is whether the IP address changed. (The permanent fix for this — not covered in this clip, but worth knowing — is called an **Elastic IP**, which is a fixed IP address you can attach to your instance so it never changes.)

---

## Part C — Starting the actual database design (schemas & tables)

This is the heart of "Class 1's" real subject: designing a relational database.

### Creating a schema

```sql
CREATE SCHEMA IF NOT EXISTS operational;
```

**What's a schema?** Think of a schema as a named folder *inside* a database, used to group related tables together. Instead of dumping every table into one big pile, you organize them — here, everything related to the business's day-to-day operations goes into the `operational` schema.

### Creating the first table: `customers`

```sql
CREATE TABLE IF NOT EXISTS operational.customers (
    customer_id INTEGER PRIMARY KEY,
    customer_name VARCHAR(150) NOT NULL,
    email VARCHAR(150),
    phone_number VARCHAR(30),
    customer_segment VARCHAR(50),
    postcode VARCHAR(10),
    area VARCHAR(100),
    state VARCHAR(100),
    registration_date DATE
);
```

**Breaking this down column by column:**

| Column | Type | Meaning |
|---|---|---|
| `customer_id` | `INTEGER PRIMARY KEY` | A unique whole-number ID for each customer. `PRIMARY KEY` means this value must be unique and is used to identify each row — no two customers can share one. |
| `customer_name` | `VARCHAR(150) NOT NULL` | Text, up to 150 characters. `NOT NULL` means this field is required — a customer row can't be saved without a name. |
| `email` | `VARCHAR(150)` | Text field for their email, optional (no `NOT NULL`) |
| `phone_number` | `VARCHAR(30)` | Stored as text, not a number — phone numbers often have symbols like `+` and leading zeros that a number type would break |
| `customer_segment` | `VARCHAR(50)` | Some kind of category/grouping label for the customer (e.g. "regular," "VIP") |
| `postcode` | `VARCHAR(10)` | Also text, for the same reason as phone numbers — postcodes aren't really "numbers" you'd do math on |
| `area` / `state` | `VARCHAR(100)` | Location fields |
| `registration_date` | `DATE` | A proper date type — lets you sort, filter, and calculate with it correctly (unlike storing dates as plain text) |

The class was also mid-way through creating a second table, `operational.branches`, when the recording ends — following the same pattern (a `branch_id` as `INTEGER PRIMARY KEY`, then descriptive columns).

### Why design it this way? (A preview of "normalisation")

Notice `customers` and `branches` are **separate tables**, not one giant table with everything crammed in. This is the core idea behind **database normalisation** — splitting data into logical, non-repeating groups (customers in one table, branches in another, etc.), and later *linking* them together using IDs (this is what the "PK / FK" — Primary Key / Foreign Key — objective from the very first slide of the course refers to). This keeps data consistent and avoids storing the same information over and over.

---

## Summary of everything new in this part

1. **`.env` files** keep secrets (like passwords) out of your code — and out of GitHub.
2. **`psycopg2` + `python-dotenv`** is a standard combo for connecting Python to PostgreSQL.
3. **`if __name__ == "__main__":`** controls whether code runs on import vs. direct execution — a Python fundamental.
4. **`requirements.txt`** lists your project's dependencies so anyone can reinstall them in one command.
5. **Virtual environments (`venv`)** isolate each project's Python packages from the rest of the system.
6. **EC2 public IPs change** when you stop/start an instance (unless you set up an Elastic IP) — a very common real-world "why did my connection break?" moment.
7. **Schemas** group related tables inside a database.

# Using GCP (Google Cloud Platform) — A Detailed Beginner's Guide

The class notes so far only covered AWS. This guide covers the GCP equivalent of the same workflow — since a `GCP-Ubuntu` host appeared in the class's SSH config but was never explained on screen.

Almost everything you already learned transfers directly — SSH, Docker, PostgreSQL, Git don't change. Only *how you create and access the server* differs.

## AWS → GCP: matching up the concepts

| AWS term | GCP equivalent | What it is |
|---|---|---|
| EC2 | Compute Engine | The service for renting virtual servers |
| EC2 Instance | VM instance | An individual virtual server |
| Security Group | Firewall rules | Controls what traffic can reach your server |
| Key Pair | SSH keys (per-project or per-instance) | How you prove your identity to log in |
| Elastic IP | Static external IP address | A fixed IP that doesn't change on restart |
| AWS Console | Google Cloud Console | The web dashboard for everything |
| Region/Availability Zone | Region/Zone | Same concept, same naming |

## Step 1: Create a Google Cloud account

1. Go to console.cloud.google.com and sign in with a Google account.
2. Create a new Project via the project dropdown (top-left) → New Project.
3. Search "Compute Engine" in the top search bar and click Enable (first time only).

## Step 2: Create your first VM instance

1. Compute Engine → VM instances → Create Instance.
2. Name it, pick a Region/Zone close to you (e.g. asia-southeast1 for Singapore).
3. Machine type: e2-medium (roughly equivalent to AWS's t2.small).
4. Boot disk → Change → Public images → Ubuntu → Ubuntu 24.04 LTS → Select.
5. Check "Allow HTTP traffic" / "Allow HTTPS traffic" if needed (SSH/port 22 is open by default).
6. Click Create.

## Step 3: Connecting to your GCP server

Three options, easiest to most useful for this class:

**A. Browser-based SSH (built-in)** — click the "SSH" button next to your instance in the console. Opens a terminal in a new tab instantly, no key setup. Good for a quick look, but other tools (like VS Code) can't use this session.

**B. gcloud CLI (recommended)**
```
# Install the Google Cloud CLI from cloud.google.com/sdk, then:
gcloud auth login
gcloud compute ssh <instance-name> --zone=<your-zone>
```
Handles key management for you automatically.

**C. Manual SSH key (matches your AWS workflow exactly)**
```
ssh-keygen -t ed25519
```
Then in the Console: Compute Engine → Metadata → SSH Keys → Add your public key.
Connect the same way you did on AWS:
```
ssh <username>@<external-ip>
```
This is the method that plugs directly into VS Code Remote-SSH.

## Step 4: Opening a port for PostgreSQL (equivalent of the AWS security group fix)

1. Console search bar → "Firewall" → VPC network → Firewall.
2. Create Firewall Rule → name it e.g. "allow-postgres".
3. Targets: "All instances in the network" (simplest for learning).
4. Source IPv4 ranges: your IP + /32 (safer) or 0.0.0.0/0 (testing only).
5. Protocols and ports: TCP, port 5432.
6. Create. Takes effect immediately.

## Step 5: VS Code SSH config for both servers at once

`~/.ssh/config` (Mac/Linux) or `C:\Users\<You>\.ssh\config` (Windows):

```
Host AWS-Ubuntu
    HostName 47.129.150.225
    User ubuntu

Host GCP-Ubuntu
    HostName 136.119.233.46
    User ubuntu
```

VS Code's Remote-SSH extension then lets you pick either host from a list instead of retyping IPs.

## Step 6: Everything after this is identical

Once connected, every command from the rest of your notes works unchanged:
```bash
cd /opt
sudo mkdir services
git clone git@github.com:<username>/<repo>.git
sudo chown -R ubuntu:ubuntu services/
```
Docker install, PostgreSQL, the Python `.env`/`psycopg2` setup, DBeaver (just point it at the GCP external IP) — nothing changes. AWS and GCP are different landlords renting you the same kind of apartment.

## GCP-specific things worth knowing

- **External IPs change on restart** too, unless you reserve a Static External IP (VM instances → click instance → Reserve Static External IP Address) — the direct equivalent of an AWS Elastic IP.
- **Billing**: stop your VM when not in use (no compute charges while stopped); set a budget alert under Billing → Budgets & alerts; watch your free trial credit balance.
- **Default SSH username** may be based on your Google account name rather than always "ubuntu" like on AWS — if `ssh <username>@<ip>` fails, run `gcloud compute ssh` once to see the username it uses.

8. **`PRIMARY KEY`**, **`NOT NULL`**, and choosing sensible column types (`VARCHAR` vs `INTEGER` vs `DATE`) are the basic building blocks of designing a table.
9. Splitting data into separate, linked tables (customers, branches, etc.) instead of one giant table is the beginning of **database normalisation**.

If a follow-up video continues the table design (branches, foreign keys linking tables together, and actual normalisation rules), send it over and I'll add it to this guide.

