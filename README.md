# GeoFlux

Production-grade multi-country VPN egress proxy service for routing application traffic through different VPN endpoints on a per-request basis.

## Overview

GeoFlux provides instant, secure access to VPN endpoints across 15 countries using a simple HTTP proxy interface. Route traffic through different countries using custom headers or dedicated ports, with built-in killswitch protection and DNS leak prevention.

## Architecture

```
┌─────────────┐
│   Client    │
│ Application │
└──────┬──────┘
       │
       │ HTTP Proxy Request
       ├─ Header: X-Exit-Country: nl  OR  Port: 8891
       │
       ▼
┌──────────────────────────────────────────────┐
│            HAProxy (Port 8888)               │
│  ┌────────────────────────────────────────┐  │
│  │  Header-based routing (X-Exit-Country) │  │
│  │  Port-based routing (8891-8905)        │  │
│  │  Optional Basic Auth                   │  │
│  │  Health Checks & Stats (8404)          │  │
│  └────────────────────────────────────────┘  │
└────┬──────┬──────┬──────┬──────┬───────┬────┘
     │      │      │      │      │       │
     ▼      ▼      ▼      ▼      ▼       ▼
  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐  ┌────┐
  │ NL │ │ US │ │ DE │ │ UK │ │ SE │  │... │  (15 Gluetun containers)
  └────┘ └────┘ └────┘ └────┘ └────┘  └────┘
     │      │      │      │      │       │
     └──────┴──────┴──────┴──────┴───────┘
                    │
              WireGuard VPN Tunnels
              (ProtonVPN/Mullvad)
```

## Features

- **15 Country Endpoints**: NL, US, DE, NO, UK, EE, LV, SE, FI, CH, FR, ES, IT, CA, AU
- **Dual Routing Modes**: Header-based (`X-Exit-Country`) or port-based (8891-8905)
- **VPN Providers**: ProtonVPN and Mullvad support via WireGuard
- **Security**: Killswitch, DNS-over-TLS, firewall protection
- **Monitoring**: Built-in HAProxy stats dashboard (port 8404)
- **Optional Authentication**: HTTP Basic Auth for access control
- **Health Checks**: Automatic container health monitoring

## Quick Start

### Prerequisites

**Required:**
- Docker Engine 20.10+ and Docker Compose 2.0+ (or docker-compose 1.29+)
- Linux, macOS, or Windows with WSL2
- 8GB+ RAM recommended for running all 15 VPN containers
- Active ProtonVPN (Plus tier or higher) or Mullvad subscription
- WireGuard configuration keys for each country you want to use

**Minimum to get started:**
- You can start with just one country configured to test the system

### Installation

1. Clone the repository:
```bash
git clone <repository-url>
cd geoflux
```

2. Copy and configure environment file:
```bash
cp .env.example .env
```

3. Add your VPN credentials to `.env`:
```bash
# Set your provider (mullvad or protonvpn)
VPN_SERVICE_PROVIDER=mullvad

# Configure WireGuard keys for each country
WG_NL_PRIVATE_KEY=your_netherlands_key_here
WG_NL_ADDRESSES=10.x.x.x/32
# ... repeat for all countries
```

See [docs/providers.md](docs/providers.md) for detailed instructions on obtaining WireGuard keys.

4. Start the service:
```bash
docker compose up -d
```

5. Verify all containers are healthy:
```bash
docker compose ps
```

## Usage

### Header-Based Routing

Route requests through different countries using the `X-Exit-Country` header:

```bash
# Netherlands
curl -x http://localhost:8888 -H "X-Exit-Country: nl" https://ifconfig.me

# United States
curl -x http://localhost:8888 -H "X-Exit-Country: us" https://ifconfig.me

# Germany
curl -x http://localhost:8888 -H "X-Exit-Country: de" https://ifconfig.me
```

### Port-Based Routing

Use dedicated ports for direct country access:

```bash
# Netherlands (8891)
curl -x http://localhost:8891 https://ifconfig.me

# United States (8892)
curl -x http://localhost:8892 https://ifconfig.me

# Germany (8893)
curl -x http://localhost:8893 https://ifconfig.me
```

### Port Mapping

| Country       | Code | Port |
|---------------|------|------|
| Netherlands   | NL   | 8891 |
| United States | US   | 8892 |
| Germany       | DE   | 8893 |
| Norway        | NO   | 8894 |
| United Kingdom| UK   | 8895 |
| Estonia       | EE   | 8896 |
| Latvia        | LV   | 8897 |
| Sweden        | SE   | 8898 |
| Finland       | FI   | 8899 |
| Switzerland   | CH   | 8900 |
| France        | FR   | 8901 |
| Spain         | ES   | 8902 |
| Italy         | IT   | 8903 |
| Canada        | CA   | 8904 |
| Australia     | AU   | 8905 |

### Monitoring

Access the HAProxy stats dashboard:
```
http://localhost:8404
```

## Configuration

See [docs/configuration.md](docs/configuration.md) for detailed configuration options including:
- Environment variables
- Basic authentication setup
- Custom health check intervals
- Logging configuration

**Docker Images Used:**
- Gluetun: `qmcgaw/gluetun:v3.39.1`
- HAProxy: `haproxy:3.0-alpine`

Images are pinned to specific versions for production stability. To update, modify the version tags in `docker-compose.yml`.

## Common Issues

### Container fails to start with "permission denied"

Ensure your Docker user has NET_ADMIN capability. On Linux, you may need to run:
```bash
sudo docker compose up -d
```

### VPN connection fails immediately

- Verify your WireGuard private keys are correct in `.env`
- Check that your VPN subscription is active
- Ensure the country is supported by your VPN provider
- Review container logs: `docker compose logs gluetun-nl`

### "No route to host" or connection timeouts

- Wait 30-60 seconds after startup for VPN tunnels to establish
- Check container health: `docker compose ps`
- Verify firewall rules aren't blocking Docker networks
- Check HAProxy stats at http://localhost:8404

### DNS leaks detected

All DNS queries are routed through the VPN tunnel using DNS-over-TLS. If you suspect leaks:
- Verify DOT=on in docker-compose.yml
- Test with: `curl -x http://localhost:8891 https://dnsleaktest.com`
- Check Gluetun logs for DNS configuration

### Proxy returns 503 Service Unavailable

- The selected VPN tunnel may not be established yet
- Check container health status
- Review HAProxy backend status at http://localhost:8404
- Restart the specific country container: `docker compose restart gluetun-nl`

### Performance issues or slow connections

- Some VPN servers may be overloaded; try a different country
- Increase timeout values in haproxy.cfg if needed
- Check your internet connection speed
- Review resource usage: `docker stats`

## Documentation

- [Configuration Guide](docs/configuration.md) - Detailed configuration options
- [Provider Setup](docs/providers.md) - VPN provider WireGuard key setup
- [Usage Examples](docs/usage.md) - Integration examples for common applications
- [Extending](docs/extending.md) - Adding new countries or providers

## Security

See [SECURITY.md](SECURITY.md) for security best practices and hardening recommendations.

## License

MIT License - See [LICENSE](LICENSE) file for details.

## Author

**David Nagtzaam**
Email: david@davidnagtzaam.com

## Support

For issues, questions, or contributions, please open an issue on the repository.

---

**Note**: This software is provided as-is. Ensure compliance with your VPN provider's terms of service and local regulations when using this proxy service.
