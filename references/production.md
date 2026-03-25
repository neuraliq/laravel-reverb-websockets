# Production Deployment

## Supervisor Configuration

```ini
[program:reverb]
command=php /var/www/html/artisan reverb:start --host=0.0.0.0 --port=8080
directory=/var/www/html
user=www-data
autostart=true
autorestart=true
stopwaitsecs=10
redirect_stderr=true
stdout_logfile=/var/log/supervisor/reverb.log
stdout_logfile_maxbytes=10MB
stdout_logfile_backups=3
```

**Full stack Supervisor layout:**

```ini
[group:laravel]
programs=octane,horizon,reverb

[program:octane]
command=php /var/www/html/artisan octane:start --host=0.0.0.0 --port=8000
# ... octane config

[program:horizon]
command=php /var/www/html/artisan horizon
# ... horizon config

[program:reverb]
command=php /var/www/html/artisan reverb:start --host=0.0.0.0 --port=8080
# ... reverb config
```

## Nginx Reverse Proxy with SSL

```nginx
# WebSocket server (Reverb)
upstream reverb {
    server 127.0.0.1:8080;
}

server {
    listen 443 ssl http2;
    server_name ws.example.com;

    ssl_certificate /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    location / {
        proxy_pass http://reverb;
        proxy_http_version 1.1;

        # WebSocket upgrade headers (critical)
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts for long-lived connections
        proxy_connect_timeout 7d;
        proxy_send_timeout 7d;
        proxy_read_timeout 7d;
    }
}
```

### Same-Domain Setup (Single nginx)

If WebSocket runs on the same domain (recommended for simpler CORS):

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # Regular app traffic → Octane
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket traffic → Reverb
    location /app {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 7d;
    }
}
```

## SSL/TLS Configuration

When using SSL termination at nginx:

```env
# Server-side: Reverb listens on plain HTTP
REVERB_HOST=0.0.0.0
REVERB_PORT=8080
REVERB_SCHEME=http

# Client-side: connects via nginx HTTPS
VITE_REVERB_HOST=ws.example.com
VITE_REVERB_PORT=443
VITE_REVERB_SCHEME=https
```

When Reverb handles TLS directly:

```env
REVERB_HOST=0.0.0.0
REVERB_PORT=8080
REVERB_SCHEME=https
REVERB_TLS_CERTIFICATE_PATH=/etc/ssl/certs/example.com.pem
REVERB_TLS_KEY_PATH=/etc/ssl/private/example.com.key
```

## Authentication with Sanctum

```php
// config/broadcasting.php
'connections' => [
    'reverb' => [
        'driver' => 'reverb',
        'key' => env('REVERB_APP_KEY'),
        'secret' => env('REVERB_APP_SECRET'),
        'app_id' => env('REVERB_APP_ID'),
        'options' => [
            'host' => env('REVERB_HOST'),
            'port' => env('REVERB_PORT', 443),
            'scheme' => env('REVERB_SCHEME', 'https'),
        ],
    ],
],
```

```typescript
// Echo with Sanctum auth
window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT,
    wssPort: import.meta.env.VITE_REVERB_PORT,
    forceTLS: true,
    enabledTransports: ['ws', 'wss'],
    // Sanctum authentication
    authorizer: (channel: any) => ({
        authorize: (socketId: string, callback: Function) => {
            axios.post('/broadcasting/auth', {
                socket_id: socketId,
                channel_name: channel.name,
            })
            .then(response => callback(null, response.data))
            .catch(error => callback(error))
        }
    }),
})
```

## Deployment Checklist

1. Reverb runs as a separate Supervisor process
2. nginx proxies WebSocket connections with `Upgrade` headers
3. SSL terminated at nginx (recommended) or at Reverb
4. `VITE_REVERB_*` env vars point to the public-facing host/port
5. `routes/channels.php` has authorization for all private/presence channels
6. Broadcasting queue runs in Horizon (events are queued by default)
7. Firewall allows traffic on the WebSocket port
8. Health check monitors WebSocket connection status
