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

## HAProxy Installation Steps

>For the latest official PPA version, check on https://haproxy.debian.net  

1. Add Official PPA: ` sudo add-apt-repository ppa:vbernat/haproxy-3.3 `
2. Update packages: ` sudo apt update && sudo apt upgrade -y `
3. Install HAProxy: ` sudo apt install haproxy -y `
4. Verify: ` haproxy -v ` (shows version, e.g., 3.2+ on Ubuntu 24.04.4)  


## HAProxy Configuration

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

- To test config:
  - ` sudo haproxy -c -f /etc/haproxy/haproxy.cfg `
- To run config on startup:
  - Enable on boot: ` sudo systemctl enable haproxy `.
  - Restart: ` sudo systemctl restart haproxy `.
- To verify config:
  - Check status: ` sudo systemctl status haproxy `.
  - sudo ss -tlnp | grep haproxy  (Should show :80 and :8404)

## Cloudflared Tunneling Configuration




## Installation

```bash
sudo apt update
sudo apt install openjdk-21-jdk
```

> Make sure Java is installed first.

## Checklist
- [x] Install Java
- [ ] Download server jar
- [ ] Configure firewall
