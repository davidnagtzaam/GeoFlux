# VPN Provider Setup

This guide explains how to obtain WireGuard configuration keys from supported VPN providers.

## Supported Providers

- Mullvad VPN
- ProtonVPN

Both providers require an active subscription to use their services.

## Mullvad VPN

### Prerequisites

- Active Mullvad account
- Mullvad account number (16-digit number)

### Obtaining WireGuard Keys

Mullvad supports up to 5 WireGuard keys per account.

#### Method 1: Mullvad Website (Recommended)

1. Go to [https://mullvad.net/account/](https://mullvad.net/account/)
2. Log in with your account number
3. Navigate to **WireGuard configuration**
4. Click **Generate key** for each country you need
5. Note the generated keys - you'll need:
   - Private key
   - IPv4 address

#### Method 2: Mullvad CLI

1. Install Mullvad CLI:
```bash
# Linux
curl -O https://mullvad.net/download/mullvad-cli-latest-linux.tar.gz
tar -xzf mullvad-cli-latest-linux.tar.gz

# macOS
brew install mullvad-cli
```

2. Generate keys:
```bash
mullvad account login YOUR_ACCOUNT_NUMBER
mullvad wireguard key generate
```

3. List your keys:
```bash
mullvad wireguard key list
```

#### Method 3: Manual API Call

```bash
# Generate a new WireGuard keypair locally
wg genkey > privatekey
wg pubkey < privatekey > publickey

# Upload public key to Mullvad
curl -X POST https://api.mullvad.net/wg/ \
  -H "Content-Type: application/json" \
  -d '{"account": "YOUR_ACCOUNT_NUMBER", "pubkey": "'$(cat publickey)'"}'
```

The response contains your IPv4 address.

### Configuration for .env

```bash
VPN_SERVICE_PROVIDER=mullvad

# Netherlands example
WG_NL_PRIVATE_KEY=YourPrivateKeyFromMullvadHere=
WG_NL_ADDRESSES=10.67.123.45/32
```

### Country Server Selection

Mullvad automatically selects the best server in the specified country. To use a specific city:

```yaml
# In docker-compose.yml for specific container
environment:
  - SERVER_CITIES=Amsterdam
```

Available cities vary by country. See [Mullvad servers](https://mullvad.net/en/servers/) for options.

### Key Management

- Maximum 5 keys per account
- Keys don't expire
- Delete old keys via the website if you need more
- Each key can be used on multiple containers simultaneously

## ProtonVPN

### Prerequisites

- Active ProtonVPN Plus or higher subscription (Free/Basic don't support WireGuard)
- ProtonVPN account credentials

### Obtaining WireGuard Keys

#### Method 1: ProtonVPN Dashboard (Recommended)

1. Go to [https://account.protonvpn.com/](https://account.protonvpn.com/)
2. Log in to your account
3. Navigate to **Downloads** → **WireGuard configuration**
4. Select the country you want
5. Download the configuration file or copy values manually

The downloaded config file contains:
```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = YOUR_IP_ADDRESS/32

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = SERVER_ENDPOINT
AllowedIPs = 0.0.0.0/0
```

You need:
- `PrivateKey` → Use for `WG_XX_PRIVATE_KEY`
- `Address` → Use for `WG_XX_ADDRESSES`

#### Method 2: ProtonVPN CLI

1. Install ProtonVPN CLI:
```bash
pip install protonvpn-cli
```

2. Login:
```bash
protonvpn init
```

3. Generate WireGuard config:
```bash
protonvpn c --protocol wireguard --country NL
protonvpn status
```

Note: This method connects directly. Extract keys from `/etc/wireguard/` configs instead.

#### Method 3: Using protonwire Tool

```bash
# Install protonwire
pip install protonwire

# Generate config
protonwire generate --country nl --output nl-config.conf
```

### Configuration for .env

```bash
VPN_SERVICE_PROVIDER=protonvpn

# Netherlands example
WG_NL_PRIVATE_KEY=YourProtonVPNPrivateKeyHere=
WG_NL_ADDRESSES=10.2.0.2/32
```

### Country Server Selection

ProtonVPN uses country codes. Gluetun will automatically select an appropriate server.

For specific server selection:

```yaml
# In docker-compose.yml for specific container
environment:
  - SERVER_NAMES=NL-FREE#1  # Specific server
```

See [ProtonVPN servers](https://account.protonvpn.com/downloads) for server names.

### ProtonVPN Considerations

- Free/Basic plans don't support WireGuard
- Plus or higher subscription required
- Some countries require Secure Core (higher tier)
- P2P/torrenting may be restricted on some servers

## Multi-Provider Setup

You can use different providers for different countries:

**Not Supported** - All containers must use the same provider (limitation of the current implementation).

To use multiple providers, run separate GeoFlux instances with different `.env` files.

## Generating WireGuard Keys Manually

If you need to generate keys outside provider tools:

```bash
# Generate private key
wg genkey > private.key

# Generate public key from private
wg pubkey < private.key > public.key

# Display keys
echo "Private: $(cat private.key)"
echo "Public: $(cat public.key)"
```

Then submit the public key to your provider's API or web interface.

## Testing Your Configuration

After configuring keys in `.env`:

1. Start a single container:
```bash
docker compose up -d gluetun-nl
```

2. Watch the logs:
```bash
docker compose logs -f gluetun-nl
```

3. Look for successful connection:
```
INFO [vpn] You are running on the bleeding edge of latest!
INFO [ip getter] Public IP address is XXX.XXX.XXX.XXX (Netherlands)
INFO [vpn] VPN is up
```

4. Test the proxy:
```bash
# Should show Netherlands IP
curl -x http://localhost:8891 https://ifconfig.me

# Should show Netherlands location
curl -x http://localhost:8891 https://ipapi.co/json/
```

## Troubleshooting

### Invalid Key Error

```
ERROR [wireguard] cannot start: key is invalid
```

**Solution:**
- Ensure key is exactly 44 characters (base64)
- Check for extra spaces or newlines
- Verify key is the private key, not public
- Regenerate key if corrupted

### Authentication Failed

```
ERROR [vpn] authentication failed
```

**Solutions:**
- Verify your subscription is active
- Check account is in good standing
- For ProtonVPN: Ensure Plus or higher tier
- For Mullvad: Verify account has time remaining

### Cannot Connect to Server

```
ERROR [wireguard] handshake did not complete
```

**Solutions:**
- Check firewall allows UDP port 51820
- Verify server country is available for your subscription tier
- Try different server/city
- Check provider status page for outages

### Wrong Country IP

If proxy shows wrong country:

**Solutions:**
- Verify `SERVER_COUNTRIES` matches your key's country
- For Mullvad: Ensure key was generated for correct country
- For ProtonVPN: Download config specifically for target country
- Check provider's server list for country availability

## Key Security

- Never share private keys
- Don't commit keys to version control
- Rotate keys every 3-6 months
- Use unique keys per environment (dev/staging/prod)
- Store keys in a secure password manager
- Set restrictive permissions: `chmod 600 .env`

## Key Rotation

To rotate keys without downtime:

1. Generate new key from provider
2. Update `.env` with new key
3. Restart specific container:
```bash
docker compose up -d --force-recreate gluetun-nl
```
4. Verify connection
5. Delete old key from provider dashboard

## Provider Comparison

| Feature          | Mullvad | ProtonVPN |
|------------------|---------|-----------|
| WireGuard        | Yes     | Yes (Plus+)|
| Max Keys         | 5       | Unlimited |
| Key Expiry       | Never   | Never     |
| API Access       | Yes     | Limited   |
| Port Forwarding  | Yes     | No        |
| Anonymous Signup | Yes     | No        |
| Payment Methods  | Crypto, Cash | Card, PayPal |

## Additional Resources

- [Mullvad WireGuard Setup](https://mullvad.net/help/wireguard/)
- [ProtonVPN WireGuard Setup](https://protonvpn.com/support/wireguard-configurations/)
- [Gluetun Wiki](https://github.com/qdm12/gluetun/wiki)
- [WireGuard Official Site](https://www.wireguard.com/)

## Support

If you encounter issues:

1. Check provider status pages:
   - [Mullvad Status](https://mullvad.net/en/check)
   - [ProtonVPN Status](https://protonstatus.com/)

2. Review Gluetun logs for specific errors

3. Consult provider support:
   - Mullvad: support@mullvad.net
   - ProtonVPN: https://protonvpn.com/support/

4. Check GeoFlux GitHub issues for similar problems
