# Google Workspace MCP Server — Self-Hosting Guide (Debian Desktop)

A step-by-step guide to self-hosting the [Google Workspace MCP server](https://github.com/taylorwilsdon/google_workspace_mcp) in a Docker container on a Debian 13 VM running on Proxmox. This version uses a minimal desktop environment so that OAuth flows and Claude Code authentication can be completed directly on the VM through a browser, avoiding the headless authentication issues that plague server-only installs.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Debian 13 VM Setup on Proxmox](#debian-13-vm-setup-on-proxmox)
4. [Install Docker on Debian](#install-docker-on-debian)
5. [Google Cloud OAuth Setup](#google-cloud-oauth-setup)
6. [Securing MCP Access to Your Google Account](#securing-mcp-access-to-your-google-account)
7. [Deploy the MCP Server](#deploy-the-mcp-server)
8. [First-Time OAuth Authentication](#first-time-oauth-authentication)
9. [Connect Your MCP Client](#connect-your-mcp-client)
10. [Managing the Server](#managing-the-server)
11. [Running Claude Code on the VM with SSHFS](#running-claude-code-on-the-vm-with-sshfs)
12. [Troubleshooting](#troubleshooting)

---

## Overview

### What This Stack Looks Like

```
Your Machine (Claude Code / VS Code)
        |
        | HTTP (port 8000)
        v
Proxmox Host
  └── Debian 13 VM (Xfce desktop)
        ├── Firefox (for OAuth + Claude Code login)
        └── Docker
              └── google_workspace_mcp container
                    ├── streamable-http transport on :8000
                    ├── OAuth tokens persisted in a Docker volume
                    └── client_secret.json mounted read-only
```

The MCP server runs as a Docker container inside the Debian VM. Your MCP clients connect to it over HTTP. OAuth tokens are stored in a Docker volume so they survive container restarts.

### Why a Desktop Instead of Headless

Both Google OAuth and Claude Code's login flow expect a browser. On a headless server, you end up fighting X11 forwarding, SSH port tunnelling, and clipboard gymnastics. A lightweight desktop adds roughly 500 MB of disk and minimal RAM overhead, but makes every browser-based auth flow work out of the box through the Proxmox noVNC console. Once initial setup is done, you can manage the VM entirely over SSH — the desktop just sits idle.

---

## Prerequisites

Before starting, you should have:

- A Proxmox host with enough resources to allocate a small VM (2 CPU cores, 2–4 GB RAM, 20 GB disk)
- A Debian 13 (Trixie) ISO — download the netinst image from [debian.org/distrib/netinst](https://www.debian.org/distrib/netinst)
- A Google account with access to Google Workspace (Gmail, Drive, Calendar, etc.)
- A Google Cloud project (free tier is fine — you will not be billed for OAuth credentials)

---

## Debian 13 VM Setup on Proxmox

If you already have a Debian 13 VM with a desktop environment running, skip to [Install Docker on Debian](#install-docker-on-debian).

### Step 1: Upload the Debian ISO

1. In the Proxmox web UI, go to your storage (e.g., `local`) and click **ISO Images**
2. Click **Upload** and select your Debian 13 netinst ISO

### Step 2: Create the VM

1. Click **Create VM** in the top right
2. Configure with these recommended settings:

| Setting | Value |
|---------|-------|
| **OS** | Select your uploaded Debian 13 ISO |
| **System** | Default (SeaBIOS or OVMF both work) |
| **Disk** | 20 GB (virtio-scsi) |
| **CPU** | 2 cores |
| **Memory** | 2048–4096 MB (2 GB minimum, 4 GB comfortable with desktop) |
| **Network** | vmbr0 (bridged), virtio NIC |

3. Start the VM and open the console to run through the Debian installer

### Step 3: Debian Installer Walkthrough

The Debian netinst installer is text-based by default. Follow these key steps:

1. Select **Install** (not Graphical Install — the text installer is faster in noVNC)
2. Choose your language, location, and keyboard layout
3. Set a hostname (e.g., `mcp-server`)
4. Set a root password when prompted, then create your regular user account
5. When you reach **partitioning**, select **Guided — use entire disk** and accept the defaults

### Step 4: Network Configuration

During installation, the installer will attempt DHCP. Let it complete, then configure a static IP after install (see Step 6). Alternatively, if you want to set a static IP during install:

1. When the installer asks to auto-configure networking, let DHCP succeed first
2. You will set the static IP after installation — it is easier to do post-install on Debian

### Step 5: Software Selection

When the installer reaches the **Software selection** (tasksel) screen, this is where you pick the desktop environment:

1. **Deselect** "Debian desktop environment" and "GNOME" (if selected by default)
2. **Select** "Xfce" — a lightweight, polished desktop that uses around 300–400 MB RAM at idle
3. Keep **SSH server** selected
4. Keep **standard system utilities** selected

The selection should look like:

```
[ ] Debian desktop environment
[ ] GNOME
[*] Xfce
[ ] MATE
[ ] Cinnamon
[ ] LXQt
[ ] KDE Plasma
[ ] LXDE
[ ] Budgie
[*] SSH server
[*] standard system utilities
```

**Why Xfce:** It is the most polished lightweight desktop available. GTK3-based, actively maintained, and widely used. It includes a file manager (Thunar), terminal emulator, and a clean panel layout out of the box.

The installer will download and install the selected packages. This takes a few minutes depending on your internet speed.

### Step 6: Post-Install — Set a Static IP

Once the VM boots into the Xfce desktop, open a terminal (Applications > System > Xfce Terminal) and configure a static IP.

Debian uses `/etc/network/interfaces` for network configuration:

```bash
sudo nano /etc/network/interfaces
```

Replace the DHCP line for your interface (usually `ens18`) with a static configuration:

```
# The primary network interface
auto ens18
iface ens18 inet static
    address 192.168.1.50
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

**How to find your subnet and gateway:** Check your Proxmox host, since it is already on the same network. From the Proxmox shell:

```bash
# Shows your subnet (look for the "inet" line, e.g. inet 192.168.1.100/24)
ip addr show vmbr0

# Shows your gateway (e.g. default via 192.168.1.1)
ip route | grep default
```

**Choosing the static IP:** Pick an address in your subnet that is not already in use and is outside your router's DHCP range. Most routers hand out DHCP leases in a range like `192.168.1.100-254`, so something low like `192.168.1.50` is usually safe.

Apply the new network config:

```bash
sudo systemctl restart networking
```

Verify connectivity:

```bash
ip addr show ens18
ping -c 3 google.com
```

### Step 7: Install a Browser

The Xfce desktop does not include a full browser by default. Install Firefox ESR:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y firefox-esr
```

### Step 8: Verify SSH Access

From your local machine, confirm you can SSH in:

```bash
ssh your-username@<vm-ip>
```

From this point on, day-to-day management can happen over SSH. The desktop is there for when you need a browser (OAuth flows, Claude Code login).

---

## Install Docker on Debian

Docker is how the MCP server runs. Install Docker Engine using the official apt repository.

### Step 1: Add Docker's GPG Key and Repository

```bash
# Install prerequisites
sudo apt install -y ca-certificates curl

# Add Docker's official GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 2: Install Docker Engine and Compose

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Step 3: Allow Your User to Run Docker Without `sudo`

```bash
sudo usermod -aG docker $USER
```

**Important:** Log out and back in (or run `newgrp docker`) for the group change to take effect.

### Step 4: Verify Docker Works

```bash
docker run hello-world
```

You should see "Hello from Docker!" confirming everything is installed correctly.

---

## Google Cloud OAuth Setup

The MCP server needs OAuth credentials to access Google APIs on your behalf. This section creates those credentials.

### Step 1: Create a Google Cloud Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com/)
2. Click the project dropdown at the top and select **New Project**
3. Give it a name (e.g., "Workspace MCP") and click **Create**
4. Make sure the new project is selected in the dropdown

### Step 2: Enable the APIs You Need

Go to **APIs & Services > Library** and enable each API you plan to use:

| API | Required For |
|-----|-------------|
| **Gmail API** | Reading/sending email |
| **Google Drive API** | File access and search |
| **Google Calendar API** | Calendar events |
| **Google Docs API** | Document reading/editing |
| **Google Sheets API** | Spreadsheet access |
| **Google Slides API** | Presentation access |
| **Google Tasks API** | Task management |
| **People API** | Contacts |

You only need to enable the ones you plan to use. Gmail, Drive, and Calendar are the most common.

### Step 3: Configure the OAuth Consent Screen

1. Go to **APIs & Services > OAuth consent screen**
2. Select **External** as the user type (unless you have a Google Workspace org, in which case choose **Internal**)
3. Fill in the required fields:
   - **App name:** Workspace MCP (or whatever you like)
   - **User support email:** Your email
   - **Developer contact email:** Your email
4. On the **Scopes** page, click **Add or Remove Scopes** and add the scopes matching the APIs you enabled (e.g., Gmail read/send, Drive file access, etc.) — or skip this and the server will request them at runtime
5. On the **Test users** page, add your Google email address — this is required while the app is in "Testing" status
6. Click **Save and Continue** through to the end

**Key point:** While your app is in "Testing" status, only the test users you add can authenticate. This is fine for personal self-hosting — you do not need to publish the app.

### Step 4: Create OAuth 2.0 Credentials

1. Go to **APIs & Services > Credentials**
2. Click **Create Credentials > OAuth client ID**
3. Application type: **Desktop application**
4. Name: anything you like (e.g., "MCP Desktop")
5. Click **Create**
6. Click **Download JSON** — this is your `client_secret.json` file

Save this file somewhere safe. You will copy it to the server in the next section.

---

## Securing MCP Access to Your Google Account

By default, the MCP server requests broad permissions across all Google Workspace services — Gmail read/write/send, full Drive access, Calendar, Contacts, Docs, Sheets, Slides, Tasks, Chat, Forms, and Apps Script. Before completing the OAuth flow, take a moment to decide what you actually need and what you are comfortable granting.

### Principle of Least Privilege

Only grant the permissions the MCP server actually needs for your use case. You can control access at three levels:

**Level 1: Limit tools in the `.env` file (server-side)**

The `TOOLS` variable controls which Google services the MCP server loads. Only the scopes for the listed services will be requested during OAuth:

```env
# Only load Gmail and Calendar tools
TOOLS=gmail calendar
```

Available tool names: `gmail`, `drive`, `calendar`, `docs`, `sheets`, `slides`, `tasks`, `contacts`, `chat`, `forms`, `search`, `appscript`

Set this **before** completing the OAuth flow so you are only asked to grant relevant permissions. Without this variable, the server defaults to loading **all** tools and will request permissions for every Google service — regardless of which APIs you enabled in the Google Cloud Console. The Google Cloud API settings only control whether API calls succeed, not what the OAuth consent screen displays. The `TOOLS` variable is the only way to limit what permissions are requested.

**Level 2: Deselect permissions on the Google consent screen (account-side)**

When you complete the OAuth flow in the browser, Google shows a consent screen listing every permission the app is requesting. You can **deselect individual permissions** you do not want to grant. For example, if you only want read access to email, you can deselect Gmail send/compose/modify permissions and only approve read-only access.

This is the most granular control you have, and it happens at the Google account level — even if the server requests broad scopes, you decide what to approve.

**Level 3: Use a separate Google account (strongest isolation)**

For maximum safety, create a dedicated Google account just for MCP access. Share only specific calendars, documents, or Drive folders with it from your primary account. This way, even if all permissions are granted, the MCP server can only see what you have explicitly shared — your primary account's data is never directly accessible.

### What Each Permission Means

The Google consent screen presents every permission individually with a checkbox. Here is the full list the MCP server requests, grouped by service:

**Gmail**

| Permission | Risk |
|-----------|------|
| View your email messages and settings | Low |
| Send email on your behalf | **High** — an LLM could send emails as you |
| See and edit your email labels | Low |
| See, edit, create, or change your email settings and filters in Gmail | Medium |
| Manage drafts and send emails | **High** |
| Read, compose, and send emails from your Gmail account | **High** |

**Google Drive**

| Permission | Risk |
|-----------|------|
| See and download all your Google Drive files | Low |
| See, edit, create, and delete all of your Google Drive files | **High** |
| See, edit, create, and delete only the specific Google Drive files you use with this app | Medium — scoped to files the app touches |

**Google Chat**

| Permission | Risk |
|-----------|------|
| View chat and spaces in Google Chat | Low |
| See messages as well as their reactions and message content in Google Chat | Low |
| Create conversations and spaces and see or update metadata (including history settings and access settings) in Google Chat | **High** |
| See, compose, send, update, and delete messages as well as their message content; add, see, and delete reactions to messages | **High** |

**Contacts**

| Permission | Risk |
|-----------|------|
| See and download your contacts | Low |
| See, edit, download, and permanently delete your contacts | **High** |

**Google Slides**

| Permission | Risk |
|-----------|------|
| See all your Google Slides presentations | Low |
| See, edit, create, and delete all your Google Slides presentations | **High** |

**Google Docs**

| Permission | Risk |
|-----------|------|
| See all your Google Docs documents | Low |
| See, edit, create, and delete all your Google Docs documents | **High** |

**Google Sheets**

| Permission | Risk |
|-----------|------|
| See all your Google Sheets spreadsheets | Low |
| See, edit, create, and delete all your Google Sheets spreadsheets | **High** |

**Google Calendar**

| Permission | Risk |
|-----------|------|
| See and download any calendar you can access using your Google Calendar | Low |
| View and edit events on all your calendars | Medium |
| See, edit, share, and permanently delete all the calendars you can access using your Google Calendar | **High** |

**Google Forms**

| Permission | Risk |
|-----------|------|
| See all responses to your Google Forms forms | Low |
| See all your Google Forms forms | Low |
| See, edit, create, and delete all your Google Forms forms | **High** |

**Google Tasks**

| Permission | Risk |
|-----------|------|
| View your tasks | Low |
| Create, edit, organize, and delete all your tasks | Medium |

**Google Apps Script**

| Permission | Risk |
|-----------|------|
| View Google Apps Script processes | Low |
| View Google Apps Script project's metrics | Low |
| View Google Apps Script deployments | Low |
| View Google Apps Script projects | Low |
| Create and update Google Apps Script deployments | **High** |
| Create and update Google Apps Script projects | **High** |

**Other**

| Permission | Risk |
|-----------|------|
| View your Custom Search Engine results and metadata | Low |

### Recommended Starting Configuration

Start with the minimum you need and expand later:

```env
# Read-only email is the safest starting point
TOOLS=gmail
```

On the Google consent screen, deselect everything except **Gmail read-only** and basic profile/email access. You can always revoke and re-grant permissions later.

### Revoking Access

If you want to revoke the MCP server's access to your Google account:

1. Go to [myaccount.google.com/permissions](https://myaccount.google.com/permissions)
2. Find "Workspace MCP" (or whatever you named your app)
3. Click **Remove Access**

On the server side, remove the stored tokens:

```bash
cd ~/google_workspace_mcp
docker compose down
docker volume rm google_workspace_mcp_store_creds
```

### Protecting Your Credentials

- **`client_secret.json`** is mounted read-only into the container. Do not commit it to version control or share it.
- **OAuth tokens** are stored in the `store_creds` Docker volume. Anyone with access to this volume can access your Google account without re-authenticating.
- **The `.env` file** contains your client secret in plain text. Restrict file permissions:
  ```bash
  chmod 600 ~/google_workspace_mcp/.env
  chmod 600 ~/google_workspace_mcp/client_secret.json
  ```
- **Network access** — the MCP server listens on port 8000 with no authentication. Anyone on your local network can use it. If this is a concern, restrict access with a firewall rule:
  ```bash
  # Only allow connections from a specific IP (e.g., your workstation)
  sudo iptables -A INPUT -p tcp --dport 8000 -s 192.168.1.100 -j ACCEPT
  sudo iptables -A INPUT -p tcp --dport 8000 -j DROP
  ```

---

## Deploy the MCP Server

### Step 1: Clone the Repository

SSH into your Debian VM and clone the repo:

```bash
cd ~
git clone https://github.com/taylorwilsdon/google_workspace_mcp.git
cd google_workspace_mcp
```

### Step 2: Add Your `client_secret.json`

Copy the `client_secret.json` you downloaded from Google Cloud onto the server. From your local machine:

```bash
scp /path/to/client_secret.json your-username@<vm-ip>:~/google_workspace_mcp/client_secret.json
```

Or paste the contents directly on the server:

```bash
nano ~/google_workspace_mcp/client_secret.json
# Paste the JSON contents, save with Ctrl+O, exit with Ctrl+X
```

### Step 3: Create the `.env` File

Create a `.env` file in the repo root with your configuration:

```bash
nano ~/google_workspace_mcp/.env
```

Add the following (replace the placeholder values with the client ID and secret from your `client_secret.json`):

```env
# Required — these must match the values in client_secret.json
GOOGLE_OAUTH_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_OAUTH_CLIENT_SECRET=your-client-secret

# Required for non-HTTPS OAuth flow (since this is a local/private server)
OAUTHLIB_INSECURE_TRANSPORT=1

# Required — used by the start_google_auth tool to identify which account to authenticate
USER_GOOGLE_EMAIL=you@gmail.com
```

**Where to find the client ID and secret:** Open `client_secret.json` and look for the `client_id` and `client_secret` fields inside the `installed` object.

### Step 4: Build and Start the Container

```bash
docker compose up --build -d
```

This will:
- Build the Docker image from the repo's `Dockerfile`
- Start the container in detached mode (runs in the background)
- Expose port 8000 on the VM
- Create a `store_creds` Docker volume that persists OAuth tokens across restarts
- Mount `client_secret.json` read-only into the container

### Step 5: Verify It's Running

```bash
# Check the container is up
docker compose ps

# Check the logs
docker compose logs -f
```

You should see the server starting up on port 8000. Press `Ctrl+C` to stop following the logs.

---

## First-Time OAuth Authentication

On the first run, you need to complete the Google OAuth flow so the server gets permission to access your Google account. Since you have a desktop environment with Firefox, this is straightforward.

### Step 1: Trigger the OAuth Flow

The server does not prompt for OAuth on startup. Instead, the OAuth flow is triggered when a client first makes a request that requires Google API access. To trigger it, either:

- **Option A:** Connect your MCP client (see [Connect Your MCP Client](#connect-your-mcp-client)) and ask it to perform a Google action (e.g., "list my recent emails"). The server logs will then show the OAuth authorization URL.
- **Option B:** Use `curl` to hit the MCP endpoint directly, which will also trigger the auth flow.

Watch the logs to see the OAuth URL appear:

```bash
cd ~/google_workspace_mcp
docker compose logs -f
```

When the OAuth flow is triggered, the logs will show a URL like:

```
Please visit this URL to authorize this application:
https://accounts.google.com/o/oauth2/auth?client_id=...
```

### Step 2: Complete the OAuth Flow

1. Copy the OAuth URL and open it in Firefox on the VM desktop (through the Proxmox noVNC console)
2. Sign in with the Google account you added as a test user
3. You may see a "Google hasn't verified this app" warning — click **Advanced > Go to [app name] (unsafe)**. This is expected for apps in testing mode.
4. Grant the requested permissions
5. The redirect to `localhost` will work because the browser is running on the same machine as the server

Since the browser and server are both on the VM, the `localhost` redirect that causes problems in headless setups works correctly here.

**Tip — noVNC clipboard:** The Proxmox noVNC console does not support universal clipboard sharing, so you cannot paste the OAuth URL directly from your local machine. Instead, save the URL to a file on the VM and open it from there:

```bash
# If the URL is already in a file (e.g. ~/oauth_url.txt):
firefox "$(cat ~/oauth_url.txt)"

# Or pipe it from the Docker logs:
docker compose logs | grep -o 'https://accounts.google.com/o/oauth2/auth[^ ]*' > ~/oauth_url.txt
firefox "$(cat ~/oauth_url.txt)"
```

### Step 3: Verify Authentication

Once the OAuth flow completes, the server will store the tokens in the `store_creds` Docker volume. Check the logs:

```bash
docker compose logs --tail 20
```

You should see the server running normally without any auth prompts. The tokens will auto-refresh, so you should not need to re-authenticate unless you revoke access or the refresh token expires (which is rare).

---

## Connect Your MCP Client

The server is now running and authenticated. Point your MCP client at it.

### Claude Code

Use the `claude mcp add` CLI command to register the server. **Do not manually edit `~/.claude.json`** — it contains auto-generated state that is easy to corrupt.

**User-wide configuration (recommended for Google Workspace):**

This makes the MCP server available in every Claude Code session, regardless of which directory you start it from:

```bash
claude mcp add --transport http --scope user google-workspace http://<vm-ip>:8000/mcp
```

If Claude Code is running on the same VM as the MCP server, use `http://localhost:8000/mcp`. If connecting from a different machine, replace `<vm-ip>` with the VM's IP address on the local network.

**Project-level configuration:**

This makes the MCP server available only when Claude Code is started from the current project directory:

```bash
claude mcp add --transport http --scope project google-workspace http://<vm-ip>:8000/mcp
```

This creates a `.mcp.json` file in the project root that can be checked into version control.

**Managing MCP servers:**

```bash
# List all configured servers
claude mcp list

# Get details for a specific server
claude mcp get google-workspace

# Remove a server
claude mcp remove google-workspace
```

### VS Code

Add to your VS Code MCP settings (`.vscode/mcp.json` or user settings):

```json
{
  "servers": {
    "google-workspace": {
      "type": "http",
      "url": "http://<vm-ip>:8000/mcp"
    }
  }
}
```

### Verify the Connection

After updating the configuration, **restart Claude Code** for the changes to take effect. You can test the connection by asking Claude to search your recent emails. If the Google Workspace tools show up in the available tools list, everything is working.

---

## Managing the Server

### Common Docker Compose Commands

All commands should be run from the `~/google_workspace_mcp` directory.

```bash
# Start the server (detached)
docker compose up -d

# Stop the server
docker compose down

# Restart the server
docker compose restart

# View live logs
docker compose logs -f

# Rebuild after pulling repo updates
docker compose up --build -d

# Check container health
docker compose ps
```

### Updating the MCP Server

When the upstream repo releases updates:

```bash
cd ~/google_workspace_mcp
git pull origin main
docker compose up --build -d
```

Your OAuth tokens are safe in the `store_creds` Docker volume and will not be affected by rebuilds.

### Limiting Available Tools

If you don't want all Google Workspace tools exposed, you can control which are loaded.

**Option 1: Specific tools only** — add to your `.env` file:

```env
TOOLS=gmail drive calendar
```

**Option 2: Use a tier** — add to your `.env` file:

```env
TOOL_TIER=core
```

After changing `.env`, restart the container:

```bash
docker compose up -d
```

### Auto-Start on Boot

Docker containers with `restart: unless-stopped` will auto-start when Docker starts. Add the restart policy to `docker-compose.yml`:

```yaml
services:
  gws_mcp:
    build: .
    container_name: gws_mcp
    restart: unless-stopped    # <-- add this line
    ports:
      - "8000:8000"
    environment:
      - GOOGLE_MCP_CREDENTIALS_DIR=/app/store_creds
    volumes:
      - ./client_secret.json:/app/client_secret.json:ro
      - store_creds:/app/store_creds:rw
    env_file:
      - .env

volumes:
  store_creds:
```

Docker itself is enabled to start on boot by default on Debian. Verify with:

```bash
sudo systemctl is-enabled docker
# Should output: enabled
```

---

## Running Claude Code on the VM with SSHFS

You can install Claude Code directly on the Debian VM and use SSHFS to mount your local project files. This lets Claude Code run Docker commands on the server while also editing docs or config files that live on your local machine. Since the VM has a desktop, the Claude Code login flow works without any workarounds.

### Step 1: Install Node.js

Claude Code requires Node.js 18+.

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs
```

### Step 2: Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
```

### Step 3: Authenticate Claude Code

Open Xfce Terminal on the VM desktop (through the Proxmox noVNC console) and run:

```bash
claude
```

Claude Code will open Firefox for OAuth login. Complete the login in the browser, and Claude Code will be authenticated. Once authenticated, you can continue using Claude Code over SSH for all future sessions — the desktop is only needed for this initial login.

**Alternative: Use an API Key**

If you prefer not to use the browser login at all, set an Anthropic API key:

```bash
export ANTHROPIC_API_KEY="sk-ant-api03-your-key-here"
```

Generate a key at [console.anthropic.com](https://console.anthropic.com) under **API Keys**. This uses your API credits rather than a Pro/Max subscription.

To make it persistent:

```bash
echo 'export ANTHROPIC_API_KEY="sk-ant-api03-your-key-here"' >> ~/.bashrc
source ~/.bashrc
```

### Step 4: Mount Your Local Repo with SSHFS

SSHFS lets the VM access files on your local machine over SSH as if they were local directories.

```bash
# Install SSHFS
sudo apt install -y sshfs

# Create a mount point
mkdir -p ~/Guides

# Mount your local Guides repo onto the VM
sshfs your-mac-username@<mac-ip>:/Users/your-mac-username/Desktop/CodeProjects/Guides ~/Guides
```

**Requirements for this to work:**
- Your Mac must have **Remote Login** enabled (**System Settings > General > Sharing > Remote Login**)
- The VM must be able to reach your Mac over the network (they should be on the same subnet)
- Use your Mac's local IP address (find it with `ifconfig en0` on the Mac)

Verify the mount:

```bash
ls ~/Guides
# You should see GIT_GUIDE.md, GOOGLE_WORKSPACE_MCP_GUIDE.md, etc.
```

### Step 5: Run Claude Code

```bash
cd ~/Guides
claude
```

Claude Code now has direct access to both:
- **Docker on the VM** — can build, start, and debug the MCP container
- **Your Guides repo** — can edit docs in real time via the SSHFS mount

### Unmounting

When you're done, unmount cleanly:

```bash
fusermount -u ~/Guides
```

### Making the Mount Persistent (Optional)

To auto-mount on boot, add an entry to `/etc/fstab`:

```
your-mac-username@<mac-ip>:/Users/your-mac-username/Desktop/CodeProjects/Guides /home/your-vm-username/Guides fuse.sshfs defaults,_netdev,IdentityFile=/home/your-vm-username/.ssh/id_rsa,allow_other,reconnect 0 0
```

This requires SSH key authentication (no password prompts). Set that up with:

```bash
ssh-keygen -t ed25519    # if you don't already have a key
ssh-copy-id your-mac-username@<mac-ip>
```

---

## Troubleshooting

### Docker build fails with DNS errors

Debian uses `systemd-resolved` which can cause DNS resolution failures inside Docker builds.

**Step 1: Configure Docker's DNS**

```bash
sudo nano /etc/docker/daemon.json
```

Add:

```json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

Then restart Docker:

```bash
sudo systemctl restart docker
```

**Step 2: Clear the build cache and rebuild**

```bash
docker builder prune -f
docker compose build --no-cache
docker compose up -d
```

**If that still fails**, disable BuildKit to use the legacy builder which respects `daemon.json`:

```bash
DOCKER_BUILDKIT=0 docker compose build --no-cache
docker compose up -d
```

### Docker repository not available for Debian Trixie

**Note (as of March 2026):** Docker's apt repository now includes `trixie` as a supported release, so the standard install instructions work without modification. If you are following this guide on a future Debian release that is not yet supported, you can fall back to the most recent supported codename. For example, to use `bookworm` (Debian 12) instead:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  bookworm stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

The Docker packages are generally compatible across Debian releases.

### Container won't start

```bash
# Check logs for error messages
docker compose logs

# Common causes:
# - Missing client_secret.json (check it's in the repo root)
# - Malformed .env file (no quotes around values with special characters)
# - Port 8000 already in use (check with: sudo lsof -i :8000)
```

### "Permission denied" errors in logs

The container runs as a non-root user (`app`). Make sure the `store_creds` volume has correct permissions:

```bash
# This is usually handled automatically, but if needed:
docker compose down
docker volume rm google_workspace_mcp_store_creds
docker compose up --build -d
```

**Warning:** Removing the volume deletes stored OAuth tokens. You will need to re-authenticate.

### OAuth flow redirect fails

Since the browser runs on the same machine as the Docker container, the `localhost` redirect should work. If it still fails:

1. Check that the container is exposing port 8000 correctly: `docker compose ps`
2. Try accessing `http://localhost:8000/health` from Firefox on the VM
3. If using a non-standard port, make sure the redirect URI in your Google Cloud credentials matches

### Xfce desktop not starting

If the VM boots to a text login instead of the desktop:

```bash
# Check if the display manager is running
sudo systemctl status lightdm

# If it's not running, start and enable it
sudo systemctl enable --now lightdm
```

If `lightdm` is not installed (can happen with minimal installs):

```bash
sudo apt install -y lightdm
sudo systemctl enable --now lightdm
```

### MCP client can't connect

- Verify the container is running: `docker compose ps`
- Verify the port is accessible from your local machine: `curl http://<vm-ip>:8000/health`
- Check that no firewall is blocking port 8000:
  ```bash
  # On the Debian VM — check if nftables/iptables rules are blocking traffic
  sudo iptables -L -n

  # Debian does not enable a firewall by default, but if you installed ufw:
  sudo ufw status
  sudo ufw allow 8000/tcp
  ```
- Make sure the URL in your MCP config uses the VM's IP, not `localhost`

### Token refresh failures

If the server stops working after a long period and logs show auth errors:

```bash
# Remove stored tokens and re-authenticate
docker compose down
docker volume rm google_workspace_mcp_store_creds
docker compose up --build -d
# Then follow the OAuth flow again from the VM desktop
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Start the server | `docker compose up -d` |
| Stop the server | `docker compose down` |
| View logs | `docker compose logs -f` |
| Rebuild after update | `docker compose up --build -d` |
| Check container status | `docker compose ps` |
| Check health endpoint | `curl http://<vm-ip>:8000/health` |
| SSH into the VM | `ssh your-username@<vm-ip>` |
| Copy file to VM | `scp file.json your-username@<vm-ip>:~/google_workspace_mcp/` |
| Open VM desktop | Proxmox web UI > VM > Console (noVNC) |
| Mount local repo via SSHFS | `sshfs user@<mac-ip>:/path ~/mount-point` |
| Unmount SSHFS | `fusermount -u ~/mount-point` |