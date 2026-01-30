---
layout: default
title: macOS Setup Guide
---

# macOS Setup Guide

**Complete beginner-friendly guide for macOS Monterey, Ventura, and Sonoma**

This guide assumes you've never set up a programming environment before. Follow each step exactly as written.

---

## What You'll Install

| Software | Purpose | Size |
|----------|---------|------|
| Homebrew | Package manager | ~500MB |
| PostgreSQL | Database for memories | ~100MB |
| Python | Programming language | ~100MB |
| Git | Download code | ~50MB |
| Node.js | For Claude Code | ~100MB |
| Claude Code | AI assistant | ~50MB |

**Total: ~1GB of disk space needed**

---

## Step 1: Open Terminal

1. Press `Cmd + Space` to open Spotlight
2. Type `Terminal`
3. Press `Enter`

A white or black window will open - this is Terminal.

---

## Step 2: Install Xcode Command Line Tools

This includes essential developer tools (including Git).

```bash
xcode-select --install
```

A popup will appear. Click **Install**, then **Agree** to the license.

*Wait for installation to complete (5-10 minutes)*

---

## Step 3: Install Homebrew

Homebrew is macOS's package manager - it makes installing software easy.

Copy and paste this entire command:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Press Enter and follow the prompts:
- Enter your Mac password when asked (you won't see it as you type)
- Press `Enter` when asked to continue

**Important: After installation, you'll see instructions to add Homebrew to your PATH.**

For Apple Silicon Macs (M1/M2/M3), run:

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

For Intel Macs:

```bash
echo 'eval "$(/usr/local/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/usr/local/bin/brew shellenv)"
```

Verify Homebrew works:

```bash
brew --version
```

---

## Step 4: Install PostgreSQL

```bash
brew install postgresql@14
```

Start PostgreSQL:

```bash
brew services start postgresql@14
```

Add PostgreSQL to your PATH:

```bash
echo 'export PATH="/opt/homebrew/opt/postgresql@14/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

*For Intel Macs, replace `/opt/homebrew` with `/usr/local`*

Create your database:

```bash
createdb mempheromone
```

Verify it works:

```bash
psql -d mempheromone -c "SELECT 'Database working!' as status;"
```

You should see:
```
      status
-------------------
 Database working!
```

---

## Step 5: Install pgvector

```bash
brew install pgvector
```

Enable the extension:

```bash
psql -d mempheromone -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

---

## Step 6: Install Python

macOS comes with Python, but we'll install the latest version:

```bash
brew install python@3.11
```

Make sure you're using the new Python:

```bash
echo 'export PATH="/opt/homebrew/opt/python@3.11/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

*For Intel Macs, replace `/opt/homebrew` with `/usr/local`*

Verify:

```bash
python3 --version
```

Should show: `Python 3.11.x`

---

## Step 7: Install Node.js

```bash
brew install node
```

Verify:

```bash
node --version
npm --version
```

---

## Step 8: Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

---

## Step 9: Download Legal-Claw-RLMemory

Go to your home directory:

```bash
cd ~
```

Download the repository:

```bash
git clone https://github.com/Ibreitenbach/Legal-Claw-RLMemory.git
```

Go into the directory:

```bash
cd Legal-Claw-RLMemory
```

---

## Step 10: Run the Setup Script

```bash
./scripts/setup.sh
```

Follow the prompts:
- **Database name**: Press Enter to accept `mempheromone`
- If asked about existing installations, type `y` to overwrite

---

## Step 11: Configure Environment Variables

Open your shell configuration:

```bash
nano ~/.zshrc
```

Add these lines at the bottom:

```bash
# Mempheromone Database
export PGDATABASE=mempheromone
export PGUSER=$USER
export PGHOST=localhost
export PGPORT=5432
```

Save:
1. Press `Ctrl + O`
2. Press `Enter`
3. Press `Ctrl + X` to exit

Reload:

```bash
source ~/.zshrc
```

---

## Step 12: Verify Everything Works

Check database:

```bash
psql -d mempheromone -c "SELECT COUNT(*) FROM debugging_facts;"
```

Check plugin:

```bash
ls ~/.claude/plugins/rlm-mempheromone/
```

Test Claude Code:

```bash
cd ~/Legal-Claw-RLMemory
claude
```

You should see Claude Code start with your memory system loaded!

---

## PostgreSQL Auto-Start

PostgreSQL will automatically start when your Mac boots up because we used `brew services start`.

To check if it's running:

```bash
brew services list
```

Should show `postgresql@14` with status `started`.

---

## Troubleshooting

### "brew: command not found"

Homebrew wasn't added to your PATH. Run the commands from Step 3 again:

For Apple Silicon:
```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

For Intel:
```bash
eval "$(/usr/local/bin/brew shellenv)"
```

### "createdb: command not found"

PostgreSQL isn't in your PATH:

```bash
# Apple Silicon
export PATH="/opt/homebrew/opt/postgresql@14/bin:$PATH"

# Intel
export PATH="/usr/local/opt/postgresql@14/bin:$PATH"
```

### "psql: connection refused"

PostgreSQL isn't running. Start it:

```bash
brew services start postgresql@14
```

### "Permission denied" on scripts

Make them executable:

```bash
chmod +x scripts/*.sh
```

### Apple Silicon vs Intel - How to Check

Click the Apple menu () > **About This Mac**

- **Apple Silicon**: Shows "Chip" as M1, M2, M3, etc.
- **Intel**: Shows "Processor" as Intel Core i5, i7, etc.

### "xcrun: error: invalid active developer path"

Reinstall Xcode Command Line Tools:

```bash
xcode-select --install
```

---

## Optional: Background Memory Processing

Set up automatic memory organization:

```bash
crontab -e
```

If it asks which editor to use, type `nano` and press Enter.

Add this line:

```
0 * * * * /Users/$USER/Legal-Claw-RLMemory/scripts/membox_cron.sh
```

Save (`Ctrl + O`, Enter) and exit (`Ctrl + X`).

---

## Next Steps

You're all set! Here's what to do next:

1. **Try Claude Code**: Run `claude` and start asking questions
2. **Read the Immigration Guide**: [Immigration Law Guide](https://github.com/Ibreitenbach/Legal-Claw-RLMemory/blob/main/examples/legal-research/IMMIGRATION_LAW_GUIDE.md)
3. **Learn About the Architecture**: [Architecture Docs](ARCHITECTURE)

---

<div align="center">

[Back to Start Here](START_HERE) | [Windows Guide](START-HERE-WINDOWS) | [Linux Guide](START-HERE-LINUX)

</div>
