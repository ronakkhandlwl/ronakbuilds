---
title: "How to Host n8n on Oracle Cloud Free Tier: Complete Guide"
description: "Step-by-step guide to hosting n8n workflow automation tool on Oracle Cloud's always-free tier, including domain setup and SSL configuration"
pubDate: 2026-01-03
heroImage: '../../assets/n8n_on_oracle.png'
tags: ["n8n", "oracle-cloud", "automation", "self-hosting", "docker"]
---

n8n is a powerful workflow automation tool that lets you connect different services and automate tasks. In this comprehensive guide, I'll show you how to host n8n on Oracle Cloud's generous free tier, which gives you a server that's available 24/7 at no cost.

## What You'll Need

- An Oracle Cloud account (free tier)
- A terminal/command line application
- Basic familiarity with SSH and command line
- Optional: A domain name for custom URL and SSL

## What is Oracle Cloud Free Tier?

Oracle Cloud offers an "Always Free" tier that includes:
- 2 AMD-based Compute VMs (1/8 OCPU, 1 GB memory each) OR
- 4 Arm-based Ampere A1 cores and 24 GB memory total
- 200 GB of block storage
- 10 TB of outbound data transfer per month

This is perfect for hosting n8n!

## Step 1: Create an Oracle Cloud Account

1. Visit [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/)
2. Click "Start for free" and complete the registration
3. Verify your email and add a payment method (required but won't be charged on free tier)
4. Wait for account activation (usually instant)

## Step 2: Create a Compute Instance

### Launch the Instance

1. Log in to the Oracle Cloud Console
2. Click the hamburger menu (☰) → **Compute** → **Instances**
3. Click **Create Instance**

### Configure Your Instance

**Basic Information:**
- **Name**: `n8n-server` (or any name you prefer)
- **Compartment**: Select your compartment (usually root)

**Image and Shape:**
1. **Image**: Click "Change Image"
   - Select **Ubuntu 22.04** (Canonical Ubuntu 22.04)
   - Click "Select Image"

2. **Shape**: Click "Change Shape"
   - Select **Virtual Machine**
   - Select **Specialty and previous generation**
   - Choose **VM.Standard.E2.1.Micro**
   - Specs: 1/8 OCPU, 1 GB Memory
   - Click "Select Shape"

> **Note**: If you try the Ampere ARM instances (VM.Standard.A1.Flex) and get an "Out of capacity" error, don't worry - just use the E2.1.Micro. It works great for n8n!

**Networking:**
- Keep the default VCN or create a new one
- Ensure "Assign a public IPv4 address" is checked ✓

**SSH Keys:**
- Select "Generate a key pair for me"
- Click **Save Private Key** - This is important! You'll need this to access your server
- Optionally save the public key too

**Boot Volume:**
- Leave default (50 GB is sufficient)

Click **Create** and wait about 1-2 minutes for the instance to provision. Note down the **Public IP Address** displayed on the instance page.

## Step 3: Configure Network Security Rules

Your instance needs firewall rules to allow web traffic.

1. From your instance page, click on the **Subnet** link
2. Click on **Security Lists** in the left sidebar
3. Click **Default Security List**
4. Click **Add Ingress Rules**

Add these three rules one by one:

**Rule 1 - HTTP (Port 80):**
- Source Type: CIDR
- Source CIDR: `0.0.0.0/0`
- IP Protocol: TCP
- Destination Port Range: `80`
- Description: HTTP
- Click "Add Ingress Rules"

**Rule 2 - HTTPS (Port 443):**
- Source CIDR: `0.0.0.0/0`
- IP Protocol: TCP
- Destination Port Range: `443`
- Description: HTTPS
- Click "Add Ingress Rules"

**Rule 3 - n8n (Port 5678):**
- Source CIDR: `0.0.0.0/0`
- IP Protocol: TCP
- Destination Port Range: `5678`
- Description: n8n
- Click "Add Ingress Rules"

## Step 4: Connect to Your Instance via SSH

Open your terminal (Linux/Mac) or Git Bash/PowerShell (Windows).

### Set Correct Permissions for Your Key
```bash
# Navigate to where you downloaded the key
cd ~/Downloads

# Set permissions (Linux/Mac)
chmod 400 ssh-key-*.key

# For Windows using PowerShell:
# icacls ssh-key-*.key /inheritance:r
# icacls ssh-key-*.key /grant:r "%username%:R"
```

### Connect to Your Server
```bash
ssh -i ssh-key-*.key ubuntu@YOUR_PUBLIC_IP
```

Replace:
- `ssh-key-*.key` with your actual key filename
- `YOUR_PUBLIC_IP` with the IP from Oracle Console

Type `yes` when prompted to accept the fingerprint.

## Step 5: Install Docker

Once connected to your server, update the system and install Docker:
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add your user to docker group
sudo usermod -aG docker ubuntu

# Verify installation
docker --version
```

**Important**: Log out and log back in for the group changes to take effect:
```bash
exit
```

Then reconnect using the same SSH command.

## Step 6: Configure Ubuntu Firewall

Allow the necessary ports through Ubuntu's firewall:
```bash
# Add firewall rules
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 5678 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT

# Install iptables-persistent to save rules
sudo apt install -y iptables-persistent

# Save the rules
sudo netfilter-persistent save
```

## Step 7: Run n8n with Docker

Create a directory for n8n data and run the container:
```bash
# Create data directory
mkdir -p ~/.n8n

# Create a Docker volume for n8n data
docker volume create n8n_data

# Run n8n container
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="0.0.0.0" \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=http \
  -e N8N_SECURE_COOKIE=false \
  -e WEBHOOK_URL="http://YOUR_PUBLIC_IP:5678/" \
  -e GENERIC_TIMEZONE="America/New_York" \
  -v n8n_data:/home/node/.n8n \
  --restart unless-stopped \
  n8nio/n8n
```

Replace:
- `YOUR_PUBLIC_IP` with your actual server IP
- `America/New_York` with your timezone ([list of timezones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones))

### Verify n8n is Running
```bash
# Check if container is running
docker ps

# View logs
docker logs -f n8n
```

Wait until you see: "n8n ready on ::, port 5678"

Press `Ctrl+C` to exit the logs.

## Step 8: Access n8n

Open your browser and navigate to:
```
http://YOUR_PUBLIC_IP:5678
```

You should see the n8n welcome screen! Create your owner account and start building workflows.

## Useful Docker Commands

Here are some helpful commands for managing your n8n instance:
```bash
# Stop n8n
docker stop n8n

# Start n8n
docker start n8n

# Restart n8n
docker restart n8n

# View live logs
docker logs -f n8n

# View last 100 log lines
docker logs --tail 100 n8n

# Update n8n to latest version
docker stop n8n
docker rm n8n
docker pull n8nio/n8n
# Then run the docker run command again

# Backup your data
docker run --rm -v n8n_data:/data -v $(pwd):/backup ubuntu tar czf /backup/n8n-backup.tar.gz /data
```

## Optional: Connect a Custom Domain

Using your own domain name makes your n8n instance more professional and enables SSL/HTTPS.

### Prerequisites

- A domain name (you can get one from Namecheap, Google Domains, Cloudflare, etc.)
- Access to your domain's DNS settings

### Step 1: Add DNS A Record

Log in to your domain registrar and add an A record:

**For most providers (Namecheap, GoDaddy, Cloudflare, etc.):**

1. Go to DNS Management for your domain
2. Add a new record:
   - **Type**: A
   - **Name**: `n8n` (or `@` for root domain)
   - **Value/Points to**: Your Oracle instance public IP
   - **TTL**: 3600 or Auto

Example: If your domain is `example.com`, this creates `n8n.example.com` pointing to your server.

Wait 5-15 minutes for DNS propagation. You can check if it's working:
```bash
# On your local computer
ping n8n.example.com
# Should show your Oracle instance IP
```

### Step 2: Install and Configure Nginx

SSH back into your server and install Nginx:
```bash
# Install Nginx
sudo apt update
sudo apt install -y nginx

# Create Nginx configuration for n8n
sudo nano /etc/nginx/sites-available/n8n
```

Paste this configuration (replace `n8n.example.com` with your actual domain):
```nginx
server {
    listen 80;
    server_name n8n.example.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Save the file (`Ctrl+X`, then `Y`, then `Enter`).
```bash
# Enable the site
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/

# Remove the default site to avoid conflicts
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx

# Check Nginx status
sudo systemctl status nginx
```

Now you can access n8n at: `http://n8n.example.com` (without the port number!)

### Step 3: Add SSL Certificate (HTTPS)

Let's Encrypt provides free SSL certificates. Install Certbot:
```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Get SSL certificate (replace with your domain)
sudo certbot --nginx -d n8n.example.com
```

Follow the prompts:
1. Enter your email address
2. Agree to the terms of service
3. Choose whether to share your email
4. Select option 2 to redirect HTTP to HTTPS

### Step 4: Update n8n for HTTPS

Update your n8n container to use HTTPS:
```bash
# Stop and remove the container
docker stop n8n
docker rm n8n

# Run with HTTPS configuration
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="n8n.example.com" \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=https \
  -e WEBHOOK_URL="https://n8n.example.com/" \
  -e GENERIC_TIMEZONE="America/New_York" \
  -v n8n_data:/home/node/.n8n \
  --restart unless-stopped \
  n8nio/n8n
```

Your n8n instance is now accessible at: `https://n8n.example.com` 🎉

The SSL certificate will auto-renew thanks to Certbot's automatic renewal service.

## Monitoring and Maintenance

### Check Resource Usage

Since we're on the 1GB RAM instance, it's good to monitor usage:
```bash
# Check memory usage
free -h

# Check Docker container stats
docker stats n8n

# Check disk usage
df -h
```

### Backup Your Data

Your n8n data is stored in the Docker volume. Regular backups are important:
```bash
# Create a backup
docker run --rm \
  -v n8n_data:/data \
  -v $(pwd):/backup \
  ubuntu tar czf /backup/n8n-backup-$(date +%Y%m%d).tar.gz /data

# Download to your local machine (run on your local computer)
scp -i ~/Downloads/ssh-key-*.key ubuntu@YOUR_PUBLIC_IP:~/n8n-backup-*.tar.gz ./
```

To restore a backup:
```bash
# Upload backup to server, then:
docker run --rm \
  -v n8n_data:/data \
  -v $(pwd):/backup \
  ubuntu tar xzf /backup/n8n-backup-YYYYMMDD.tar.gz -C /
```

### Update n8n

Keep n8n updated for the latest features and security patches:
```bash
# Pull the latest image
docker pull n8nio/n8n

# Stop and remove old container
docker stop n8n
docker rm n8n

# Run with the same configuration (use your specific settings)
docker run -d \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="n8n.example.com" \
  -e N8N_PORT=5678 \
  -e N8N_PROTOCOL=https \
  -e WEBHOOK_URL="https://n8n.example.com/" \
  -e GENERIC_TIMEZONE="America/New_York" \
  -v n8n_data:/home/node/.n8n \
  --restart unless-stopped \
  n8nio/n8n
```

## Troubleshooting Common Issues

### Can't Access n8n

1. **Check if n8n is running**: `docker ps`
2. **Check logs**: `docker logs n8n`
3. **Verify firewall rules** in Oracle Cloud Security Lists
4. **Check Ubuntu firewall**: `sudo iptables -L -n | grep 5678`

### 502 Bad Gateway Error

This means Nginx can't connect to n8n:

1. **Check if n8n is running**: `docker ps`
2. **Restart n8n**: `docker restart n8n`
3. **Check if port 5678 is accessible**: `curl -I http://localhost:5678`

### Out of Memory Errors

If n8n crashes due to memory (1GB is limited):

1. **Consider upgrading** to the ARM instance when capacity is available (4GB+ RAM)
2. **Limit workflows** - avoid running too many complex workflows simultaneously
3. **Monitor usage**: `docker stats n8n`

## Conclusion

You now have a fully functional n8n instance running on Oracle Cloud's free tier! This setup costs you nothing and gives you a powerful automation platform that's accessible 24/7.

Key benefits of this setup:
- ✅ Completely free (Oracle Cloud free tier)
- ✅ Always online
- ✅ Your data stays private on your server
- ✅ Full control over customization and integrations
- ✅ Optional custom domain with SSL

Start building your workflows and automating your tasks! Check out [n8n's documentation](https://docs.n8n.io/) for workflow ideas and integrations.

Happy automating! 🚀

**Found this helpful?** Share it with others who might benefit from self-hosting n8n!