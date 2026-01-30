---
layout: default
title: Linux Setup Guide
---

# Linux Setup Guide

**Complete beginner-friendly guide for Ubuntu, Debian, Fedora, and other Linux distributions**

This guide assumes you've never set up a programming environment before. Follow each step exactly as written.

---

## Choose Your Distribution

<div style="display: flex; gap: 20px; flex-wrap: wrap; margin: 20px 0;">
  <a href="#ubuntu-debian" style="padding: 15px 30px; background: #E95420; color: white; border-radius: 8px; text-decoration: none;">Ubuntu / Debian</a>
  <a href="#fedora-rhel" style="padding: 15px 30px; background: #0B57D0; color: white; border-radius: 8px; text-decoration: none;">Fedora / RHEL</a>
  <a href="#arch" style="padding: 15px 30px; background: #1793D1; color: white; border-radius: 8px; text-decoration: none;">Arch Linux</a>
</div>

---

## What You'll Install

| Software | Purpose | Size |
|----------|---------|------|
| PostgreSQL | Database for memories | ~100MB |
| Python | Programming language | ~100MB |
| Git | Download code | ~50MB |
| Node.js | For Claude Code | ~100MB |
| Claude Code | AI assistant | ~50MB |

**Total: ~500MB of disk space needed**

---

<h2 id="ubuntu-debian">Ubuntu / Debian Setup</h2>

### Step 1: Open Terminal

- Press `Ctrl + Alt + T` to open Terminal
- Or search for "Terminal" in your applications

### Step 2: Update Your System

```bash
sudo apt update && sudo apt upgrade -y
```

*Enter your password when asked (you won't see it as you type)*

### Step 3: Install PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib -y
```

Start PostgreSQL:

```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Set up your user:

```bash
sudo -u postgres createuser --superuser $USER
createdb mempheromone
```

Verify it works:

```bash
psql -d mempheromone -c "SELECT 'Database working!' as status;"
```

### Step 4: Install pgvector

```bash
sudo apt install postgresql-14-pgvector -y
```

*Note: Replace `14` with your PostgreSQL version if different*

Enable the extension:

```bash
psql -d mempheromone -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### Step 5: Install Python

```bash
sudo apt install python3 python3-pip python3-venv -y
```

Verify:

```bash
python3 --version
```

### Step 6: Install Git

```bash
sudo apt install git -y
```

### Step 7: Install Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
```

Verify:

```bash
node --version
npm --version
```

### Step 8: Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

**Continue to [Final Setup Steps](#final-setup-steps)**

---

<h2 id="fedora-rhel">Fedora / RHEL / CentOS Setup</h2>

### Step 1: Open Terminal

Search for "Terminal" in your applications

### Step 2: Update Your System

```bash
sudo dnf update -y
```

### Step 3: Install PostgreSQL

```bash
sudo dnf install postgresql postgresql-server postgresql-contrib -y
```

Initialize and start:

```bash
sudo postgresql-setup --initdb
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Set up your user:

```bash
sudo -u postgres createuser --superuser $USER
createdb mempheromone
```

### Step 4: Install pgvector

```bash
sudo dnf install pgvector -y
```

If not available, build from source:

```bash
sudo dnf install gcc postgresql-devel -y
git clone https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install
cd ..
rm -rf pgvector
```

Enable the extension:

```bash
psql -d mempheromone -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

### Step 5: Install Python

```bash
sudo dnf install python3 python3-pip -y
```

### Step 6: Install Git

```bash
sudo dnf install git -y
```

### Step 7: Install Node.js

```bash
sudo dnf install nodejs npm -y
```

### Step 8: Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

**Continue to [Final Setup Steps](#final-setup-steps)**

---

<h2 id="arch">Arch Linux Setup</h2>

### Step 1: Update System

```bash
sudo pacman -Syu
```

### Step 2: Install PostgreSQL

```bash
sudo pacman -S postgresql
```

Initialize and start:

```bash
sudo -u postgres initdb -D /var/lib/postgres/data
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

Set up your user:

```bash
sudo -u postgres createuser --superuser $USER
createdb mempheromone
```

### Step 3: Install pgvector

From AUR:

```bash
yay -S pgvector
```

Or build from source:

```bash
git clone https://github.com/pgvector/pgvector.git
cd pgvector
make
sudo make install
cd ..
rm -rf pgvector
```

### Step 4: Install Other Dependencies

```bash
sudo pacman -S python python-pip git nodejs npm
```

### Step 5: Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

**Continue to [Final Setup Steps](#final-setup-steps)**

---

<h2 id="final-setup-steps">Final Setup Steps (All Distributions)</h2>

### Step 9: Download Legal-Claw-RLMemory

```bash
cd ~
git clone https://github.com/Ibreitenbach/Legal-Claw-RLMemory.git
cd Legal-Claw-RLMemory
```

### Step 10: Run the Setup Script

```bash
./scripts/setup.sh
```

Follow the prompts:
- **Database name**: Press Enter to accept `mempheromone`
- If asked about existing installations, type `y` to overwrite

### Step 11: Configure Environment Variables

Open your shell configuration:

```bash
nano ~/.bashrc
```

Add these lines at the bottom:

```bash
# Mempheromone Database
export PGDATABASE=mempheromone
export PGUSER=$USER
export PGHOST=localhost
export PGPORT=5432
```

Save (`Ctrl + O`, Enter) and exit (`Ctrl + X`).

Reload:

```bash
source ~/.bashrc
```

### Step 12: Verify Everything Works

```bash
# Check database
psql -d mempheromone -c "SELECT COUNT(*) FROM debugging_facts;"

# Check plugin
ls ~/.claude/plugins/rlm-mempheromone/

# Test Claude Code
cd ~/Legal-Claw-RLMemory
claude
```

---

## Troubleshooting

### "Permission denied" when running scripts

```bash
chmod +x scripts/*.sh
```

### PostgreSQL won't start

Check if it's already running:
```bash
sudo systemctl status postgresql
```

Check logs:
```bash
sudo journalctl -u postgresql
```

### "Role does not exist"

Create your PostgreSQL user:
```bash
sudo -u postgres createuser --superuser $USER
```

### pgvector not found

Check your PostgreSQL version:
```bash
psql --version
```

Then search for the correct package:
```bash
apt search pgvector    # Ubuntu/Debian
dnf search pgvector    # Fedora
```

### "npm: command not found"

Node.js wasn't installed correctly. Reinstall:
```bash
# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y

# Fedora
sudo dnf install nodejs npm -y
```

---

## Optional: Auto-start PostgreSQL

PostgreSQL should auto-start on boot. To verify:

```bash
sudo systemctl is-enabled postgresql
```

If it shows "disabled":

```bash
sudo systemctl enable postgresql
```

---

## Optional: Background Memory Processing

Set up a cron job for automatic memory organization:

```bash
crontab -e
```

Add this line:

```
0 * * * * /home/$USER/Legal-Claw-RLMemory/scripts/membox_cron.sh
```

This runs hourly to organize memories into topic boxes.

---

## Next Steps

You're all set! Here's what to do next:

1. **Try Claude Code**: Run `claude` and start asking questions
2. **Read the Immigration Guide**: [Immigration Law Guide](https://github.com/Ibreitenbach/Legal-Claw-RLMemory/blob/main/examples/legal-research/IMMIGRATION_LAW_GUIDE.md)
3. **Learn About the Architecture**: [Architecture Docs](ARCHITECTURE)

---

<div align="center">

[Back to Start Here](START_HERE) | [Windows Guide](START-HERE-WINDOWS) | [macOS Guide](START-HERE-MACOS)

</div>
