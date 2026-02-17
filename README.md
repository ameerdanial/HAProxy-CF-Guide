# HAProxy + Cloudflare Tunnel: 🚀 Zero Port Forwarding Powerhouse

### Transform your Ubuntu server into a production-grade, zero-port-exposure powerhouse that:

 - Load balances HTTP traffic across multiple web servers
 - Routes domain-specific traffic (web, gaming, custom apps)
 - Provides automatic HTTPS + DDoS protection via Cloudflare
 - Eliminates firewall port forwarding completely
 - Works with dynamic IPs (ISP changes? No problem!)
 - Scales from homelab to production with HAProxy health checks  

Perfect for: Homelabs, small businesses, game servers, or anyone tired of port forwarding nightmares.

## 🎯 What This Beast Does

| Component                | Subdomain/Service  | Magic Delivered                                                    |
| ------------------------ | ------------------ | ------------------------------------------------------------------ |
| 🌐 HAProxy Load Balancer | ha1.yourdomain.com | Round-robin to web servers (192.168.1.10:80, etc.) + health checks |
| ☁️ Cloudflare DNS        | All subdomains     | CNAME → tunnel UUID + automatic HTTPS + DDoS protection            |
| 🔄 Domain Routing        | Any hostname       | config.yml ingress rules route to specific ports/services          |
| 🛡️ Zero Port Exposure   | Entire server      | Cloudflare Tunnel = no public ports open                           |
| ⚡ Dynamic IP Proof       | Any IP change      | Tunnel finds your server automatically                             |
| ✅ Auto Failover          | Backend servers    | HAProxy health checks remove dead servers                          | 

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
- Hardware: 1 vCPU, 1-2GB RAM, 10GB+ storage (scales with traffic. e.g., 4GB+ RAM for high-load balancing)  
- Network: Ports 80/443 open (UFW: ufw allow 80)
- Dependencies: Root/sudo access

### Cloudflared Requirements

- Ubuntu server + sudo
- Cloudflare account (FREE tier works!)
- Domain added to Cloudflare

## 🛠️ HAProxy Setup (Load Balancing Foundation)

### Installation (PPA Method) 

1. Add Official PPA
```bash
sudo add-apt-repository ppa:vbernat/haproxy-3.3
```
> For the latest official PPA version, check on https://haproxy.debian.net  
2. Update packages
```bash
sudo apt update && sudo apt upgrade -y
```
3. Install HAProxy
```bash
sudo apt install haproxy -y
```
4. Verify Installation
```bash
haproxy -v #(Should show version, e.g., 3.2+ on Ubuntu 24.04.4)
```

### Configuration
Edit HAProxy config file  
```bash
sudo nano /etc/haproxy/haproxy.cfg
```

Here's a simple HTTP load-balancing example for two backend web servers (replace IPs/ports):  

```yml
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

1. Test config:
```bash  
sudo sudo haproxy -c -f /etc/haproxy/haproxy.cfg
```
2. Start the service automatically on system boot
```bash
sudo systemctl enable haproxy && sudo systemctl start haproxy
```
3. Check service status
```bash
sudo systemctl status haproxy
```
4. sudo ss -tlnp | grep haproxy # Should show :80

## 🌥️ Cloudflared Tunnel Setup (The Zero-Port Magic)

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
cloudflared tunnel create my-haproxy-tunnel
```
2. json file will be created in ` /root/.cloudflared/ `. (eg., {UUID}.json)
3. Copy the UUID (e.g., a1b2c3d4-c3d4...)
4. To verify the tunnel
```bash
cloudflared tunnel list
```

### Configure Tunnel

1. Create file
```bash
sudo nano /root/.cloudflared/config.yml
```
2. Cloudflare config file
```yml
tunnel: <your-tunnel-UUID>
credentials-file: /root/.cloudflared/<your-tunnel-UUID>.json

ingress:
  - hostname: yourdomain.com
    service: https://localhost:443
  - service: http_status:404
```

### Route DNS & Test Tunnel

1. Route DNS
```bash
cloudflared tunnel route dns my-haproxy-tunnel yourdomain.com
```
2. Test run
```bash
cloudflared tunnel --config /root/.cloudflared/config.yml run
```

## ☁️ Cloudflared DNS Setup

### Cloudflare Dashboard

1. Login → Your Domain → DNS tab
2. Add Record:
   - Type: CNAME
   - Name: yourdomain (creates example.domain.com)
   - Target: YOUR-UUID-HERE.cfargotunnel.com
   - Proxy status: Proxied (orange cloud ⚡)
 3. Save

## 🔄 Production Service (Set & Forget)

1. Install systemd service
```bash
cloudflared service install  
```
2. Fix config path
```bash
sudo nano /etc/systemd/system/cloudflared.service
```
3. Change the file path from ` /etc/cloudflared/config.yml ` to ` /root/.cloudflared/config.yml ` 
4. Update systemd manager configuration
```bash
systemctl daemon-reload
```
5. Start the service automatically on system boot
```bash
sudo systemctl enable cloudflared && sudo systemctl start cloudflared
```
6. Check service status
```bash
sudo systemctl status cloudflared
```
7. Check service logs
```bash
journalctl -u cloudflared -f
```

## 🧪 Final Testing

Test Domain
```Bash
curl -I yourdomain.com
```
Local connectivity Test
```bash
nc -zv 127.0.0.1 80
telnet 127.0.0.1 80
```

### 📁 File Structure

```yml
/root/.cloudflared/
├── cert.pem
└── YOUR-UUID.json
└── config.yml

/etc/haproxy/
└── haproxy.cfg
```

## 🔍 Monitoring Commands

1. Listening ports
```Bash
sudo ss -tlnp | grep haproxy
```
2. Live Logs
```Bash
sudo journalctl -u haproxy -f
sudo journalctl -u cloudflared -f
```


