# HAProxy + Cloudflare Tunnel: Zero Port Forwarding Powerhouse

## 🎯 What This Does

🌐 ha1.yourdomain.com → Multiple web servers (load balanced)  
🎮 mcw.yourdomain.com → TCP server (port 25565, no client port needed via SRV)  
🔒 Zero public ports exposed  
✅ Automatic HTTPS via Cloudflare Free  
⚡ Health checks + automatic failover  

### Why This Setup Rocks for Your Use Case

✅ Zero ports exposed (no firewall rules needed)  
✅ Free HTTPS from Cloudflare (no cert management)  
✅ HAProxy still does load balancing (your backends unchanged)  
✅ Cloudflare DDoS protection automatically  
✅ Works with your ISP dynamic IP (tunnel finds your server) 

### Traffic Flow Breakdown

1. User types: https://example.domain.com  
2. Cloudflare DNS resolves → CNAME → your-tunnel-uuid.cfargotunnel.com  
3. Cloudflare Edge → establishes secure tunnel → your Ubuntu server (cloudflared)  
4. cloudflared (per config.yml) → forwards to http://localhost:80  
5. HAProxy frontend http-in → load balances → backend web servers (192.168.1.10:80, etc.)  
6. Web servers respond → HAProxy → cloudflared → Cloudflare → User (HTTPS)  

## 📋 Prerequisites

Ensure these are ready before starting:

### HAProxy  

- OS: Ubuntu 24.04.4 LTS or newer (preferred for latest packages).  
- Hardware: 1 vCPU, 1-2GB RAM, 10GB+ storage; scales with traffic (e.g., 4GB+ RAM for high-load balancing).  
- Network: Open ports 80/443 (HTTP/HTTPS), static IP; firewall rules via UFW (e.g., ufw allow 80).
- Dependencies: None beyond apt; root/sudo access required

### Cloudflared

- Ubuntu server with sudo access.  
- Cloudflare account with your domain added/managed (free tier works).  
- Domain ready: Add a CNAME record later (e.g., tunnel.example.com → {UUID}.cfargotunnel.com).  

## HAProxy Setup

### Installation  

>For the latest official PPA version, check on https://haproxy.debian.net  

1. Add Official PPA: ` sudo add-apt-repository ppa:vbernat/haproxy-3.3 `
2. Update packages: ` sudo apt update && sudo apt upgrade -y `
3. Install HAProxy: ` sudo apt install haproxy -y `
4. Verify: ` haproxy -v ` (shows version, e.g., 3.2+ on Ubuntu 24.04.4)  


### Configuration

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

### Test & Restart

- Test config:
  - ` sudo haproxy -c -f /etc/haproxy/haproxy.cfg `
- Run config on startup:
  - Enable on boot: ` sudo systemctl enable haproxy `.
  - Restart: ` sudo systemctl restart haproxy `.
- Verify config:
  - Check status: ` sudo systemctl status haproxy `.
  - sudo ss -tlnp | grep haproxy  (Should show :80 and :8404)

## Cloudflared Tunnel Setup

### Installation 

1. Add cloudflare gpg key
```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-public-v2.gpg | sudo tee /usr/share/keyrings/cloudflare-public-v2.gpg >/dev/null
```
2. Add this repo to your apt repositories
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
7. Test run: ` cloudflared tunnel --config /etc/cloudflared/config.yml run `

### Test Tunnel

Test the configured tunnel
```bash
cloudflared tunnel --config /etc/cloudflared/config.yml run example-tunnel
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

sudo cloudflared service install /etc/cloudflared/config.yml  
sudo systemctl start cloudflared  
sudo systemctl enable cloudflared  
sudo systemctl status cloudflared  

cloudflared service install (uses config.yml)  
nano 
Enable/start: sudo systemctl enable cloudflared && sudo systemctl start cloudflared. 
Check status: sudo systemctl status cloudflared 
logs: journalctl -u cloudflared -f

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


