# VPS Setup Guide for Cosy Deployment

This guide walks you through setting up a fresh VPS for deploying Cosy with nginx, SSL, and Docker.

## Overview

- **One-time Setup**: Use Ansible to configure the VPS (security, nginx, SSL, Docker, secrets).
- **Automated Deployment**: GitHub Actions handles all deployments via SSH.

## Prerequisites

- A fresh Ubuntu/Debian VPS (20.04 or newer)
- Root SSH access to the VPS
- A domain name pointed to your VPS IP
- Ansible installed locally (for initial setup only): `sudo apt install ansible`

## Part 1: Initial VPS Setup (One-Time, Uses Ansible)

This section uses Ansible to prepare your VPS. You only need to do this once per VPS.

### 1. Generate SSH Key for Deployment

This key will be used by you to SSH into the `deploy` user account, and by GitHub Actions to deploy the application.

```bash
# On your local machine
ssh-keygen -t ed25519 -f ~/.ssh/cosy_deploy -C "cosy-deployment"
```

### 2. Configure Variables

Set these environment variables in your local shell before running the setup playbook.

```bash
export DEPLOY_USER_SSH_KEY="$(cat ~/.ssh/cosy_deploy.pub)"
export COSY_DOMAIN="your.domain.com"
export LETSENCRYPT_EMAIL="your-email@example.com"
```

### 3. Update Inventory

Edit `docker/ansible/inventory-setup.yml` and set `ansible_host` to your VPS IP address.
```yaml
ansible_host: 1.2.3.4 # Replace with your VPS IP
```

### 4. Run Setup Playbook

This command will configure the server. At the end, it will output the database credentials that you need for GitHub Secrets.

```bash
cd docker/ansible
ansible-playbook -i inventory-setup.yml setup-vps.yml
```

### 5. Configure GitHub Secrets

After the playbook finishes, it will display the following secrets which must be stored in the GitHub repository secrets at **Settings → Secrets and variables → Actions**.

Required GitHub Secrets:
- `VPS_HOST`: Your VPS IP address.
- `DEPLOY_USER`: The deployment username (the default is `deploy`).
- `VPS_SSH_KEY`: The **private** SSH key you generated (`cat ~/.ssh/cosy_deploy`).

### 6. Test New User Access

You should now be able to SSH into the server as the `deploy` user with the key you generated.

```bash
ssh -i ~/.ssh/cosy_deploy deploy@YOUR_VPS_IP
```

## What the Initial Setup Does

✅ **System Updates**: Updates all packages and enables automatic security updates.
✅ **User Management**: Creates a `deploy` user with passwordless sudo access, adds your SSH key, and hardens SSH security by disabling root and password-based logins.
✅ **Docker Installation**: Installs Docker Engine & Docker Compose and adds the `deploy` user to the `docker` group.
✅ **Firewall**: Configures UFW to allow only SSH (22), HTTP (80), and HTTPS (443) traffic.
✅ **Security**: Installs and configures `fail2ban` to protect against SSH brute-force attacks.
✅ **Nginx & SSL**: Installs and configures nginx as a reverse proxy, obtains a Let's Encrypt SSL certificate, and sets up automatic renewal.
✅ **Application Setup**: Creates the `/opt/cosy` directory, copies the `docker-compose.yml` file, and generates database secrets. Starts the Docker containers.

## Updating Secrets on Existing Installation

If you need to add new secrets (e.g., InfluxDB credentials) to an already configured VPS:

### Running the Playbook on Existing VPS

The Ansible playbook is **idempotent**, meaning you can safely run it multiple times. It will:
- **Skip** tasks that are already complete (e.g., Docker installation, user creation)
- **Generate** only missing secrets without overwriting existing ones
- **Update** configuration files and restart services as needed

```bash
cd docker/ansible
ansible-playbook -i inventory-setup.yml setup-vps.yml
```

### What Happens with Secrets

The playbook checks if secret files exist in `/opt/cosy/secrets/` before generating new ones:
- **Existing secrets** (e.g., `loki_password`, `influx_password`) are **preserved** and reused
- **Missing secrets** are automatically generated
- All secrets are written to `/opt/cosy/.env` which is used by Docker Compose

### Behavior for Each Secret Type

| Secret | File Location | Generated If Missing | Overwritten on Re-run |
|--------|---------------|---------------------|----------------------|
| Loki Password | `/opt/cosy/secrets/loki_password` | ✅ Yes | ❌ No |
| InfluxDB Username | `/opt/cosy/secrets/influx_username` | ✅ Yes | ❌ No |
| InfluxDB Password | `/opt/cosy/secrets/influx_password` | ✅ Yes | ❌ No |
| InfluxDB Admin Token | `/opt/cosy/secrets/influx_admin_token` | ✅ Yes | ❌ No |

### Manual Secret Inspection

To view the generated secrets on the VPS:

```bash
ssh -i ~/.ssh/cosy_deploy deploy@YOUR_VPS_IP
sudo cat /opt/cosy/.env
```

**Note**: Keep these secrets secure. They should also be stored in your password manager or GitHub Secrets for backup.

