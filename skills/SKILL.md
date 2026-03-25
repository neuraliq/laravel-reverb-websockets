---
name: laravel-reverb-websockets
description: Laravel Reverb WebSocket server and Echo client integration. Use when setting up real-time features, configuring Reverb, creating broadcast events, implementing private/presence channels, building live notifications, or adding WebSocket functionality to Laravel apps.
compatible_agents:
  - Claude Code
  - Cursor
  - Windsurf
  - Copilot
tags:
  - laravel
  - reverb
  - websockets
  - echo
  - broadcasting
  - real-time
  - channels
---

# Laravel Reverb + Echo

Reverb is Laravel's first-party WebSocket server. Echo is the client-side library that connects to it. Together they enable real-time features without external services like Pusher.

## Context

You are working with Laravel applications that need real-time bidirectional communication. This skill covers Reverb server setup, broadcasting events, channel authorization (public/private/presence), and Vue.js integration patterns.

## Rules

1. Always implement `ShouldBroadcast` (not `ShouldBroadcastNow`) for non-critical events — queues the broadcast for reliability
2. Always use `broadcastAs()` with a dot-prefixed name — prevents event class name exposure to clients
3. Always authorize private/presence channels in `routes/channels.php` — never skip authorization
4. Always use `toOthers()` when the broadcasting user shouldn't receive their own event
5. Always clean up channels in `onUnmounted()` with `Echo.leave()` — prevents memory leaks
6. Always use presence channels for "who's online" features — not custom polling
7. Use whisper (client events) only for ephemeral data like typing indicators — never for persistent data
8. Keep broadcast payloads small — use `broadcastWith()` to select only needed fields
9. Use separate channels for different concerns — prevents event collision
10. Always run Reverb as a separate Supervisor process — never share with Octane or Horizon

## Examples

### Installation

```bash
php artisan install:broadcasting
```

### Environment Configuration

```env
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=my-app
REVERB_APP_KEY=my-app-key
REVERB_APP_SECRET=my-app-secret
REVERB_HOST=localhost
REVERB_PORT=8080
```

### Echo Client Setup

```typescript
// resources/js/echo.ts
import Echo from 'laravel-echo'
import Pusher from 'pusher-js'

window.Pusher = Pusher

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT ?? 80,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
})
```

### Broadcast Event

```php
// app/Events/OrderShipped.php
class OrderShipped implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(public Order $order) {}

    public function broadcastOn(): array
    {
        return [new PrivateChannel("orders.{$this->order->user_id}")];
    }

    public function broadcastAs(): string
    {
        return 'order.shipped';
    }

    public function broadcastWith(): array
    {
        return [
            'id' => $this->order->id,
            'status' => $this->order->status,
        ];
    }
}
```

### Channel Authorization

```php
// routes/channels.php
Broadcast::channel('orders.{userId}', function ($user, $userId) {
    return $user->id === (int) $userId;
});
```

### Presence Channel (Online Users)

```php
// routes/channels.php
Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return ['id' => $user->id, 'name' => $user->name];
    }
});
```

```typescript
// Client
Echo.join(`chat.${roomId}`)
    .here((users) => console.log('Online:', users))
    .joining((user) => console.log('Joined:', user))
    .leaving((user) => console.log('Left:', user))
    .listen('.message.sent', (e) => messages.push(e))
```

### Typing Indicator (Whisper)

```typescript
const channel = Echo.private(`chat.${roomId}`)

channel.whisper('typing', { user: currentUser.value })

channel.listenForWhisper('typing', (e) => {
    typingUsers.value.set(e.user.id, e.user)
})
```

## Anti-Patterns

- Broadcasting sensitive data — payload is visible to clients
- Using public channels for user-specific data — anyone can listen
- Skipping channel authorization — security vulnerability
- Not cleaning up listeners — memory leaks in SPA
- Using polling for presence — inefficient, use presence channels

## References

- [Laravel Broadcasting](https://laravel.com/docs/broadcasting)
- [Laravel Reverb](https://reverb.laravel.com/)
- [Laravel Echo](https://laravel-echo.readthedocs.io/)

## When to Use This Skill

Use this skill when:
- Setting up real-time features in Laravel
- Configuring Reverb server
- Creating broadcast events
- Implementing private/presence channels
- Building live notifications or chat systems
- Adding WebSocket functionality to existing Laravel apps
