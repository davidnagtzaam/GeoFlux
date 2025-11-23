# Usage Examples

This guide provides practical examples for using GeoFlux with various applications and programming languages.

## Basic Usage

### Command Line (curl)

#### Header-Based Routing

```bash
# Netherlands
curl -x http://localhost:8888 \
  -H "X-Exit-Country: nl" \
  https://ifconfig.me

# United States
curl -x http://localhost:8888 \
  -H "X-Exit-Country: us" \
  https://ifconfig.me

# Multiple headers work too
curl -x http://localhost:8888 \
  -H "X-Exit-Country: de" \
  -H "User-Agent: Mozilla/5.0" \
  https://api.ipify.org
```

#### Port-Based Routing

```bash
# Netherlands (8891)
curl -x http://localhost:8891 https://ifconfig.me

# United States (8892)
curl -x http://localhost:8892 https://ifconfig.me

# Germany (8893)
curl -x http://localhost:8893 https://ifconfig.me
```

#### With Authentication

If Basic Auth is enabled:

```bash
curl -x http://admin:password@localhost:8888 \
  -H "X-Exit-Country: nl" \
  https://ifconfig.me
```

### Testing Your Connection

#### Check IP Address

```bash
curl -x http://localhost:8891 https://ifconfig.me
```

#### Check Location

```bash
curl -x http://localhost:8891 https://ipapi.co/json/
```

#### DNS Leak Test

```bash
curl -x http://localhost:8891 https://www.dnsleaktest.com/
```

## Programming Language Examples

### Python

#### Using requests Library

```python
import requests

# Port-based routing
proxies = {
    'http': 'http://localhost:8891',   # Netherlands
    'https': 'http://localhost:8891',
}

response = requests.get('https://ifconfig.me', proxies=proxies)
print(f"IP: {response.text}")

# Header-based routing
proxies = {
    'http': 'http://localhost:8888',
    'https': 'http://localhost:8888',
}

headers = {'X-Exit-Country': 'us'}
response = requests.get('https://ifconfig.me',
                       proxies=proxies,
                       headers=headers)
print(f"IP: {response.text}")
```

#### Using aiohttp (Async)

```python
import aiohttp
import asyncio

async def fetch_with_proxy(country_code):
    proxies = 'http://localhost:8888'
    headers = {'X-Exit-Country': country_code}

    async with aiohttp.ClientSession() as session:
        async with session.get('https://ifconfig.me',
                              proxy=proxies,
                              headers=headers) as response:
            ip = await response.text()
            print(f"{country_code}: {ip}")

# Fetch from multiple countries concurrently
async def main():
    await asyncio.gather(
        fetch_with_proxy('nl'),
        fetch_with_proxy('us'),
        fetch_with_proxy('de'),
    )

asyncio.run(main())
```

#### Using urllib

```python
import urllib.request

proxy = urllib.request.ProxyHandler({
    'http': 'http://localhost:8891',
    'https': 'http://localhost:8891'
})

opener = urllib.request.build_opener(proxy)
urllib.request.install_opener(opener)

response = urllib.request.urlopen('https://ifconfig.me')
print(response.read().decode())
```

### Node.js

#### Using axios

```javascript
const axios = require('axios');

// Port-based routing
const config = {
  proxy: {
    host: 'localhost',
    port: 8891  // Netherlands
  }
};

axios.get('https://ifconfig.me', config)
  .then(response => console.log('IP:', response.data))
  .catch(error => console.error(error));

// Header-based routing
const configHeaders = {
  proxy: {
    host: 'localhost',
    port: 8888
  },
  headers: {
    'X-Exit-Country': 'us'
  }
};

axios.get('https://ifconfig.me', configHeaders)
  .then(response => console.log('IP:', response.data))
  .catch(error => console.error(error));
```

#### Using node-fetch

```javascript
const fetch = require('node-fetch');
const HttpsProxyAgent = require('https-proxy-agent');

// Port-based
const proxyAgent = new HttpsProxyAgent('http://localhost:8891');

fetch('https://ifconfig.me', { agent: proxyAgent })
  .then(res => res.text())
  .then(body => console.log('IP:', body))
  .catch(err => console.error(err));

// Header-based
const proxyAgentHeader = new HttpsProxyAgent('http://localhost:8888');

fetch('https://ifconfig.me', {
  agent: proxyAgentHeader,
  headers: { 'X-Exit-Country': 'nl' }
})
  .then(res => res.text())
  .then(body => console.log('IP:', body))
  .catch(err => console.error(err));
```

#### Using native http/https

```javascript
const https = require('https');

const options = {
  hostname: 'ifconfig.me',
  port: 443,
  path: '/',
  method: 'GET',
  headers: {
    'Host': 'ifconfig.me'
  },
  agent: new (require('https-proxy-agent'))('http://localhost:8891')
};

https.request(options, (res) => {
  let data = '';
  res.on('data', (chunk) => data += chunk);
  res.on('end', () => console.log('IP:', data));
}).end();
```

### Go

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "net/url"
)

func main() {
    // Port-based routing
    proxyURL, _ := url.Parse("http://localhost:8891")
    client := &http.Client{
        Transport: &http.Transport{
            Proxy: http.ProxyURL(proxyURL),
        },
    }

    resp, err := client.Get("https://ifconfig.me")
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)
    fmt.Println("IP:", string(body))

    // Header-based routing
    proxyURL2, _ := url.Parse("http://localhost:8888")
    client2 := &http.Client{
        Transport: &http.Transport{
            Proxy: http.ProxyURL(proxyURL2),
        },
    }

    req, _ := http.NewRequest("GET", "https://ifconfig.me", nil)
    req.Header.Set("X-Exit-Country", "us")

    resp2, err := client2.Do(req)
    if err != nil {
        panic(err)
    }
    defer resp2.Body.Close()

    body2, _ := io.ReadAll(resp2.Body)
    fmt.Println("IP:", string(body2))
}
```

### PHP

```php
<?php

// Port-based routing
$context = stream_context_create([
    'http' => [
        'proxy' => 'tcp://localhost:8891',
        'request_fulluri' => true,
    ],
]);

$response = file_get_contents('https://ifconfig.me', false, $context);
echo "IP: $response\n";

// Header-based routing with cURL
$ch = curl_init('https://ifconfig.me');
curl_setopt($ch, CURLOPT_PROXY, 'localhost:8888');
curl_setopt($ch, CURLOPT_HTTPHEADER, ['X-Exit-Country: nl']);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

$response = curl_exec($ch);
curl_close($ch);

echo "IP: $response\n";
?>
```

### Ruby

```ruby
require 'net/http'
require 'uri'

# Port-based routing
uri = URI.parse('https://ifconfig.me')
proxy = URI.parse('http://localhost:8891')

http = Net::HTTP.new(uri.host, uri.port, proxy.host, proxy.port)
http.use_ssl = true

response = http.get(uri.path)
puts "IP: #{response.body}"

# Header-based routing
proxy2 = URI.parse('http://localhost:8888')
http2 = Net::HTTP.new(uri.host, uri.port, proxy2.host, proxy2.port)
http2.use_ssl = true

request = Net::HTTP::Get.new(uri.path)
request['X-Exit-Country'] = 'de'

response2 = http2.request(request)
puts "IP: #{response2.body}"
```

### Java

```java
import java.net.*;
import java.io.*;

public class ProxyExample {
    public static void main(String[] args) throws Exception {
        // Port-based routing
        Proxy proxy = new Proxy(Proxy.Type.HTTP,
            new InetSocketAddress("localhost", 8891));

        URL url = new URL("https://ifconfig.me");
        HttpURLConnection conn = (HttpURLConnection) url.openConnection(proxy);

        BufferedReader in = new BufferedReader(
            new InputStreamReader(conn.getInputStream()));
        System.out.println("IP: " + in.readLine());
        in.close();

        // Header-based routing
        Proxy proxy2 = new Proxy(Proxy.Type.HTTP,
            new InetSocketAddress("localhost", 8888));

        HttpURLConnection conn2 = (HttpURLConnection) url.openConnection(proxy2);
        conn2.setRequestProperty("X-Exit-Country", "us");

        BufferedReader in2 = new BufferedReader(
            new InputStreamReader(conn2.getInputStream()));
        System.out.println("IP: " + in2.readLine());
        in2.close();
    }
}
```

## Application Integration

### Web Scraping

#### Scrapy (Python)

```python
# settings.py
DOWNLOADER_MIDDLEWARES = {
    'scrapy.downloadermiddlewares.httpproxy.HttpProxyMiddleware': 110,
}

# In your spider
class MySpider(scrapy.Spider):
    name = 'myspider'

    def start_requests(self):
        urls = ['https://example.com']
        for url in urls:
            yield scrapy.Request(
                url,
                meta={
                    'proxy': 'http://localhost:8891',  # Netherlands
                },
                headers={
                    'X-Exit-Country': 'nl'  # If using header-based
                }
            )
```

#### Puppeteer (Node.js)

```javascript
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch({
    args: ['--proxy-server=localhost:8891']  // Netherlands
  });

  const page = await browser.newPage();
  await page.goto('https://ifconfig.me');

  const content = await page.content();
  console.log(content);

  await browser.close();
})();
```

### API Testing

#### Rotating Countries

```python
import requests
import random

COUNTRIES = ['nl', 'us', 'de', 'uk', 'fr', 'es']
PROXY = {'http': 'http://localhost:8888', 'https': 'http://localhost:8888'}

def fetch_with_random_country(url):
    country = random.choice(COUNTRIES)
    headers = {'X-Exit-Country': country}

    response = requests.get(url, proxies=PROXY, headers=headers)
    return response, country

# Use in your tests
response, country = fetch_with_random_country('https://api.example.com/data')
print(f"Fetched from {country}: {response.status_code}")
```

### Docker Applications

#### Using GeoFlux from Another Container

```yaml
# docker-compose.yml
version: '3.9'

services:
  my-app:
    image: my-application
    environment:
      - HTTP_PROXY=http://haproxy:8888
      - HTTPS_PROXY=http://haproxy:8888
      - EXIT_COUNTRY=nl
    networks:
      - geoflux_vpn-network

networks:
  geoflux_vpn-network:
    external: true
```

### Environment Variables

Many applications respect proxy environment variables:

```bash
# System-wide
export HTTP_PROXY=http://localhost:8891
export HTTPS_PROXY=http://localhost:8891

# Then any application will use the proxy
curl https://ifconfig.me
python my_script.py
npm install
```

## Advanced Usage

### Load Balancing Across Countries

```python
import requests
from itertools import cycle

# Port-based
ports = cycle([8891, 8892, 8893])  # NL, US, DE

for i in range(10):
    port = next(ports)
    proxy = {'http': f'http://localhost:{port}',
             'https': f'http://localhost:{port}'}
    response = requests.get('https://ifconfig.me', proxies=proxy)
    print(f"Request {i} via port {port}: {response.text}")
```

### Retry Logic with Country Fallback

```python
import requests

COUNTRIES = ['nl', 'us', 'de', 'uk']
PROXY = {'http': 'http://localhost:8888', 'https': 'http://localhost:8888'}

def fetch_with_fallback(url):
    for country in COUNTRIES:
        try:
            headers = {'X-Exit-Country': country}
            response = requests.get(url, proxies=PROXY, headers=headers, timeout=10)

            if response.status_code == 200:
                return response

        except requests.RequestException as e:
            print(f"Failed with {country}: {e}")
            continue

    raise Exception("All countries failed")

# Use it
response = fetch_with_fallback('https://api.example.com/data')
print(response.json())
```

### Per-Request Country Selection

```javascript
// Express.js middleware
const axios = require('axios');

app.use(async (req, res, next) => {
  // Get country from request parameter
  const country = req.query.country || 'nl';

  req.proxy = {
    host: 'localhost',
    port: 8888,
    headers: {
      'X-Exit-Country': country
    }
  };

  next();
});

// Use in route
app.get('/fetch', async (req, res) => {
  try {
    const response = await axios.get('https://api.example.com/data', {
      proxy: req.proxy.proxy,
      headers: req.proxy.headers
    });

    res.json(response.data);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

## Testing Examples

### Verify All Countries Work

```bash
#!/bin/bash

COUNTRIES=("nl" "us" "de" "no" "uk" "ee" "lv" "se" "fi" "ch" "fr" "es" "it" "ca" "au")

for country in "${COUNTRIES[@]}"; do
    echo -n "Testing $country: "
    ip=$(curl -s -x http://localhost:8888 -H "X-Exit-Country: $country" https://ifconfig.me)
    echo "$ip"
done
```

### Check Response Times

```bash
#!/bin/bash

PORTS=(8891 8892 8893 8894 8895)

for port in "${PORTS[@]}"; do
    echo -n "Port $port: "
    time curl -s -x http://localhost:$port https://ifconfig.me > /dev/null
done
```

## Common Patterns

### Session Persistence

Maintain the same country for a user session:

```python
from flask import Flask, session
import requests

app = Flask(__name__)
app.secret_key = 'your-secret-key'

@app.route('/api/<path:path>')
def proxy_api(path):
    # Use same country for entire session
    if 'country' not in session:
        session['country'] = 'nl'  # Default

    proxies = {'http': 'http://localhost:8888', 'https': 'http://localhost:8888'}
    headers = {'X-Exit-Country': session['country']}

    response = requests.get(f'https://api.example.com/{path}',
                          proxies=proxies,
                          headers=headers)

    return response.json()

@app.route('/set-country/<country>')
def set_country(country):
    session['country'] = country
    return f'Country set to {country}'
```

### Concurrent Requests

```python
import asyncio
import aiohttp

async def fetch_url(session, url, country):
    proxy = 'http://localhost:8888'
    headers = {'X-Exit-Country': country}

    async with session.get(url, proxy=proxy, headers=headers) as response:
        return await response.text()

async def main():
    urls = ['https://api.example.com/endpoint1',
            'https://api.example.com/endpoint2',
            'https://api.example.com/endpoint3']

    countries = ['nl', 'us', 'de']

    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url, country)
                for url, country in zip(urls, countries)]

        results = await asyncio.gather(*tasks)
        return results

results = asyncio.run(main())
print(results)
```

## Monitoring Usage

### Log Requests by Country

```python
import requests
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def proxied_request(url, country):
    proxies = {'http': 'http://localhost:8888', 'https': 'http://localhost:8888'}
    headers = {'X-Exit-Country': country}

    logger.info(f"Request to {url} via {country}")

    try:
        response = requests.get(url, proxies=proxies, headers=headers)
        logger.info(f"Response: {response.status_code}")
        return response
    except Exception as e:
        logger.error(f"Error: {e}")
        raise
```

## Troubleshooting

### Connection Refused

```bash
# Check if HAProxy is running
docker compose ps haproxy

# Check if port is accessible
nc -zv localhost 8888
```

### Slow Responses

```bash
# Test direct connection
time curl https://ifconfig.me

# Test through proxy
time curl -x http://localhost:8891 https://ifconfig.me
```

### Wrong Country

```bash
# Verify location
curl -x http://localhost:8891 https://ipapi.co/json/ | jq '.country'
```
