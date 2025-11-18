# OpenProject with Cloudflare Tunnel - Production Setup

Production-ready OpenProject deployment using the all-in-one Docker container with Cloudflare Tunnel for secure HTTPS access.

## Deployment Methods

### Coolify (Recommended for VPS)

**Complete guide**: See [COOLIFY_SETUP.md](./COOLIFY_SETUP.md)

This is the recommended approach if you're running Coolify on your VPS. The guide includes:
- Cloudflare Tunnel setup with Docker method
- Coolify deployment steps
- Environment variable configuration
- Backup strategies
- Production hardening

### Standalone Docker Compose

If you're deploying without Coolify, follow the instructions below.

## Prerequisites

- Docker and Docker Compose installed
- A Cloudflare account with a domain
- Access to Cloudflare Zero Trust dashboard

## Quick Start (Standalone Docker)

### 1. Set Up Cloudflare Tunnel

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
2. Navigate to **Networks** > **Tunnels**
3. Click **Create a tunnel**
4. Choose **Cloudflared** as the connector
5. Name your tunnel (e.g., `openproject-tunnel`)
6. Copy the tunnel token (you'll need this for `.env`)
7. Configure a public hostname:
   - **Subdomain**: Your desired subdomain (e.g., `openproject`)
   - **Domain**: Your domain from the dropdown
   - **Service Type**: `HTTP`
   - **URL**: `http://openproject:80`
     - Protocol: `http` (Cloudflare handles SSL/TLS)
     - Container: `openproject` (service name from docker-compose.yml)
     - Port: `80` (OpenProject's internal port)
8. Save the tunnel

### 2. Configure Environment Variables

```bash
# Copy the example environment file
cp .env.example .env

# Generate a secure secret key
openssl rand -hex 64

# Edit .env and fill in:
# - SECRET_KEY_BASE (use the generated key above)
# - OPENPROJECT_HOST (your Cloudflare tunnel domain, e.g., openproject.yourdomain.com)
# - CLOUDFLARE_TUNNEL_TOKEN (from step 1.6)
```

### 3. Start the Services

```bash
# Start OpenProject and Cloudflare Tunnel
docker-compose up -d

# Check logs
docker-compose logs -f

# Wait for OpenProject to fully start (can take 2-3 minutes on first run)
```

### 4. Access OpenProject

1. Navigate to your Cloudflare tunnel URL (e.g., `https://openproject.yourdomain.com`)
2. Default login credentials:
   - **Username**: `admin`
   - **Password**: `admin`
3. **Important**: Change the admin password immediately after first login

## Production Recommendations

### Security

- [ ] Change the default admin password immediately
- [ ] Set up regular backups (see Backup section below)
- [ ] Configure 2FA for admin accounts in OpenProject
- [ ] Review Cloudflare Access policies to restrict access if needed
- [ ] Keep the `.env` file secure and never commit it to version control

### Backups

To backup your OpenProject data:

```bash
# Stop the services
docker-compose down

# Backup the volumes
docker run --rm \
  -v openproject-zenrise_openproject-pgdata:/data/pgdata \
  -v openproject-zenrise_openproject-assets:/data/assets \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/openproject-backup-$(date +%Y%m%d-%H%M%S).tar.gz -C /data .

# Restart the services
docker-compose up -d
```

### Restore from Backup

```bash
# Stop the services
docker-compose down

# Remove old volumes
docker volume rm openproject-zenrise_openproject-pgdata openproject-zenrise_openproject-assets

# Recreate volumes
docker volume create openproject-zenrise_openproject-pgdata
docker volume create openproject-zenrise_openproject-assets

# Restore from backup
docker run --rm \
  -v openproject-zenrise_openproject-pgdata:/data/pgdata \
  -v openproject-zenrise_openproject-assets:/data/assets \
  -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/your-backup-file.tar.gz -C /data

# Restart the services
docker-compose up -d
```

## Maintenance

### Update OpenProject

```bash
# Pull the latest image
docker-compose pull

# Recreate containers with new image
docker-compose up -d

# Check logs for any migration messages
docker-compose logs -f openproject
```

### View Logs

```bash
# All services
docker-compose logs -f

# OpenProject only
docker-compose logs -f openproject

# Cloudflare Tunnel only
docker-compose logs -f cloudflared
```

### Restart Services

```bash
# Restart all services
docker-compose restart

# Restart specific service
docker-compose restart openproject
```

## Troubleshooting

### OpenProject won't start

Check the logs:
```bash
docker-compose logs openproject
```

Common issues:
- Invalid `SECRET_KEY_BASE` - regenerate using `openssl rand -hex 64`
- Insufficient disk space - check with `df -h`

### Cannot access through Cloudflare Tunnel

1. Check tunnel status in Cloudflare Zero Trust dashboard
2. Verify `CLOUDFLARE_TUNNEL_TOKEN` is correct in `.env`
3. Ensure the public hostname points to `http://openproject:80`
4. Check cloudflared logs: `docker-compose logs cloudflared`

### Database issues

If you need to reset the database:
```bash
docker-compose down
docker volume rm openproject-zenrise_openproject-pgdata
docker-compose up -d
```

**Warning**: This will delete all data!

## Architecture

- **OpenProject**: All-in-one container including PostgreSQL database
- **Cloudflare Tunnel**: Secure tunnel providing HTTPS access without exposing ports
- **Volumes**: Persistent storage for database and uploaded files

## What's Included

- `docker-compose.yml` - Production-ready Docker Compose configuration
- `.env.example` - Environment variables template
- `COOLIFY_SETUP.md` - Complete Coolify deployment guide
- `.gitignore` - Protects sensitive files

## Requirements

### Minimum System Requirements
- 2 GB RAM
- 2 CPU cores
- 10 GB disk space
- Docker 20.10+
- Docker Compose 2.0+

### For Production
- 4 GB RAM recommended
- Regular backups (automated via cron)
- SSL/TLS via Cloudflare Tunnel
- Monitoring and alerting

## Support

- **OpenProject Documentation**: https://www.openproject.org/docs/
- **Cloudflare Tunnel Docs**: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- **Coolify Docs**: https://coolify.io/docs
