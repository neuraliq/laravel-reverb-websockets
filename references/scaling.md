# Scaling Reverb

## Connection Limits

Default Reverb handles thousands of concurrent connections on a single process. For most applications, a single Reverb instance is sufficient.

### Tuning Connection Limits

```php
// config/reverb.php
'servers' => [
    'reverb' => [
        'host' => env('REVERB_HOST', '0.0.0.0'),
        'port' => env('REVERB_PORT', 8080),
        'max_request_size' => 10_000, // Max message size in bytes
    ],
],
```

### OS-Level Tuning

```bash
# Increase file descriptor limits (each connection = 1 fd)
# /etc/security/limits.conf
www-data soft nofile 65536
www-data hard nofile 65536

# Kernel parameters
# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535

# Apply
sudo sysctl -p
```

## Horizontal Scaling with Redis

For multiple Reverb instances behind a load balancer, use Redis as the pub/sub backend so events reach all connected clients regardless of which instance they're connected to.

```php
// config/reverb.php
'apps' => [
    [
        'key' => env('REVERB_APP_KEY'),
        'secret' => env('REVERB_APP_SECRET'),
        'app_id' => env('REVERB_APP_ID'),
        'ping_interval' => 60,
        'activity_timeout' => 120,
        'allowed_origins' => ['*'],
    ],
],
```

### Load Balancer Configuration

WebSocket connections are stateful — you need sticky sessions or a shared pub/sub backend.

**nginx load balancing:**

```nginx
upstream reverb_cluster {
    # IP hash for sticky sessions
    ip_hash;
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080;
}

server {
    location /app {
        proxy_pass http://reverb_cluster;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 7d;
    }
}
```

## Monitoring Connections

### Artisan Commands

```bash
# Check Reverb status
php artisan reverb:connections

# Restart Reverb (drops all connections)
supervisorctl restart reverb
```

### Custom Health Check

```php
// Ping the WebSocket server from a scheduled command
// Requires: composer require textalk/websocket
$schedule->call(function () {
    try {
        $client = new \WebSocket\Client("ws://127.0.0.1:8080/app/" . config('reverb.apps.0.key'));
        $client->setTimeout(5);
        $client->send(json_encode(['event' => 'pusher:ping']));
        $response = $client->receive();
        $client->close();

        Cache::put('reverb:healthy', true, 60);
    } catch (\Throwable $e) {
        Cache::put('reverb:healthy', false, 60);
        Log::error('Reverb health check failed', ['error' => $e->getMessage()]);
    }
})->everyMinute();
```

## Common Patterns

### Broadcast Only to Specific Users

```php
// Using notifications channel (built-in)
class InvoicePaid extends Notification implements ShouldBroadcast
{
    public function via($notifiable): array
    {
        return ['broadcast', 'database'];
    }

    public function toBroadcast($notifiable): BroadcastMessage
    {
        return new BroadcastMessage([
            'invoice_id' => $this->invoice->id,
            'amount' => $this->invoice->amount,
        ]);
    }
}

// Dispatch
$user->notify(new InvoicePaid($invoice));

// Client listens on the automatic notification channel
Echo.private(`App.Models.User.${userId}`)
    .notification((n) => console.log(n.type, n))
```

### Model Broadcasting (Auto-broadcast on model events)

```php
use Illuminate\Database\Eloquent\BroadcastsEvents;

class Message extends Model
{
    use BroadcastsEvents;

    public function broadcastOn(string $event): array
    {
        return match ($event) {
            'created' => [new PrivateChannel("chat.{$this->chat_room_id}")],
            'updated' => [new PrivateChannel("chat.{$this->chat_room_id}")],
            default => [],
        };
    }

    // Customize payload per event
    public function broadcastWith(string $event): array
    {
        return [
            'id' => $this->id,
            'body' => $this->body,
            'user' => [
                'id' => $this->user->id,
                'name' => $this->user->name,
            ],
            'created_at' => $this->created_at->toIso8601String(),
        ];
    }
}
```

```typescript
// Client — listen to model events
Echo.private(`chat.${roomId}`)
    .listen('.MessageCreated', (e) => {
        messages.value.push(e.model)
    })
    .listen('.MessageUpdated', (e) => {
        const idx = messages.value.findIndex(m => m.id === e.model.id)
        if (idx !== -1) messages.value[idx] = e.model
    })
```
