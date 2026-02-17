# HAProxy + Cloudflare Tunnel: 🚀 Zero Port Forwarding Powerhouse

## 🎯 What This Beast Does

### Key Wins:

 - 🔒 Zero public ports exposed (bye-bye firewall rules!)  
 - ✅ Free HTTPS from Cloudflare (no cert nightmares)  
 - ⚡ HAProxy load balancing (backends unchanged)  
 - 🛡️ Cloudflare DDoS protection (automatic)  
 - 🌐 Dynamic IP proof (tunnel always finds your server)  

### 🔄 Traffic Flow (The Magic Happens Here)

```yaml
User → https://yourdomain.com
     ↓
Cloudflare DNS → CNAME → your-tunnel-uuid.cfargotunnel.com
     ↓
Cloudflare Edge → Secure Tunnel → Ubuntu (cloudflared)
     ↓
cloudflared → http://localhost:80
     ↓
HAProxy → Load Balances → Backend Servers (192.168.1.10:80, etc.)
     ↓
Response → HAProxy → cloudflared → Cloudflare → User (HTTPS ✨)

``` 

## 📋 Prerequisites Checklist

Ensure these are ready before starting:

### HAProxy Requirements

- OS: Ubuntu 24.04.4 LTS+ (latest packages) 
- Hardware: 1 vCPU, 1-2GB RAM, 10GB+ storage; scales with traffic (e.g., 4GB+ RAM for high-load balancing).  
- Network: Ports 80/443 open (UFW: ufw allow 80)
- Dependencies: Root/sudo access

### Cloudflared Requirements

- Ubuntu server + sudo
- Cloudflare account (FREE tier works!)
- Domain added to Cloudflare

## HAProxy Setup (Load Balancing Foundation)

### Install Latest HAProxy (PPA Method) 

1. Add Official PPA: ` sudo add-apt-repository ppa:vbernat/haproxy-3.3 `
> For the latest official PPA version, check on https://haproxy.debian.net  
2. Update packages: ` sudo apt update && sudo apt upgrade -y `
3. Install HAProxy: ` sudo apt install haproxy -y `
> Verify installation  
4. Verify: ` haproxy -v ` (shows version, e.g., 3.2+ on Ubuntu 24.04.4)  


### Configure HAProxy
Edit ` /etc/haproxy/haproxy.cfg ` with ` sudo nano /etc/haproxy/haproxy.cfg `.  
Here's a simple HTTP load-balancing example for two backend web servers (replace IPs/ports):  

```bash
global
    log /dev/log local0
    maxconn 4096
    user haproxy
    group haproxy

defaults
    log global
    mode http
    option httplog
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:80
    default_backend webservers

backend webservers
    balance roundrobin
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
```

### Test & Deploy

- Test config:
  - ` sudo haproxy -c -f /etc/haproxy/haproxy.cfg `
- Run config on startup:
  - Enable on boot: ` sudo systemctl enable haproxy `.
  - Restart: ` sudo systemctl restart haproxy `.
- Verify config:
  - Check status: ` sudo systemctl status haproxy `.
  - sudo ss -tlnp | grep haproxy  (Should show :80)

## Cloudflared Tunnel Setup

### Installation 

1. Add cloudflare gpg key
```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null
```
2. Add this repo to apt repositories
```bash
echo 'deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] https://pkg.cloudflare.com/cloudflared any main' | sudo tee /etc/apt/sources.list.d/cloudflared.list
```
3. Install Cloudflare
```bash  
sudo apt-get update && sudo apt-get install cloudflared
```
4. Check version
```bash
cloudflared version
```

### Authentication

1. Login to Cloudflare tunnel  
```bash
cloudflared tunnel login
```
2. Open the provided link to the browser, login and authorize
3. Once the Cloudflare Account been authorized, the ` cert.pem ` will appear at ` /root/.cloudflared/ `

### Create Tunnel

1. Create the tunnel
```bash
cloudflared tunnel create example-tunnel
```
2. json file will be created in ` /root/.cloudflared/ `. (eg., {UUID}.json)
3. Copy the UUID (e.g., a1b2c3d4-c3d4...)
4. To check tunnel ` cloudflared tunnel list `

### Configure Tunnel

1. Create file
```bash
sudo nano /root/.cloudflared/config.yml
```
2. Create Cloudflare config file
```yml
tunnel: <your-tunnel-UUID>
credentials-file: /root/.cloudflared/<your-tunnel-UUID>.json

ingress:
  - hostname: yourdomain.com
    service: https://localhost:443
  - service: http_status:404
```
6. Route DNS: ` cloudflared tunnel route dns my-haproxy-tunnel yourdomain.com `
7. Test run: ` cloudflared tunnel --config /root/.cloudflared/config.yml run `

### Test Tunnel

Test the configured tunnel
```bash
cloudflared tunnel --config /root/.cloudflared/config.yml run example-tunnel
```

## Cloudflared DNS Setup

### Cloudflare Dashboard

1. Login → Your Domain → DNS tab
2. Add Record:
   - Type: CNAME
   - Name: example (creates example.domain.com)
   - Target: YOUR-UUID-HERE.cfargotunnel.com
   - Proxy status: Proxied (orange cloud ⚡)
   - Save

## Run as Production Service

1.cloudflared service install    
2. nano /etc/systemd/system/cloudflared.service  
3. change line from ` /etc/cloudflared/config.yml ` to ` /root/.cloudflared/config.yml `  
4. ` systemctl daemon-reload `  
5. Enable/start: ` sudo systemctl enable cloudflared && sudo systemctl start cloudflared `  
6. Check status: ` sudo systemctl status cloudflared `  
7. logs: ` journalctl -u cloudflared -f `  

## Final Testing

Test Domain
```Bash
curl -I https://example.domain.com
```
Test TCP Connection
```bash
nc -zv 172.20.20.201 80
telnet 127.0.0.1 80
```

### File Locations (Reference)

```yml
/root/.cloudflared/
├── cert.pem
└── YOUR-UUID.json
└── config.yml

/etc/haproxy/
└── haproxy.cfg
```

## Monitoring Commands

sudo ss -tlnp | grep haproxy  

Logs  
sudo journalctl -u haproxy -f  
sudo journalctl -u cloudflared -f  


