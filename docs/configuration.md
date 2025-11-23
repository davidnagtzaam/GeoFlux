# Configuration Guide

This guide covers all configuration options for GeoFlux.

## Environment Variables

All configuration is done via the `.env` file. Copy `.env.example` to `.env` and configure the following variables:

### Core Configuration

#### VPN_SERVICE_PROVIDER

Specifies which VPN provider to use.

```bash
VPN_SERVICE_PROVIDER=mullvad
# or
VPN_SERVICE_PROVIDER=protonvpn
```

**Supported values:**
- `mullvad` - Mullvad VPN
- `protonvpn` - ProtonVPN

### WireGuard Configuration

Each country requires two environment variables:

#### Private Key

The WireGuard private key for the specific country server.

```bash
WG_NL_PRIVATE_KEY=your_private_key_here
```

Format: Base64-encoded 32-byte private key

#### IP Address

The IP address assigned to your WireGuard interface.

```bash
WG_NL_ADDRESSES=10.x.x.x/32
```

Format: IPv4 address with /32 subnet mask

### Country Variables

Each country has its own set of variables:

| Country       | Code | Private Key Variable    | Address Variable    |
|---------------|------|-------------------------|---------------------|
| Netherlands   | NL   | WG_NL_PRIVATE_KEY       | WG_NL_ADDRESSES     |
| United States | US   | WG_US_PRIVATE_KEY       | WG_US_ADDRESSES     |
| Germany       | DE   | WG_DE_PRIVATE_KEY       | WG_DE_ADDRESSES     |
| Norway        | NO   | WG_NO_PRIVATE_KEY       | WG_NO_ADDRESSES     |
| United Kingdom| UK   | WG_UK_PRIVATE_KEY       | WG_UK_ADDRESSES     |
| Estonia       | EE   | WG_EE_PRIVATE_KEY       | WG_EE_ADDRESSES     |
| Latvia        | LV   | WG_LV_PRIVATE_KEY       | WG_LV_ADDRESSES     |
| Sweden        | SE   | WG_SE_PRIVATE_KEY       | WG_SE_ADDRESSES     |
| Finland       | FI   | WG_FI_PRIVATE_KEY       | WG_FI_ADDRESSES     |
| Switzerland   | CH   | WG_CH_PRIVATE_KEY       | WG_CH_ADDRESSES     |
| France        | FR   | WG_FR_PRIVATE_KEY       | WG_FR_ADDRESSES     |
| Spain         | ES   | WG_ES_PRIVATE_KEY       | WG_ES_ADDRESSES     |
| Italy         | IT   | WG_IT_PRIVATE_KEY       | WG_IT_ADDRESSES     |
| Canada        | CA   | WG_CA_PRIVATE_KEY       | WG_CA_ADDRESSES     |
| Australia     | AU   | WG_AU_PRIVATE_KEY       | WG_AU_ADDRESSES     |

## HAProxy Configuration

The HAProxy configuration is in `haproxy.cfg`. Key sections:

### Global Settings

```
global
    log stdout format raw local0 info
    maxconn 4096
    user haproxy
    group haproxy
    daemon
```

**Options:**
- `maxconn` - Maximum concurrent connections (default: 4096)
- `log` - Logging configuration

### Timeouts

```
defaults
    timeout connect 10s
    timeout client 300s
    timeout server 300s
    timeout tunnel 3600s
```

**Timeout values:**
- `connect` - Connection establishment timeout (default: 10s)
- `client` - Client inactivity timeout (default: 300s)
- `server` - Server inactivity timeout (default: 300s)
- `tunnel` - Long-lived connection timeout (default: 3600s)

Adjust these based on your use case:
- Short-lived requests: Decrease client/server timeouts
- Long-polling/WebSockets: Increase tunnel timeout
- Slow networks: Increase connect timeout

### Basic Authentication

To enable Basic Authentication:

1. Generate a password hash:
```bash
docker run --rm haproxy:lts-alpine \
  sh -c 'echo -n "YourPassword" | mkpasswd --method=sha-256 --stdin'
```

2. Edit `haproxy.cfg` and uncomment:
```
frontend main_proxy
    # Uncomment these lines:
    acl auth_ok http_auth(basic_auth)
    http-request auth realm GeoFlux if !auth_ok
```

3. At the end of `haproxy.cfg`, configure the userlist:
```
userlist basic_auth
    user admin password $5$rounds=535000$YOUR_HASH_HERE
```

4. Restart HAProxy:
```bash
docker compose restart haproxy
```

5. Use credentials when connecting:
```bash
curl -x http://admin:YourPassword@localhost:8888 https://ifconfig.me
```

### Stats Page Configuration

The stats page is accessible on port 8404:

```
listen stats
    bind *:8404
    mode http
    stats enable
    stats uri /
    stats refresh 30s
```

**Options:**
- `bind` - Change to `127.0.0.1:8404` to restrict to localhost
- `stats uri` - URL path (default: `/`)
- `stats refresh` - Auto-refresh interval
- `stats auth user:pass` - Add authentication (uncomment and set)

### Health Check Configuration

Each backend has health checks:

```
backend gluetun_nl
    option httpchk GET /
    server gluetun-nl gluetun-nl:8888 check inter 30s fall 3 rise 2
```

**Parameters:**
- `inter` - Check interval (default: 30s)
- `fall` - Failed checks before marking down (default: 3)
- `rise` - Successful checks before marking up (default: 2)

Adjust for your reliability needs:
- Critical services: Decrease `inter`, increase `rise`
- Resource-constrained: Increase `inter`

## Docker Compose Configuration

### Health Checks

Each Gluetun container has a health check:

```yaml
healthcheck:
  test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8888"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 45s
```

**Parameters:**
- `test` - Uses `wget` to check HTTP proxy availability (more reliable than `nc`)
- `interval` - Time between checks
- `timeout` - Max time for check to complete
- `retries` - Failed checks before unhealthy
- `start_period` - Grace period on container start

### Resource Limits

Add resource limits to prevent overuse:

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

### Restart Policies

All containers use `restart: unless-stopped`:

```yaml
restart: unless-stopped
```

**Options:**
- `no` - Never restart
- `always` - Always restart
- `on-failure` - Restart only on errors
- `unless-stopped` - Restart unless manually stopped

## Gluetun Configuration

### Firewall and Security

These options are set in `docker-compose.yml`:

```yaml
environment:
  - FIREWALL=on
  - DOT=on
  - HTTPPROXY=on
  - HTTPPROXY_LOG=off
```

**Options:**
- `FIREWALL=on` - Enables killswitch (required)
- `DOT=on` - DNS-over-TLS (recommended)
- `HTTPPROXY=on` - Enable HTTP proxy (required)
- `HTTPPROXY_LOG=off` - Disable proxy logging (privacy)

### VPN Connection Settings

```yaml
environment:
  - VPN_TYPE=wireguard
  - HEALTH_VPN_DURATION_INITIAL=30s
```

**Options:**
- `VPN_TYPE` - Always `wireguard`
- `HEALTH_VPN_DURATION_INITIAL` - Time before VPN health checks start

### Advanced Gluetun Options

Add these to specific containers if needed:

```yaml
# Custom DNS servers (if not using DOT)
- DNS_ADDRESS=1.1.1.1

# Specific city selection (provider dependent)
- SERVER_CITIES=Amsterdam

# Custom VPN port (if provider allows)
- VPN_PORT=51820

# Disable IPv6 (if issues occur)
- FIREWALL_OUTBOUND_SUBNETS=

# Custom MTU (if packet issues)
- WIREGUARD_MTU=1420
```

See [Gluetun wiki](https://github.com/qdm12/gluetun/wiki) for all options.

## Port Configuration

To change exposed ports, edit `docker-compose.yml`:

```yaml
services:
  haproxy:
    ports:
      - "8888:8888"    # Change left side for host port
      - "9000:8891"    # Example: expose Netherlands on port 9000
```

**Format:** `HOST_PORT:CONTAINER_PORT`

## Network Configuration

The default network is sufficient for most use cases:

```yaml
networks:
  vpn-network:
    driver: bridge
```

For advanced setups, you can:

1. Use a specific subnet:
```yaml
networks:
  vpn-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

2. Connect to existing network:
```yaml
networks:
  vpn-network:
    external: true
    name: my_existing_network
```

## Logging Configuration

### Log Rotation

All containers include automatic log rotation to prevent disk space issues:

```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"  # Maximum size of each log file
    max-file: "3"    # Keep 3 rotated log files
```

This configuration keeps a maximum of 30MB of logs per container (3 files × 10MB).

### HAProxy Logs

View HAProxy logs:
```bash
docker compose logs -f haproxy
```

Adjust log level in `haproxy.cfg`:
```
global
    log stdout format raw local0 info  # Change to: debug, warning, error
```

### Gluetun Logs

View VPN logs:
```bash
docker compose logs -f gluetun-nl
```

Reduce verbosity:
```yaml
environment:
  - LOG_LEVEL=info  # Options: debug, info, warning, error
```

## Environment-Specific Configuration

### Development

For development, you might want:
- Fewer countries (faster startup)
- More verbose logging
- Stats page on all interfaces

### Production

For production, ensure:
- Basic Auth enabled
- Stats restricted to localhost or secured
- Resource limits configured
- All health checks enabled
- Proper restart policies

### Example Production .env

```bash
VPN_SERVICE_PROVIDER=mullvad
WG_NL_PRIVATE_KEY=production_key_here
WG_NL_ADDRESSES=10.x.x.x/32
# ... all other countries

# Production-specific (if needed)
HAPROXY_STATS_AUTH=admin:secure_hash_here
```

## Troubleshooting Configuration Issues

### Invalid WireGuard Key

Error: `Failed to start VPN`
- Ensure key is valid base64
- Check for extra spaces or newlines
- Verify key is for correct country/provider

### Port Already in Use

Error: `bind: address already in use`
- Change host port in docker-compose.yml
- Stop conflicting services
- Check with: `lsof -i :8888`

### Container Won't Start

- Check Docker daemon: `systemctl status docker`
- Verify NET_ADMIN capability: `getcap $(which docker)`
- Review logs: `docker compose logs gluetun-nl`

### Environment Variable Not Loaded

- Ensure `.env` is in same directory as `docker-compose.yml`
- No quotes needed around values in `.env`
- Restart after changes: `docker compose down && docker compose up -d`

## Configuration Validation

Before deploying:

1. Validate Docker Compose syntax:
```bash
docker compose config
```

2. Validate HAProxy configuration:
```bash
docker run --rm -v $(pwd)/haproxy.cfg:/test.cfg haproxy:lts-alpine haproxy -c -f /test.cfg
```

3. Test VPN connectivity:
```bash
docker compose up -d gluetun-nl
docker compose logs -f gluetun-nl
# Wait for "VPN is up"
```

4. Test proxy:
```bash
curl -x http://localhost:8891 https://ifconfig.me
```
