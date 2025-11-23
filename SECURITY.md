# Security

This document outlines security considerations, best practices, and hardening recommendations for GeoFlux.

## Security Principles

GeoFlux is designed with the following security principles:

1. **Least Privilege** - Containers run with minimal required permissions
2. **Killswitch Protection** - VPN killswitch prevents IP leaks if VPN drops
3. **No DNS Leaks** - All DNS queries routed through VPN tunnel using DNS-over-TLS
4. **Secrets Management** - Credentials stored in environment variables, never committed
5. **Network Isolation** - Dedicated Docker network for inter-container communication

## Built-in Security Features

### VPN Killswitch

Each Gluetun container has `FIREWALL=on` which enables the killswitch. If the VPN tunnel drops:
- All outbound traffic is blocked
- No traffic can leak outside the VPN tunnel
- Container remains isolated until VPN reconnects

### DNS Leak Prevention

All containers use DNS-over-TLS (`DOT=on`) which:
- Encrypts DNS queries
- Routes DNS through the VPN tunnel
- Prevents DNS leaks to ISP or local network

### Network Isolation

Containers communicate over a dedicated bridge network:
- No direct host network access
- Isolated from other Docker networks
- Only HAProxy ports exposed to host

## Recommended Security Hardening

### 1. Enable HTTP Basic Authentication

Protect the proxy from unauthorized access:

1. Generate a password hash:
```bash
docker run --rm haproxy:lts-alpine sh -c 'echo -n "your_password" | mkpasswd --method=sha-256 --stdin'
```

2. Edit `haproxy.cfg` and uncomment the auth section:
```
# In frontend main_proxy
acl auth_ok http_auth(basic_auth)
http-request auth realm GeoFlux if !auth_ok

# At the end of the file
userlist basic_auth
    user admin password $5$rounds=535000$yourhashhere
```

3. Restart HAProxy:
```bash
docker compose restart haproxy
```

### 2. Restrict Network Access

Use firewall rules to limit access to the proxy:

```bash
# Allow only specific IP range (example)
iptables -A INPUT -p tcp --dport 8888:8905 -s 192.168.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 8888:8905 -j DROP
```

Or use Docker networks to restrict access to specific containers only.

### 3. Secure Environment Files

Protect your `.env` file containing VPN credentials:

```bash
chmod 600 .env
```

Never commit `.env` to version control. The `.gitignore` should include:
```
.env
*.env
!.env.example
```

### 4. Use Strong VPN Keys

- Generate unique WireGuard keys for each country
- Rotate keys periodically (every 3-6 months)
- Store keys securely (password manager, secrets vault)
- Never share keys between environments (dev/staging/prod)

### 5. Monitor and Audit

Enable logging and monitor for suspicious activity:

```bash
# View HAProxy logs
docker compose logs -f haproxy

# View specific country VPN logs
docker compose logs -f gluetun-nl

# Monitor stats dashboard
http://localhost:8404
```

Set up alerts for:
- VPN connection failures
- Unusual traffic patterns
- Health check failures
- Container restarts

### 6. Limit HAProxy Stats Access

The stats page (port 8404) exposes system information. To secure it:

1. Edit `haproxy.cfg` stats section:
```
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /stats
    stats auth admin:your_secure_password
```

2. Or bind stats only to localhost:
```
listen stats
    bind 127.0.0.1:8404
```

### 7. Regular Updates

Keep all components updated:

```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d
```

Check for updates to:
- Gluetun (monthly)
- HAProxy (quarterly)
- Base OS images (monthly)

### 8. Resource Limits

Prevent resource exhaustion by setting container limits in `docker-compose.yml`:

```yaml
services:
  gluetun-nl:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M
```

### 9. TLS/SSL for HAProxy (Optional)

For production deployments, consider enabling HTTPS on HAProxy:

1. Obtain SSL certificate (Let's Encrypt, etc.)
2. Mount certificate in docker-compose.yml
3. Update haproxy.cfg bind directives:
```
bind *:8888 ssl crt /path/to/cert.pem
```

## Threat Model

### In Scope

GeoFlux protects against:
- IP address leaks (via killswitch)
- DNS leaks (via DoT)
- Unauthorized proxy access (via optional Basic Auth)
- Container escape (via least privilege)

### Out of Scope

GeoFlux does NOT protect against:
- Compromised VPN provider
- Traffic analysis by VPN provider
- Malicious applications on the same Docker host
- Physical access to the host system
- Social engineering attacks

## Vulnerability Reporting

If you discover a security vulnerability in GeoFlux:

1. Do NOT open a public issue
2. Email details to: david@davidnagtzaam.com
3. Include:
   - Description of the vulnerability
   - Steps to reproduce
   - Potential impact
   - Suggested fix (if any)

We will respond within 48 hours and work on a fix.

## Compliance Considerations

### VPN Provider Terms of Service

Ensure your use of GeoFlux complies with your VPN provider's terms:
- Check concurrent connection limits
- Verify commercial use is allowed (if applicable)
- Review data transfer restrictions

### Legal Compliance

- Obey local laws regarding VPN usage
- Respect copyright and intellectual property laws
- Do not use for illegal activities
- Check if VPN usage is restricted in your jurisdiction

### Data Protection

- GeoFlux does not log or store traffic data
- Gluetun may log connection metadata (check Gluetun docs)
- HAProxy logs can contain request metadata (disable if needed)
- VPN provider has visibility into your traffic

## Security Checklist

Before deploying to production:

- [ ] `.env` file has restrictive permissions (600)
- [ ] Basic Auth enabled on HAProxy
- [ ] Stats page secured or restricted to localhost
- [ ] Firewall rules configured for port access
- [ ] Resource limits set on containers
- [ ] Monitoring and alerting configured
- [ ] Regular update schedule established
- [ ] Backup of configuration files
- [ ] Incident response plan documented
- [ ] VPN provider terms of service reviewed

## Additional Resources

- [Gluetun Security](https://github.com/qdm12/gluetun/wiki/Security)
- [HAProxy Security](https://www.haproxy.com/documentation/hapee/latest/security/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [WireGuard Protocol](https://www.wireguard.com/)

---

**Last Updated**: 2025-01-23
