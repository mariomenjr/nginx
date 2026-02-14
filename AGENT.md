# AGENT.md

## Architecture Overview

This project provides a containerized nginx web server with automatic SSL certificate management.

**Services:**
- `web` (nginx): Serves HTTP/HTTPS traffic, handles SSL termination
- `certbot`: Background service that auto-renews Let's Encrypt certificates every 12 hours

**Data Flow:**
1. HTTP requests on port 80 → ACME challenge handler or HTTPS redirect
2. HTTPS requests on port 443 → Serves content with SSL
3. Certbot writes to `certbot/www/` for ACME challenges
4. Certbot writes certificates to `nginx/certs/`

## Key Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Service definitions, port mappings, volume mounts |
| `nginx/conf/default.conf` | Server blocks, SSL configuration, location rules |
| `nginx/certs/` | SSL certificates (gitignored, mounted from host) |
| `nginx/logs/` | Access and error logs (gitignored) |
| `certbot/www/` | ACME challenge files for domain validation |

## Common Tasks

### Test Nginx Configuration
```bash
docker exec nginx nginx -t
```

### Reload Nginx (after config changes)
```bash
docker exec nginx nginx -s reload
```

### View Real-time Logs
```bash
# All services
docker compose logs -f

# Nginx only
docker compose logs -f web

# Certbot only
docker compose logs -f certbot
```

### Manual Certificate Renewal
```bash
docker exec certbot certbot renew
```

### Check Certificate Status
```bash
docker exec certbot certbot certificates
```

### Generate New Certificate
```bash
docker compose run --rm certbot certonly --webroot -w /var/www/certbot -d example.com -d www.example.com
```

## Directory Layout

```
/home/mariomenjr/src/web/
├── docker-compose.yml
├── .gitignore
├── README.md
├── AGENT.md
├── nginx/
│   ├── conf/
│   │   └── default.conf      # Main nginx configuration
│   ├── certs/                # SSL certificates (not in git)
│   │   └── live/
│   │       └── [domain]/
│   │           ├── fullchain.pem
│   │           └── privkey.pem
│   └── logs/                 # Log files (not in git)
│       ├── access.log
│       └── error.log
└── certbot/
    └── www/                  # ACME webroot (not in git)
        └── .well-known/
            └── acme-challenge/
```

## Deployment Checklist

Before deploying or after making changes:

- [ ] Update `nginx/conf/default.conf` with correct domain name
- [ ] Ensure DNS A/AAAA records point to server IP
- [ ] Verify ports 80 and 443 are open in firewall
- [ ] Generate initial SSL certificates before starting nginx
- [ ] Test nginx configuration: `docker exec nginx nginx -t`
- [ ] Check services are healthy: `docker compose ps`
- [ ] Verify HTTPS works: `curl -I https://domain.com`
- [ ] Verify auto-redirect: `curl -I http://domain.com` (should return 301)

## Configuration Changes

When modifying `nginx/conf/default.conf`:

1. Edit the file on the host
2. Test the configuration: `docker exec nginx nginx -t`
3. Reload nginx: `docker exec nginx nginx -s reload`
4. No container restart needed

## Adding New Domains

1. Update DNS records for new domain
2. Modify `nginx/conf/default.conf` - add new server block or update `server_name`
3. Generate certificate for new domain:
   ```bash
   docker compose run --rm certbot certonly --webroot -w /var/www/certbot -d newdomain.com
   ```
4. Test and reload nginx

## Gotchas

**Certificate Paths**: Nginx expects certificates at specific paths defined in `default.conf`. If domain changes, update both the `server_name` and SSL certificate paths.

**Initial Startup**: Nginx will fail to start if HTTPS server block references certificates that don't exist yet. Generate certificates first, or temporarily comment out the HTTPS server block.

**Permissions**: Certbot runs as root in container. Host directories need appropriate permissions for container to write certificates and logs.

**Port Conflicts**: Ensure no other service is using ports 80 or 443 on the host.

**Certbot Entrypoint**: The certbot service runs a loop that renews every 12 hours. Don't override the entrypoint unless you understand the auto-renewal logic.

**Volume Mounts**: Certificates are stored in `nginx/certs/` on host but mounted to `/etc/letsencrypt` in certbot container. This is intentional for certbot compatibility.

## Useful Commands

```bash
# Restart all services
docker compose restart

# Rebuild and restart
docker compose up -d --force-recreate

# Enter nginx container
docker exec -it nginx sh

# Enter certbot container
docker exec -it certbot sh

# Check nginx syntax without reloading
docker exec nginx nginx -t

# View nginx version
docker exec nginx nginx -v

# Follow certbot renewal logs
docker compose logs -f certbot | grep -i renew
```

## Security Considerations

- Never commit `nginx/certs/` directory
- Certificates are auto-renewed but monitor expiration
- Keep Docker images updated: `docker compose pull`
- Review SSL configuration periodically for best practices
