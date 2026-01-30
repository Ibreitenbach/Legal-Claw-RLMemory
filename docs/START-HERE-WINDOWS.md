---
layout: default
title: Windows Setup Guide
---

# Windows Setup Guide

**Complete beginner-friendly guide for Windows 10/11**

This guide assumes you've never set up a programming environment before. Follow each step exactly as written.

---

## What You'll Install

| Software | Purpose | Size |
|----------|---------|------|
| WSL2 | Linux environment on Windows | ~1GB |
| Ubuntu | Linux distribution | ~500MB |
| PostgreSQL | Database for memories | ~100MB |
| Python | Programming language | ~100MB |
| Git | Download code | ~50MB |
| Claude Code | AI assistant | ~50MB |

**Total: ~2GB of disk space needed**

---

## Step 1: Enable WSL2 (Windows Subsystem for Linux)

WSL2 lets you run Linux inside Windows. This is required because the system runs best on Linux.

### 1.1 Open PowerShell as Administrator

1. Click the **Start** button (Windows icon)
2. Type `PowerShell`
3. Right-click **Windows PowerShell**
4. Click **Run as administrator**
5. Click **Yes** when asked for permission

### 1.2 Install WSL2

In the blue PowerShell window, type this command and press Enter:

```powershell
wsl --install
```

**Wait 5-10 minutes.** You'll see downloads happening.

### 1.3 Restart Your Computer

When installation finishes:
1. Save any open work
2. Restart your computer

### 1.4 Complete Ubuntu Setup

After restart:
1. A black window titled "Ubuntu" will open automatically
2. If not, click Start and search for "Ubuntu"
3. Wait for "Installing..." to finish
4. Create a username (lowercase, no spaces) - example: `john`
5. Create a password (you won't see it as you type - that's normal)
6. Type the password again to confirm

**Write down your username and password!** You'll need them later.

---

## Step 2: Update Ubuntu

In the Ubuntu window, run these commands one at a time:

```bash
sudo apt update
```
*Type your password when asked (you won't see it appear)*

```bash
sudo apt upgrade -y
```
*Wait for this to finish (may take several minutes)*

---

## Step 3: Install PostgreSQL

PostgreSQL is the database that stores your AI's memories.

### 3.1 Install PostgreSQL

```bash
sudo apt install postgresql postgresql-contrib -y
```

### 3.2 Start PostgreSQL

```bash
sudo service postgresql start
```

### 3.3 Set Up Your Database User

```bash
sudo -u postgres createuser --superuser $USER
```

### 3.4 Create the Database

```bash
createdb mempheromone
```

### 3.5 Verify It Works

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

## Step 4: Install pgvector (Semantic Search)

pgvector enables AI-powered semantic search. This is optional but recommended.

```bash
sudo apt install postgresql-14-pgvector -y
```

Then enable it in your database:

```bash
psql -d mempheromone -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

---

## Step 5: Install Python

```bash
sudo apt install python3 python3-pip python3-venv -y
```

Verify installation:

```bash
python3 --version
```

Should show: `Python 3.10.x` or higher

---

## Step 6: Install Git

```bash
sudo apt install git -y
```

Verify installation:

```bash
git --version
```

---

## Step 7: Install Node.js (for Claude Code)

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install nodejs -y
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

### 9.1 Go to Your Home Directory

```bash
cd ~
```

### 9.2 Download the Repository

```bash
git clone https://github.com/Ibreitenbach/Legal-Claw-RLMemory.git
```

### 9.3 Go Into the Directory

```bash
cd Legal-Claw-RLMemory
```

---

## Step 10: Run the Setup Script

```bash
./scripts/setup.sh
```

Follow the prompts:
- **Database name**: Press Enter to accept `mempheromone` (or type a different name)
- If asked about existing installations, type `y` to overwrite

---

## Step 11: Configure Environment Variables

### 11.1 Open Your Shell Configuration

```bash
nano ~/.bashrc
```

### 11.2 Add These Lines at the Bottom

Use arrow keys to go to the end of the file, then add:

```bash
# Mempheromone Database
export PGDATABASE=mempheromone
export PGUSER=$USER
export PGHOST=localhost
export PGPORT=5432
```

### 11.3 Save and Exit

1. Press `Ctrl + O` (letter O, not zero)
2. Press `Enter` to confirm
3. Press `Ctrl + X` to exit

### 11.4 Reload Configuration

```bash
source ~/.bashrc
```

---

## Step 12: Verify Everything Works

### Check Database:
```bash
psql -d mempheromone -c "SELECT COUNT(*) FROM debugging_facts;"
```

### Check Plugin:
```bash
ls ~/.claude/plugins/rlm-mempheromone/
```

### Test Claude Code:
```bash
cd ~/Legal-Claw-RLMemory
claude
```

You should see Claude Code start with your memory system loaded!

---

## Starting PostgreSQL After Reboot

Every time you restart your computer, you need to start PostgreSQL:

```bash
sudo service postgresql start
```

**Pro tip:** Add this to your `.bashrc` to start it automatically:
```bash
echo 'sudo service postgresql start' >> ~/.bashrc
```

---

## Troubleshooting

### "WSL not recognized"
- Make sure you restarted your computer after installing WSL
- Try running `wsl --install` again

### "Permission denied"
- Make sure you're using `sudo` for commands that require it
- Check that you typed your password correctly

### "Command not found"
- Make sure you installed all prerequisites
- Try closing and reopening Ubuntu

### "Database connection refused"
- Run `sudo service postgresql start`
- Check that PostgreSQL is running: `sudo service postgresql status`

### "Unable to locate package postgresql-14-pgvector"
- Your Ubuntu version may use a different PostgreSQL version
- Try: `sudo apt search pgvector` to find the right package

---

## Next Steps

You're all set! Here's what to do next:

1. **Try Claude Code**: Run `claude` and start asking questions
2. **Read the Immigration Guide**: [Immigration Law Guide](https://github.com/Ibreitenbach/Legal-Claw-RLMemory/blob/main/examples/legal-research/IMMIGRATION_LAW_GUIDE.md)
3. **Learn About the Architecture**: [Architecture Docs](ARCHITECTURE)

---

<div align="center">

[Back to Start Here](START_HERE) | [Linux Guide](START-HERE-LINUX) | [macOS Guide](START-HERE-MACOS)

</div>
