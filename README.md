# Ubuntu VPS Production Server Setup Guide

A complete, clean, and reusable step-by-step guide to set up any Ubuntu VPS from scratch. Save this document for future server setups.

---

## 📋 Table of Contents

1. [System Update &amp; Essential Packages](#1-system-update--essential-packages)
2. [Git &amp; GitHub CLI Setup](#2-git--github-cli-setup)
3. [Node.js Environment (via NVM)](#3-nodejs-environment-via-nvm)
4. [MongoDB Community Edition Setup](#4-mongodb-community-edition-setup)
5. [PostgreSQL Installation &amp; Database Setup](#5-postgresql-installation--database-setup)
6. [Redis Server Installation &amp; Configuration](#6-redis-server-installation--configuration)
7. [PM2 Process Manager &amp; Auto-Boot Persistence](#7-pm2-process-manager--auto-boot-persistence)
8. [Recommended: 4GB Swap Setup](#8-recommended-4gb-swap-setup)
9. [Recommended: UFW Firewall Configuration](#9-recommended-ufw-firewall-configuration)
10. [Quick Verification Cheat Sheet](#10-quick-verification-cheat-sheet)

---

## 1. System Update & Essential Packages

Always start by updating package indexes and upgrading installed packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget gnupg build-essential ufw htop unzip software-properties-common
```

---

## 2. Git & GitHub CLI Setup

### 2.1 Install Git

```bash
sudo apt install -y git
git --version
```

### 2.2 Install Official GitHub CLI (`gh`)

```bash
# Add official GitHub CLI keyring & repository
sudo mkdir -p -m 755 /etc/apt/keyrings
wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null

# Install gh
sudo apt update
sudo apt install -y gh
```

### 2.3 Authenticate GitHub

```bash
gh auth login
```

- Follow interactive prompts (GitHub.com -> HTTPS -> Yes -> Web or Token).

---

## 3. Node.js Environment (via NVM)

Installing Node via NVM avoids system package conflicts and allows easy version switching.

### 3.1 Install NVM

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash

# Load NVM into current session
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
```

### 3.2 Install LTS Node.js

```bash
# Install latest LTS version
nvm install --lts

# Set LTS as default
nvm alias default 'lts/*'
nvm use default

# Verify
node -v
npm -v
```

### 3.3 Install Global Package Managers

```bash
npm install -g pnpm pm2
```

---

## 4. MongoDB Community Edition Setup

Official installation of MongoDB Community Edition (Ubuntu 22.04 / 24.04 LTS).

### 4.1 Import MongoDB GPG Key & Repository

```bash
# Import GPG key
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor --yes

# Add repository
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Update & Install
sudo apt update
sudo apt install -y mongodb-org
```

### 4.2 Start, Enable & Check MongoDB

```bash
sudo systemctl start mongod
sudo systemctl enable mongod
sudo systemctl status mongod --no-pager
```

### 4.3 Test MongoDB Connection

```bash
mongosh --eval "db.adminCommand('ping')"
# Returns: { ok: 1 }
```

---

## 5. PostgreSQL Installation & Database Setup

### 5.1 Install PostgreSQL

```bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo systemctl status postgresql --no-pager
```

### 5.2 Create Database, User & Permissions

Replace `my_database`, `my_user`, and `MySecurePassword123!` with your desired values:

```bash
sudo -u postgres psql
```

Inside `psql` console, run:

```sql
-- 1. Create User
CREATE USER my_user WITH PASSWORD 'MySecurePassword123!';

-- 2. Create Database
CREATE DATABASE my_database OWNER my_user;

-- 3. Grant Database Permissions
GRANT ALL PRIVILEGES ON DATABASE my_database TO my_user;
ALTER USER my_user CREATEDB;

-- 4. Connect to database and grant schema privileges
\c my_database
GRANT ALL ON SCHEMA public TO my_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO my_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO my_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO my_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO my_user;

-- 5. Exit
\q
```

### 5.3 Test PostgreSQL Connection

```bash
psql -h 127.0.0.1 -p 5432 -U my_user -d my_database
```

---

## 6. Redis Server Installation & Configuration

### 6.1 Install Redis

```bash
sudo apt install -y redis-server
```

### 6.2 Configure Redis (systemd & optional password)

Edit `/etc/redis/redis.conf`:

```bash
# 1. Enable systemd supervision
sudo sed -i 's/^supervised no/supervised systemd/' /etc/redis/redis.conf

# 2. Bind to localhost only (security)
sudo sed -i 's/^bind .*/bind 127.0.0.1 ::1/' /etc/redis/redis.conf

# 3. (Optional) Set a password (replace MyRedisPassword123 with yours)
echo "requirepass MyRedisPassword123" | sudo tee -a /etc/redis/redis.conf

# 4. Restart and enable
sudo systemctl restart redis-server
sudo systemctl enable redis-server
```

### 6.3 Test Redis

```bash
# Without password:
redis-cli ping
# Returns: PONG

# With password:
redis-cli -a "MyRedisPassword123" ping
# Returns: PONG
```

---

## 7. PM2 Process Manager & Auto-Boot Persistence

PM2 manages Node.js applications, restarts them on crash, and starts them automatically on VPS reboot.

### 7.1 Start an Application

```bash
# Start an app (from project folder)
pm2 start dist/main.js --name "my-app"
# Or for npm/pnpm start script:
pm2 start "pnpm start" --name "my-app"
```

### 7.2 Configure Auto-Start on System Boot

```bash
# Generate startup script
pm2 startup
# Copy and run the sudo command printed on your screen

# Save the current list of running apps
pm2 save
```

### 7.3 Setup Log Rotation (Prevents Server Disk Full)

```bash
pm2 install pm2-logrotate
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 14
pm2 set pm2-logrotate:compress true
```

### 7.4 Common PM2 Commands

```bash
pm2 status              # View all running apps
pm2 logs [app-name]     # View live logs
pm2 restart [app-name]  # Restart app
pm2 stop [app-name]     # Stop app
pm2 delete [app-name]   # Remove app from PM2
pm2 monit               # Terminal dashboard for CPU/RAM
```

---

## 8. Recommended: 4GB Swap Setup

Swap memory prevents unexpected process kills (`OOM Killer`) when compiling code or running databases on smaller VPS instances ($\le$ 4GB RAM).

```bash
# 1. Create 4GB swap file
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 2. Make permanent across reboots
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# 3. Optimize swappiness (10 is ideal for production)
sudo sysctl vm.swappiness=10
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf

# 4. Verify
free -h
```

---

## 9. Recommended: UFW Firewall Configuration

Lock down your server so only Web ports (80, 443) and SSH (22) are public. Databases (5432, 6379, 27017) stay local.

```bash
# 1. Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 2. Allow SSH (Do NOT skip this before enabling!)
sudo ufw allow 22/tcp

# 3. Allow HTTP & HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 4. Enable Firewall
sudo ufw --force enable

# 5. Check Status
sudo ufw status verbose
```

---

## 10. Quick Verification Cheat Sheet

Check all services at once:

```bash
# Check service status
sudo systemctl status postgresql redis-server mongod --no-pager

# Check ports (Local bindings)
sudo ss -tulpn | grep -E '5432|6379|27017'

# Check Node & PM2
node -v
npm -v
pm2 status
```
