# VPS Setup Guide for Cosy Deployment

This guide walks you through setting up a fresh VPS for deploying Cosy with nginx, SSL, and Docker.

## Prerequisites

- A fresh Ubuntu/Debian VPS (20.04 or newer)
- Root SSH access to the VPS
- A domain name pointed to your VPS IP
- Ansible installed locally: `pip install ansible`

## Quick Setup

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

Edit `ansible/inventory-setup.yml`:
```yaml
ansible_host: 80.158.76.109  # Replace with your VPS IP
```

### 4. Run Setup Playbook

```bash
cd ansible
ansible-playbook -i inventory-setup.yml setup-vps.yml --ask-pass --ask-become-pass
```

You'll be prompted for:
- SSH password (root password)
- Sudo password (same as SSH password for first run)

### 5. Test New User Access

```bash
ssh -i ~/.ssh/cosy_deploy deploy@YOUR_VPS_IP
```

### 6. Add SSH Key to GitHub Secrets

```bash
# Display private key to add to GitHub
cat ~/.ssh/cosy_deploy

# Add to GitHub repo secrets as VPS_SSH_KEY
# Also add: VPS_HOST=YOUR_VPS_IP
```

### 7. Update Deployment Inventory

Edit `ansible/inventory.yml`:
```yaml
ansible_host: YOUR_VPS_IP  # Replace with your VPS IP
```

## What the Setup Does

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

## File Structure

```
ansible/
├── setup-vps.yml              # One-time VPS setup playbook
├── inventory-setup.yml         # Inventory for initial setup (uses root)
├── inventory.yml               # Inventory for deployments (uses deploy user)
├── deploy-docker.yml           # Automated deployment playbook
└── templates/
    └── nginx-cosy.conf.j2      # Nginx configuration template

docker/
├── docker-compose.yml          # Updated with localhost bindings
└── nginx-cosy.conf             # Reference nginx config
```

## Nginx Configuration

The setup creates an nginx reverse proxy that:
- Redirects HTTP → HTTPS
- Proxies `/` → Frontend (localhost:80)
- Proxies `/api` → Backend (localhost:8080)
- Proxies `/ws` → Backend WebSocket (if needed)
- Includes rate limiting and security headers
- Has SSL certificates auto-renewed

## Docker Compose Changes

Updated to bind services to localhost only (security):
- Frontend: `127.0.0.1:80:80` (only accessible via nginx)
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
docker-compose logs -f  # View application logs
```

## Next Steps

After setup is complete:
1. Test accessing your domain: `https://cosy.yourdomain.com`
2. Deploy application: Trigger GitHub Actions workflow
3. Monitor logs: `ssh deploy@VPS "cd /opt/cosy && docker-compose logs -f"`

## Security Notes

- SSH key authentication only (passwords disabled)
- Firewall blocks all non-essential ports
- fail2ban protects against brute force
- Automatic security updates enabled
- SSL with modern TLS configuration
- Services bound to localhost (not exposed directly)

## Manual Deployment (if needed)

```bash
# SSH into VPS
ssh -i ~/.ssh/cosy_deploy deploy@YOUR_VPS_IP

# Navigate to deployment directory
cd /opt/cosy

# Pull latest images
docker-compose pull

# Restart services
docker-compose up -d

# Check status
docker-compose ps
```
