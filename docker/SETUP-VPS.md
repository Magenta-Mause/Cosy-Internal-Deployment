# VPS Setup Guide for Cosy Deployment

This guide walks you through setting up a fresh VPS for deploying Cosy with nginx, SSL, and Docker.

## Overview

- **One-time Setup**: Use Ansible to configure the VPS (security, nginx, SSL, Docker)
- **Automated Deployment**: GitHub Actions handles all deployments via SSH (no Ansible needed)

## Prerequisites

- A fresh Ubuntu/Debian VPS (20.04 or newer)
- Root SSH access to the VPS
- A domain name pointed to your VPS IP
- Ansible installed locally (for initial setup only): `sudo apt install ansible`

## Part 1: Initial VPS Setup (One-Time, Uses Ansible)

This section uses Ansible to prepare your VPS. You only need to do this once per VPS.

### 1. Generate SSH Key for Deployment

```bash
# On your local machine
ssh-keygen -t ed25519 -f ~/.ssh/cosy_deploy -C "cosy-deployment"
```

### 2. Configure Variables

Set these environment variables before running the setup:

```bash
export DEPLOY_USER_SSH_KEY="$(cat ~/.ssh/cosy_deploy.pub)"
export COSY_DOMAIN="cosy.pybay.com"
export LETSENCRYPT_EMAIL="your-email@example.com"
export DEPLOY_USER_PASSWORD="your-secure-password-here"  # Optional, will prompt to change
```

### 3. Update Inventory

Edit `docker/ansible/inventory-setup.yml`:
```yaml
ansible_host: 80.158.76.109  # Replace with your VPS IP
```

### 4. Run Setup Playbook

```bash
cd docker/ansible
ansible-playbook -i inventory-setup.yml setup-vps.yml --ask-pass --ask-become-pass
```

You'll be prompted for:
- SSH password (root password)
- Sudo password (same as SSH password for first run)

### 5. Test New User Access

```bash
ssh -i ~/.ssh/cosy_deploy deploy@YOUR_VPS_IP
```

### 6. Configure GitHub Secrets

Add these secrets to your GitHub repository (Settings → Secrets and variables → Actions):

```bash
# Display private key
cat ~/.ssh/cosy_deploy
```

Required GitHub Secrets:
- `VPS_SSH_KEY`: The private key content (output from above command)
- `VPS_HOST`: Your VPS IP address
- `DEPLOY_USER`: The deployment username (default: `deploy`)

## What the Initial Setup Does

✅ **System Updates**
- Updates all packages
- Enables automatic security updates

✅ **User Management**
- Creates `deploy` user with sudo access
- Adds your SSH key
- Disables root login
- Disables password authentication

✅ **Docker Installation**
- Installs Docker Engine & Docker Compose
- Adds deploy user to docker group

✅ **Firewall Configuration**
- Enables UFW firewall
- Opens ports: 22 (SSH), 80 (HTTP), 443 (HTTPS)
- Denies all other incoming traffic

✅ **Security Hardening**
- Configures fail2ban for SSH protection
- Sets up automatic security updates
- Hardens SSH configuration

✅ **Nginx & SSL**
- Installs nginx
- Configures reverse proxy for Cosy
- Obtains Let's Encrypt SSL certificate
- Sets up auto-renewal for SSL

✅ **Application Setup**
- Creates `/opt/cosy` directory
- Ready for Docker Compose deployment

## Part 2: Automated Deployment (GitHub Actions)

After the initial VPS setup, all deployments are handled automatically by GitHub Actions - **no Ansible required**.

### How It Works

1. Push changes to `main` branch (or trigger manually via workflow_dispatch)
2. GitHub Actions workflow runs automatically:
   - Connects to VPS via SSH
   - Copies Docker Compose files
   - Logs into GitHub Container Registry
   - Pulls latest Docker images
   - Restarts services with `docker compose up -d`
   - Cleans up old images

### Triggering a Deployment

**Automatic**: Push to `main` branch
```bash
git push origin main
```

**Manual**: Use GitHub UI
- Go to Actions tab → "Deploy to VPS" → Run workflow

### Monitoring Deployment

View deployment logs in GitHub Actions:
- Go to your repository → Actions tab
- Click on the latest "Deploy to VPS" workflow run

## File Structure

```
.github/workflows/
├── deploy-vps.yml              # Automated deployment (pure GitHub Actions)
└── update-manifests.yml        # Kubernetes manifest updates

docker/
├── docker-compose.yml          # Docker Compose configuration
├── nginx-cosy.conf             # Reference nginx config
├── SETUP-VPS.md                # This file
└── ansible/                    # One-time setup automation
    ├── setup-vps.yml           # VPS setup playbook
    ├── inventory-setup.yml     # Inventory for initial setup
    └── templates/
        └── nginx-cosy.conf.j2  # Nginx configuration template
```

## Nginx Configuration

The setup creates an nginx reverse proxy that:
- Redirects HTTP → HTTPS
- Proxies `/` → Frontend (localhost:3000)
- Proxies `/api` → Backend (localhost:8080)
- Proxies `/ws` → Backend WebSocket (if needed)
- Includes rate limiting and security headers
- Has SSL certificates auto-renewed

## Docker Compose Changes

Updated to bind services to localhost only (security):
- Frontend: `127.0.0.1:3000:80` (only accessible via nginx)
- Backend: `127.0.0.1:8080:8080` (only accessible via nginx)
- Database: Internal to Docker network

## Troubleshooting

### SSH Connection Issues
```bash
# Check if SSH service is running on VPS
ssh root@YOUR_VPS_IP "systemctl status sshd"

# Test with verbose output
ssh -vvv -i ~/.ssh/cosy_deploy deploy@YOUR_VPS_IP
```

### SSL Certificate Issues
```bash
# SSH into VPS and check certbot
ssh -i ~/.ssh/cosy_deploy deploy@YOUR_VPS_IP
sudo certbot certificates
sudo certbot renew --dry-run
```

### Nginx Issues
```bash
# Check nginx status
sudo systemctl status nginx
sudo nginx -t  # Test configuration

# View logs
sudo tail -f /var/log/nginx/cosy_error.log
```

### Docker Issues
```bash
# Check Docker status
sudo systemctl status docker
docker ps  # List running containers
docker compose logs -f  # View application logs
```

## Manual Deployment (Emergency/Testing)

If you need to deploy manually (bypassing GitHub Actions):

```bash
# SSH into VPS
ssh -i ~/.ssh/cosy_deploy deploy@YOUR_VPS_IP

# Navigate to deployment directory
cd /opt/cosy

# Log in to GitHub Container Registry (if needed)
echo "YOUR_GITHUB_TOKEN" | docker login ghcr.io -u YOUR_USERNAME --password-stdin

# Pull latest images
docker compose pull

# Restart services
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f
```

## Key Differences: Setup vs Deployment

| Aspect | Initial Setup (Ansible) | Deployment (GitHub Actions) |
|--------|------------------------|----------------------------|
| **When** | Once per VPS | Every code update |
| **Tool** | Ansible playbook | GitHub Actions SSH |
| **What** | Install Docker, nginx, SSL, security | Pull images, restart containers |
| **Requires** | Ansible installed locally | Just GitHub Secrets configured |
| **Trigger** | Manual command | Automatic on push or manual |

## Security Notes

- SSH key authentication only (passwords disabled)
- Firewall blocks all non-essential ports
- fail2ban protects against brute force
- Automatic security updates enabled
- SSL with modern TLS configuration
- Services bound to localhost (not exposed directly)

## Next Steps

After setup is complete:
1. Test accessing your domain: `https://cosy.yourdomain.com`
2. Deploy application: Trigger GitHub Actions workflow
3. Monitor logs: `ssh deploy@VPS "cd /opt/cosy && docker compose logs -f"`
