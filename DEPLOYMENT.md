# Deployment notes

This document outlines my preferences for web app deployments. Modify as necessary for your particular application and environment.

Assume it is a Flask app called “certgen”. The templates below work on Ubuntu Linux 24.04.

## Service user

* Every service should have its own user (not just roll under www-data), this enables isolation in case of trouble.
* Home should be in /home/certgen, no shell
* Need to make a .ssh under this account, create ssl key
* Add as deployment key to github

## Directory layout

```
/opt/certgen/
├── code/                   # read-only (git clone)
├── venv/                   # read-only
├── config/                 # read-only (API keys, settings)
└── data/                   # read-write (SQLite)

/var/log/certgen/           # read-write logs

/home/certgen/.ssh/         # SSH keys
```

## nginx

nginx config should have a port 80 section redirecting to ssl

```
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on; 
    
    server_name certgen.your-domain.com;

    # --- SSL ---
    ssl_certificate /etc/nginx/ssl/certgen.crt;
    ssl_certificate_key /etc/nginx/ssl/certgen.key;
    # Let the client choose the cipher (better for mobile battery life)

    # --- Logging ---
    access_log /var/log/nginx/certgen-access.log;
    error_log /var/log/nginx/certgen-error.log;

    # --- Global Proxy Settings (Inherited by all locations) ---
    proxy_http_version 1.1; # Crucial for optimization
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
   
    location / {
        proxy_pass http://127.0.0.1:8000;
    }
    # API (Long running tasks)
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        
        # Only override what is different
        proxy_read_timeout 180s;
        proxy_connect_timeout 90s;
        proxy_send_timeout 90s;
    }
}
```

## systemd

```
[Unit]
Description=Certgen SSL Certificate Portal
After=network.target


[Service]
# --- Application Configuration ---
Type=notify
User=certgen
Group=certgen
WorkingDirectory=/opt/certgen/code
# Ensure logs output instantly to systemd
Environment=PYTHONUNBUFFERED=1
Environment="PATH=/opt/certgen/venv/bin:/usr/bin:/bin"
EnvironmentFile=/opt/certgen/config/.env

# --- Log Tagging ---
# Makes logs show as "certgen[PID]" instead of "gunicorn[PID]"
SyslogIdentifier=certgen
LogsDirectory=certgen
StandardOutput=append:/var/log/certgen/certgen.log
StandardError=append:/var/log/certgen/error.log

# --- Execution ---
ExecStart=/opt/certgen/venv/bin/gunicorn --bind 127.0.0.1:5001 --workers 4 --threads 2 --timeout 30 app:app

# --- Runtime Directories ---
# Automatically creates /run/certgen/sessions owned by the User
RuntimeDirectory=certgen/sessions
RuntimeDirectoryMode=0700

# --- Stability & Resource Control ---
Restart=always
# Wait a bit before restarting to prevent CPU spinning
RestartSec=5
# If app leaks memory, throttle/kill it before it crashes the whole server
MemoryHigh=1G
MemoryMax=1.2G

# --- Security Hardening (The "Free" Security Layer) ---
NoNewPrivileges=true
PrivateTmp=true
# Mount /usr, /boot, and /etc as Read-Only
ProtectSystem=strict
# Hide /home and /root
ProtectHome=true
# Prevent app from tuning kernel parameters
ProtectKernelTunables=true
ProtectControlGroups=true


# --- Write Permissions ---
# Because ProtectSystem=strict is on, you must explicitly whitelist write targets
# Add your SQLite DB path here if applicable
ReadWritePaths=/opt/certgen/data /run/certgen/sessions

[Install]
WantedBy=multi-user.target
```

## Database backup
