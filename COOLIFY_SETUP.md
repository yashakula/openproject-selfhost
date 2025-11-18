# OpenProject Deployment on Coolify with Cloudflare Tunnel

This guide walks you through deploying OpenProject on Coolify with Cloudflare Tunnel for secure production access.

## Prerequisites

- Coolify installed and running on your VPS
- A Cloudflare account with a domain configured
- Git repository access (GitHub, GitLab, etc.)

## Step 1: Set Up Cloudflare Tunnel

### Create the Tunnel

1. Log in to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Navigate to **Networks** > **Tunnels**
3. Click **Create a tunnel**
4. Select **Cloudflared** connector
5. Name your tunnel (e.g., `openproject-prod`)
6. **Important**: Choose **Docker** as the installation method
7. Copy the tunnel token that looks like: `eyJhIjoiXXXXXXX...`
8. Save this token securely - you'll need it for Coolify

### Configure Public Hostname

1. In the tunnel configuration, go to **Public Hostname** tab
2. Click **Add a public hostname**
3. Configure:
   - **Subdomain**: Your desired subdomain (e.g., `openproject`)
   - **Domain**: Your domain from the dropdown (e.g., `yourdomain.com`)
   - **Service Type**: `HTTP`
   - **URL**: `http://openproject:80`
4. Click **Save hostname**

**Important Configuration Details:**
- **Protocol**: `http` (not https - Cloudflare handles SSL/TLS termination)
- **Container Name**: `openproject` (matches the service name in docker-compose.yml)
- **Port**: `80` (OpenProject's internal container port)
- The URL format is: `http://<container-name>:<port>`

The container name `openproject` works because both services (openproject and cloudflared) are on the same Docker network, allowing them to communicate using service names as hostnames.

Your OpenProject will be accessible at `https://openproject.yourdomain.com` (Cloudflare automatically provides HTTPS)

## Step 2: Deploy on Coolify

### Create New Project

1. Log in to your Coolify dashboard
2. Click **+ New** > **Project**
3. Name it `OpenProject` or similar
4. Click **+ Add**

### Add Resource

1. Inside the project, click **+ New** > **Resource**
2. Select **Docker Compose**
3. Choose one of these options:

#### Option A: Deploy from Git Repository (Recommended)

1. Select **Public Repository** or **Private Repository**
2. Enter your repository URL (where this docker-compose.yml is located)
3. Set branch to `main` or your default branch
4. Click **Continue**

#### Option B: Deploy from Docker Compose

1. Select **Docker Compose** > **Simple**
2. Paste the contents of `docker-compose.yml`
3. Click **Continue**

### Configure Environment Variables

In the Coolify environment variables section, add:

```env
SECRET_KEY_BASE=<generate-with-openssl-rand-hex-64>
OPENPROJECT_HOST=openproject.yourdomain.com
CLOUDFLARE_TUNNEL_TOKEN=<your-tunnel-token-from-cloudflare>
```

**To generate SECRET_KEY_BASE:**
```bash
openssl rand -hex 64
```

### Configure Deployment Settings

1. **Name**: `OpenProject Production`
2. **Destination**: Select your VPS server
3. **Network**: Use default or create a dedicated network
4. **Deployment**: Set to your preferred method (webhook, manual, etc.)

### Deploy

1. Click **Save**
2. Click **Deploy**
3. Monitor the deployment logs
4. Wait for both services to start (OpenProject takes 2-3 minutes on first run)

## Step 3: Verify Deployment

### Check Service Status

1. In Coolify, navigate to your OpenProject resource
2. Verify both containers are running:
   - `openproject` - should show healthy
   - `cloudflared` - should show running

### Check Logs

```bash
# OpenProject logs
View in Coolify UI under "Logs" > Select "openproject" service

# Cloudflare Tunnel logs
View in Coolify UI under "Logs" > Select "cloudflared" service
```

### Access OpenProject

1. Navigate to `https://openproject.yourdomain.com`
2. You should see the OpenProject login screen
3. Default credentials:
   - **Username**: `admin`
   - **Password**: `admin`
4. **Critical**: Change the admin password immediately

## Step 4: Production Hardening

### Immediate Actions

- [ ] Change default admin password
- [ ] Configure 2FA for admin account
- [ ] Review and set up user accounts for your team
- [ ] Configure email settings (SMTP) in OpenProject admin panel
- [ ] Set up regular backups (see below)

### Backup Configuration in Coolify

Coolify doesn't automatically backup Docker volumes. Set up a backup strategy:

#### Manual Backup via Coolify Terminal

1. In Coolify, open the **Terminal** for your OpenProject service
2. Run backup commands:

```bash
# Create backup directory
mkdir -p /backups

# Backup volumes (run from host VPS, not container)
docker run --rm \
  -v <project-name>_openproject-pgdata:/data/pgdata \
  -v <project-name>_openproject-assets:/data/assets \
  -v /path/to/backups:/backup \
  alpine tar czf /backup/openproject-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .
```

Replace `<project-name>` with your Coolify project's Docker prefix (check with `docker volume ls`)

#### Automated Backup Script

Create a backup script on your VPS:

```bash
#!/bin/bash
# Save as: /root/backup-openproject.sh

BACKUP_DIR="/backups/openproject"
RETENTION_DAYS=30
PROJECT_NAME="<your-coolify-project-name>"

mkdir -p $BACKUP_DIR

# Create backup
docker run --rm \
  -v ${PROJECT_NAME}_openproject-pgdata:/data/pgdata \
  -v ${PROJECT_NAME}_openproject-assets:/data/assets \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/openproject-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .

# Delete old backups
find $BACKUP_DIR -name "openproject-backup-*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $(date)"
```

Add to crontab for daily backups:
```bash
chmod +x /root/backup-openproject.sh
crontab -e
# Add: 0 2 * * * /root/backup-openproject.sh >> /var/log/openproject-backup.log 2>&1
```

## Troubleshooting

### OpenProject Not Starting

1. Check logs in Coolify for the `openproject` service
2. Common issues:
   - Invalid `SECRET_KEY_BASE`: Regenerate with `openssl rand -hex 64`
   - Insufficient disk space: Check VPS storage
   - Port conflicts: Ensure no other services use internal ports

### Cloudflare Tunnel Not Connecting

1. Verify tunnel token is correct in environment variables
2. Check `cloudflared` logs in Coolify
3. Ensure tunnel is active in Cloudflare dashboard
4. Verify public hostname points to `http://openproject:80`

### Cannot Access OpenProject via Domain

1. Check Cloudflare Tunnel status in dashboard
2. Verify DNS is propagated (can take up to 24 hours)
3. Check both service logs in Coolify
4. Test tunnel connectivity: `cloudflared tunnel info <tunnel-id>`

### Performance Issues

1. Check VPS resources (CPU, RAM, disk)
2. OpenProject all-in-one requires minimum:
   - 2 GB RAM
   - 2 CPU cores
   - 10 GB disk space
3. Consider upgrading VPS if needed

## Updating OpenProject

### Via Coolify

1. Navigate to your OpenProject resource
2. Click **Redeploy**
3. Coolify will pull the latest image and restart services
4. Monitor logs for migration messages
5. Test functionality after update

### Manual Update

If using a specific version, update `docker-compose.yml`:
```yaml
openproject:
  image: openproject/openproject:17  # Update version
```

Then redeploy in Coolify.

## Monitoring

### Health Checks

Coolify uses the healthcheck defined in docker-compose.yml:
- Checks OpenProject every 30 seconds
- Endpoint: `http://localhost:80/health_checks/default`
- Cloudflared depends on OpenProject being healthy

### Logs

Monitor logs regularly in Coolify:
- Application errors
- Database issues
- Cloudflare Tunnel connection status

## Support and Resources

- **Coolify**: https://coolify.io/docs
- **OpenProject**: https://www.openproject.org/docs/
- **Cloudflare Tunnel**: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- **OpenProject Community**: https://community.openproject.org/

## Security Notes

- Never expose environment variables in logs or public repositories
- Keep your Cloudflare tunnel token secure
- Regularly update OpenProject via Coolify
- Monitor Cloudflare Access logs for suspicious activity
- Consider setting up Cloudflare Access policies to restrict access by email/IP
