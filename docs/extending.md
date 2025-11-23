# Extending GeoFlux

This guide explains how to add new countries, customize routing, and extend GeoFlux functionality.

## Adding New Countries

To add a new country endpoint, you need to modify three files:

1. `.env` - Add WireGuard configuration
2. `docker-compose.yml` - Add Gluetun container
3. `haproxy.cfg` - Add routing configuration

### Step 1: Add Environment Variables

Edit `.env` and add the new country's variables:

```bash
# Japan (JP) example
WG_JP_PRIVATE_KEY=your_japan_wireguard_private_key_here
WG_JP_ADDRESSES=10.x.x.x/32
```

Obtain the WireGuard keys from your VPN provider (see [providers.md](providers.md)).

### Step 2: Add Gluetun Container

Edit `docker-compose.yml` and add a new service:

```yaml
  # Japan VPN Container
  gluetun-jp:
    image: qmcgaw/gluetun:v3.39.1
    container_name: gluetun-jp
    cap_drop:
      - ALL
    cap_add:
      - NET_ADMIN
    environment:
      - VPN_SERVICE_PROVIDER=${VPN_SERVICE_PROVIDER}
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WG_JP_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=${WG_JP_ADDRESSES}
      - SERVER_COUNTRIES=Japan
      - FIREWALL=on
      - DOT=on
      - HTTPPROXY=on
      - HTTPPROXY_LOG=off
      - HEALTH_VPN_DURATION_INITIAL=30s
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8888"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 45s
    deploy:
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 128M
    networks:
      - vpn-network
```

**Important:**
- Use pinned version tag (e.g., `v3.39.1`) instead of `:latest` for production stability
- Include `cap_drop: ALL` before `cap_add: NET_ADMIN` for least privilege security
- Add logging configuration to prevent disk space issues
- Add resource limits to prevent resource exhaustion
- Use `wget` in health check (more reliable than `nc`)
- Use the correct country name for `SERVER_COUNTRIES` (check provider's server list)
- Container name pattern: `gluetun-{country_code}`
- All security settings (FIREWALL, DOT, HTTPPROXY) must remain `on`

### Step 3: Update HAProxy Dependencies

In `docker-compose.yml`, add the new container to HAProxy's dependencies:

```yaml
  haproxy:
    image: haproxy:3.0-alpine
    container_name: haproxy
    depends_on:
      - gluetun-nl
      - gluetun-us
      # ... existing countries
      - gluetun-jp  # Add this line
```

### Step 4: Expose Port (Optional)

If you want a dedicated port for the new country, add it to HAProxy's ports:

```yaml
  haproxy:
    ports:
      - "8888:8888"
      # ... existing ports
      - "8906:8906"    # Japan
```

### Step 5: Configure HAProxy Routing

Edit `haproxy.cfg` and add three sections:

#### a. Header-based ACL and Route

In the `frontend main_proxy` section, add:

```
    # In the ACL section
    acl route_jp hdr(X-Exit-Country) -i jp japan

    # In the routing section
    use_backend gluetun_jp if route_jp
```

#### b. Port-based Frontend (if using dedicated port)

Add a new frontend:

```
# Japan - Port 8906
frontend port_jp
    bind *:8906
    mode http
    default_backend gluetun_jp
```

#### c. Backend Definition

Add a new backend:

```
backend gluetun_jp
    mode http
    balance roundrobin
    option httpchk GET /
    server gluetun-jp gluetun-jp:8888 check inter 30s fall 3 rise 2
```

### Step 6: Start the New Container

```bash
# Start only the new container
docker compose up -d gluetun-jp

# Or restart entire stack
docker compose down
docker compose up -d
```

### Step 7: Verify

```bash
# Check container is healthy
docker compose ps gluetun-jp

# Test header-based routing
curl -x http://localhost:8888 -H "X-Exit-Country: jp" https://ifconfig.me

# Test port-based routing (if configured)
curl -x http://localhost:8906 https://ifconfig.me

# Verify location
curl -x http://localhost:8906 https://ipapi.co/json/ | jq '.country'
```

## Complete Example: Adding Poland (PL)

### 1. `.env`

```bash
WG_PL_PRIVATE_KEY=your_poland_wireguard_private_key_here
WG_PL_ADDRESSES=10.67.123.78/32
```

### 2. `docker-compose.yml`

```yaml
  gluetun-pl:
    image: qmcgaw/gluetun:latest
    container_name: gluetun-pl
    cap_add:
      - NET_ADMIN
    environment:
      - VPN_SERVICE_PROVIDER=${VPN_SERVICE_PROVIDER}
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=${WG_PL_PRIVATE_KEY}
      - WIREGUARD_ADDRESSES=${WG_PL_ADDRESSES}
      - SERVER_COUNTRIES=Poland
      - FIREWALL=on
      - DOT=on
      - HTTPPROXY=on
      - HTTPPROXY_LOG=off
      - HEALTH_VPN_DURATION_INITIAL=30s
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "nc", "-z", "localhost", "8888"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 45s
    networks:
      - vpn-network

  haproxy:
    depends_on:
      - gluetun-pl  # Add to existing list
    ports:
      - "8906:8906"  # Add to existing list
```

### 3. `haproxy.cfg`

```
# In frontend main_proxy
frontend main_proxy
    # ... existing config
    acl route_pl hdr(X-Exit-Country) -i pl poland
    use_backend gluetun_pl if route_pl

# New frontend for port-based
frontend port_pl
    bind *:8906
    mode http
    default_backend gluetun_pl

# New backend
backend gluetun_pl
    mode http
    balance roundrobin
    option httpchk GET /
    server gluetun-pl gluetun-pl:8888 check inter 30s fall 3 rise 2
```

## Removing Countries

To remove a country you don't use:

1. Remove the Gluetun container from `docker-compose.yml`
2. Remove HAProxy frontend and backend from `haproxy.cfg`
3. Remove the port from HAProxy's port list
4. Remove from HAProxy's `depends_on`
5. Remove ACL and routing from `frontend main_proxy`
6. Optionally remove from `.env`

Then restart:

```bash
docker compose down
docker compose up -d
```

## Customizing Country Aliases

You can add more aliases for country codes in `haproxy.cfg`:

```
# Default
acl route_nl hdr(X-Exit-Country) -i nl netherlands

# Add more aliases
acl route_nl hdr(X-Exit-Country) -i nl netherlands holland nederland
```

This allows users to use:
- `X-Exit-Country: nl`
- `X-Exit-Country: netherlands`
- `X-Exit-Country: holland`
- `X-Exit-Country: nederland`

## Port Numbering Convention

Current ports follow this pattern:
- 8888: Main proxy (header-based routing)
- 8891-8905: Country-specific ports
- 8404: Stats page

When adding countries, continue the sequence:
- 8906: 16th country
- 8907: 17th country
- etc.

Avoid using:
- Ports below 1024 (require root)
- Port 8404 (stats page)
- Port 8888 (main proxy)

## Advanced Routing

### Load Balancing Within a Country

If your VPN provider allows multiple simultaneous connections to the same country:

```yaml
# docker-compose.yml
  gluetun-us-1:
    # ... config with US key 1

  gluetun-us-2:
    # ... config with US key 2
```

```
# haproxy.cfg
backend gluetun_us
    mode http
    balance roundrobin
    option httpchk GET /
    server gluetun-us-1 gluetun-us-1:8888 check
    server gluetun-us-2 gluetun-us-2:8888 check
```

### Region-Based Routing

Route to multiple countries based on region:

```
# haproxy.cfg
frontend main_proxy
    # ... existing config
    acl route_eu hdr(X-Exit-Region) -i eu europe
    use_backend eu_servers if route_eu

backend eu_servers
    mode http
    balance roundrobin
    server gluetun-nl gluetun-nl:8888 check
    server gluetun-de gluetun-de:8888 check
    server gluetun-fr gluetun-fr:8888 check
```

### City-Specific Routing

For providers that support city selection:

```yaml
# docker-compose.yml
  gluetun-us-nyc:
    environment:
      - SERVER_COUNTRIES=United States
      - SERVER_CITIES=New York

  gluetun-us-la:
    environment:
      - SERVER_COUNTRIES=United States
      - SERVER_CITIES=Los Angeles
```

```
# haproxy.cfg
acl route_us_nyc hdr(X-Exit-City) -i nyc new-york
acl route_us_la hdr(X-Exit-City) -i la los-angeles

use_backend gluetun_us_nyc if route_us_nyc
use_backend gluetun_us_la if route_us_la
```

## Switching VPN Providers

### Mullvad to ProtonVPN

1. Update `.env`:
```bash
VPN_SERVICE_PROVIDER=protonvpn
```

2. Obtain new WireGuard keys from ProtonVPN (see [providers.md](providers.md))

3. Update all `WG_*_PRIVATE_KEY` and `WG_*_ADDRESSES` variables

4. Restart containers:
```bash
docker compose down
docker compose up -d
```

### Mixed Providers (Not Supported)

Currently, all containers must use the same provider. To use multiple providers:

1. Clone the repository to separate directories
2. Configure each with different provider
3. Use different port ranges
4. Run as separate Docker Compose stacks

Example:
```bash
# Stack 1: Mullvad
cd ~/geoflux-mullvad
# Configure with Mullvad, ports 8888-8905
docker compose up -d

# Stack 2: ProtonVPN
cd ~/geoflux-proton
# Configure with ProtonVPN, ports 9888-9905
# Edit docker-compose.yml to change port mappings
docker compose up -d
```

## Custom Gluetun Configuration

### Enable Specific Features

Add to specific containers in `docker-compose.yml`:

#### Port Forwarding (Mullvad)
```yaml
environment:
  - VPN_PORT_FORWARDING=on
  - VPN_PORT_FORWARDING_PROVIDER=mullvad
```

#### IPv6 Support
```yaml
environment:
  - FIREWALL_OUTBOUND_SUBNETS=0.0.0.0/0,::/0
```

#### Custom DNS
```yaml
environment:
  - DOT=off
  - DNS_ADDRESS=1.1.1.1,1.0.0.1
```

### Container Resource Limits

Add resource constraints:

```yaml
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

## Troubleshooting New Countries

### Container won't start

```bash
# Check logs
docker compose logs gluetun-jp

# Common issues:
# - Invalid WireGuard key
# - Wrong country name
# - Provider doesn't support country
```

### Wrong IP/Country

```bash
# Verify configuration
docker compose exec gluetun-jp cat /gluetun/wireguard/wg0.conf

# Check Gluetun's detected country
docker compose logs gluetun-jp | grep "Public IP"
```

### HAProxy can't reach backend

```bash
# Check health
docker compose ps

# Test backend directly
docker compose exec haproxy nc -zv gluetun-jp 8888

# Check HAProxy stats
open http://localhost:8404
```

## Testing Changes

After extending GeoFlux:

```bash
# Validate Docker Compose
docker compose config

# Validate HAProxy config
docker run --rm -v $(pwd)/haproxy.cfg:/test.cfg \
  haproxy:lts-alpine haproxy -c -f /test.cfg

# Test new country
curl -x http://localhost:8888 -H "X-Exit-Country: jp" https://ifconfig.me

# Check all services healthy
docker compose ps
```

## Best Practices

1. **Naming Convention**: Keep container names as `gluetun-{code}`
2. **Port Allocation**: Use sequential ports starting from 8906
3. **Security Settings**: Always keep FIREWALL=on and DOT=on
4. **Health Checks**: Use identical health check config for consistency
5. **Documentation**: Update README.md port mapping table
6. **Testing**: Always test both header and port-based routing
7. **Version Control**: Commit changes with clear messages

## Example: Adding 5 Asian Countries

Quick reference for adding multiple countries at once:

```bash
# .env additions
WG_JP_PRIVATE_KEY=...
WG_JP_ADDRESSES=...
WG_SG_PRIVATE_KEY=...
WG_SG_ADDRESSES=...
WG_HK_PRIVATE_KEY=...
WG_HK_ADDRESSES=...
WG_IN_PRIVATE_KEY=...
WG_IN_ADDRESSES=...
WG_KR_PRIVATE_KEY=...
WG_KR_ADDRESSES=...
```

Then add containers, frontends, and backends following the patterns above.

## Further Customization

For more advanced customization:
- [HAProxy Documentation](https://www.haproxy.com/documentation/)
- [Gluetun Wiki](https://github.com/qdm12/gluetun/wiki)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)

---

Need help extending GeoFlux? Open an issue on GitHub with your use case.
