# Supercharge Your Homelab: A Deep Dive into HAProxy and Cloudflare

Tired of juggling multiple IPs and ports for your self-hosted services? Want to add a professional, secure, and resilient layer to your homelab or small business infrastructure? Let's walk through how to build a powerful reverse proxy using HAProxy, enhanced with the security and performance of Cloudflare.

This isn't just a basic install guide. We'll build a modular, secure, and high-performance setup that's easy to manage and scale as you add more services.

Here's the game plan:

*     Install HAProxy on a Debian-based system (like Ubuntu).
*     Reconfigure HAProxy for a modular conf.d setup, preventing a monolithic and messy config file.
*     Establish a robust base configuration with security, logging, and caching.
*     Integrate with Cloudflare for TLS termination, DDOS protection, and hiding your origin IP.
*     Lock down access to ensure only legitimate, proxied traffic reaches your services.

Let's get started!

**1. Laying the Foundation: HAProxy Installation**

First, we'll install HAProxy using the system's package manager. This is the simplest way to get the software onto your server.
```

sudo apt update
sudo apt install haproxy -y
```
With HAProxy installed, we can now shape it to our needs. By default, it uses a single, monolithic /etc/haproxy/haproxy.cfg file. For a complex environment, this becomes a nightmare to manage. We'll fix that.

**2. Going Modular: The conf.d Approach**

A modular configuration allows you to define each service or backend in its own file. Adding or removing a service is as simple as adding or deleting a small config file, rather than editing one giant one.

**Create the Configuration Directory**

First, create the directory that will hold our modular configuration files.

```
sudo mkdir -p /etc/haproxy/conf.d
```

**Teach systemd About Our New Structure**

Next, we need to tell HAProxy's service manager, systemd, to load configurations from both the main haproxy.cfg file and our new conf.d directory.

Open the service file for editing:

```
sudo systemctl edit --full haproxy.service
```

Now, modify the [Service] section. We are specifically updating the ExecStart and ExecReload lines to use the -f flag twice: once for the main config file and once for our new directory.

```
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
Documentation=file:/usr/share/doc/haproxy/configuration.txt.gz
After=network-online.target rsyslog.service
Wants=network-online.target

[Service]
EnvironmentFile=-/etc/default/haproxy
EnvironmentFile=-/etc/sysconfig/haproxy
# We tell haproxy to load our main config AND our new directory
ExecStart=/usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/conf.d/ -p /run/haproxy.pid $EXTRAOPTS
ExecReload=/usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/conf.d/ -c -q $EXTRAOPTS
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed
Restart=always
SuccessExitStatus=143
Type=notify

[Install]
WantedBy=multi-user.target
```

After saving the file, we must reload the systemd daemon to apply our changes and then restart HAProxy.

```
sudo systemctl daemon-reload
sudo systemctl restart haproxy
```

🚨 *Note: The restart will likely fail right now because we haven't created any configuration files yet. That's perfectly normal! We'll fix it in the next step.*

**3. The Core Configuration: haproxy.cfg**

This file, **/etc/haproxy/haproxy.cfg**, will contain all our global settings and defaults. It sets the stage for how HAProxy behaves overall, while the individual service configs live in conf.d.

Here is a well-tuned starting point.
Code snippet

```
# /etc/haproxy/haproxy.cfg

global
    # --- Security & Performance ---
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 50000
		
    # --- SSL/TLS Hardening ---
    # Modern cipher suite for strong security (based on Mozilla's intermediate compatibility)
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    tune.ssl.default-dh-param 2048

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    # Timeouts should be reasonable to avoid dropping long-running connections
    timeout connect 10s
    timeout client  1m
    timeout server  1m
    option  http-keep-alive
    # Use 'option forwardfor' in backends instead of http-server-close for better IP forwarding
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

# HAProxy Stats & Monitoring Page
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    # IMPORTANT: Change this password!
    stats auth admin:YourSuperSecretPasswordHere

# In-Memory Cache for Static Assets
cache static-cache
    total-max-size 256    # Cache size in megabytes
    max-age 240           # Maximum age of cached objects in seconds (4 minutes)
```

**Key Improvements Explained**

Security (global): chroot jails the HAProxy process to a directory, limiting potential damage if compromised. The ssl-default-bind-* directives enforce modern, secure TLS protocols and ciphers, protecting your traffic with strong encryption.

Logging (global, defaults): We configure HAProxy to log to the system's journal (journald) for easy viewing with journalctl -u haproxy. option httplog provides rich, detailed logs for web traffic.
Stats Page (frontend stats): This is invaluable. It creates a web dashboard on port 8404 (accessible at http://<haproxy-ip>:8404/stats) that shows the health of all your frontends and backends in real-time. Remember to set a strong password!
	
Caching (cache static-cache): This sets up a 256 MB in-memory cache. While not used yet, it can be attached to backends later to cache static files (.css, .js, images), reducing load on your backend services and speeding up response times.

**4. The Grand Entrance: Frontend Configuration**

The frontend is the public face of our proxy. It listens for incoming traffic, inspects it, and decides where it should go. We'll place this in its own file.

File: /etc/haproxy/conf.d/00-frontend.cfg

```
frontend https_frontend
    # Listen on port 443 for HTTPS traffic, using our Cloudflare Origin Certificate
    # alpn h2,http/1.1 enables HTTP/2 for clients that support it, improving performance
    bind *:443 ssl crt /opt/cloudflare/certs/wildcard/cert.pem alpn h2,http/1.1

    # --- Cloudflare Integration: The Golden Rules ---
    # 1. Restore the REAL client IP address from the Cloudflare header
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    http-request set-src req.hdr(CF-Connecting-IP)

    # 2. Define an ACL to match requests coming ONLY from Cloudflare's IP ranges
    acl cloudflare_ips src -f /etc/haproxy/cloudflare_ips.lst

    # 3. Block any direct traffic that doesn't come from Cloudflare
    http-request deny unless cloudflare_ips

    # --- Content-Based Routing ---
    # Define ACLs to match requested hostnames (case-insensitive)
    acl host_netdata hdr(host) -i netdata.cudos.org
    acl host_graylog hdr(host) -i graylog.cudos.org

    # Route traffic to the correct backend based on the hostname ACL
    use_backend netdata_backend if host_netdata
    use_backend graylog_backend if host_graylog

    # A fallback backend for any requests that don't match our services
    default_backend default_backend
```

**Deeper Dive into the Frontend Logic**

    **TLS Termination:** The bind *:443 ssl crt ... line is where HAProxy handles the encryption. It decrypts the incoming HTTPS traffic so it can read the HTTP headers (like the hostname) for routing.

    **http-request set-src req.hdr(CF-Connecting-IP):** This is CRITICAL. When you use Cloudflare's proxy, your server's logs will show all traffic coming from Cloudflare's IPs. This line reads the CF-Connecting-IP header that Cloudflare adds and tells HAProxy to treat that as the true source IP. Your backend application logs will now show the real visitor's IP address.

   ** http-request deny unless cloudflare_ips**: This is our security linchpin. It enforces that HAProxy will only accept traffic from IP addresses on our approved list. This prevents anyone from bypassing Cloudflare and hitting your server directly, keeping your origin IP address safe and ensuring all traffic is filtered by Cloudflare's WAF and DDOS protection.

    ACLs and use_backend: This is the core of a reverse proxy. Access Control Lists (acl) define conditions. We create a condition for each hostname we want to serve. The use_backend directive then acts on these conditions, sending a request for netdata.cudos.org to the netdata_backend, and so on.

**5. The Destinations: Backend Configurations**

Backends define where the traffic goes. Each service gets its own backend configuration file in our modular setup.

**Graylog Backend**

File: /etc/haproxy/conf.d/01-graylog.cfg

```
backend graylog_backend
    mode http
    balance roundrobin
    # Forward the original client's IP to the backend server
    option forwardfor
    server graylog_server 100.67.124.62:9000 check inter 5s fall 3 rise 2

backend default_backend
    # Redirect any non-matched traffic to a safe default location
    http-request redirect location https://cudos.org
```
	
**Netdata Backend**

File: /etc/haproxy/conf.d/02-netdata.cfg

```
backend netdata_backend
    mode http
    balance roundrobin
    option forwardfor
    server netdata_server 100.67.253.39:19999 check inter 5s fall 3 rise 2
```
	
**Backend Directives Explained**

    mode http: Tells HAProxy to operate at Layer 7, inspecting HTTP traffic.
	
    balance roundrobin: If you had multiple backend servers, this would distribute the load between them evenly.

    option forwardfor: This adds the X-Forwarded-For header to the request sent to the backend, which is another way to preserve the original client's IP address.

    server ... check: This defines the actual backend server. The check parameter tells HAProxy to actively monitor the health of the service. If the server goes down (fall 3), HAProxy will stop sending traffic to it and serve a 503 error until it comes back up (rise 2).

    default_backend: This is a great safety net. If a request comes in for a hostname that doesn't match any of our services, instead of showing an error, we simply redirect them to a main website.

**6. Fortifying the Gates: Cloudflare Integration & IP Whitelisting**

This is where the magic happens. We'll ensure our HAProxy server only talks to Cloudflare, making our security posture incredibly strong.

**Cloudflare Origin Certificates**

Instead of using a public certificate from Let's Encrypt on your HAProxy server, use a Cloudflare Origin Certificate.

    Why? These are free certificates that you install on your origin server (HAProxy) to encrypt traffic between Cloudflare and your server. They are only trusted by the Cloudflare proxy, meaning no one else can impersonate your server.

    How: In your Cloudflare dashboard, go to SSL/TLS -> Origin Server and create a certificate. It will give you a certificate (.pem) and a private key (.key).

    Implementation: Store these files securely on your server. In our example, we've placed them in /opt/cloudflare/certs/wildcard/. Ensure permissions are restrictive (sudo chmod 600 /path/to/your.key). Your SSL/TLS encryption mode in Cloudflare should be set to Full (Strict).

**Creating the IP Whitelist**

HAProxy needs a list of Cloudflare's IP ranges to allow traffic from. Cloudflare publishes these lists. We will also add any trusted IPs (like your home IP) that need to bypass Cloudflare for direct access or testing.

File: /etc/haproxy/cloudflare_ips.lst
```
# My Home IP (for direct access/troubleshooting)
10.23.22.456/32

# --- Cloudflare IPs (as of Q3 2025) ---
# Always check for the latest list at: https://www.cloudflare.com/ips/
# IPv4

103.21.244.0/22
103.22.200.0/22
103.31.4.0/22
104.16.0.0/13
104.24.0.0/14
108.162.192.0/18
131.0.72.0/22
141.101.64.0/18
162.158.0.0/15
172.64.0.0/13
173.245.48.0/20
188.114.96.0/20
190.93.240.0/20
197.234.240.0/22
198.41.128.0/17
```
# You can also add their IPv6 ranges here if your server is IPv6 enabled.

**Final Launch Sequence**

With all our configuration files in place, let's validate them and bring our proxy online.

    Check Configuration Syntax: Before restarting, always run a syntax check. This will prevent downtime by catching typos.

```
haproxy -c -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/conf.d/
```

If all is well, you'll see Configuration file is valid.

Restart and Enable HAProxy:

```
    sudo systemctl restart haproxy
    sudo systemctl enable haproxy  # Make sure it starts on boot
```
	
**Final Thoughts and Next Steps**

You now have a professional-grade reverse proxy that is:

    ✅ Secure: Traffic is encrypted end-to-end, and your origin is shielded from direct attack.

    ✅ Modular: Adding a new service is as easy as dropping a new file in conf.d.

    ✅ Resilient: Health checks automatically route traffic away from failed services.

    ✅ Observable: The stats page gives you a real-time view of your infrastructure's health.

From here, you can expand by adding more backends, implementing rate limiting to prevent abuse, or using the cache to speed up your websites. Happy hosting!