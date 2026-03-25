# Laravel Reverb + Echo Skill

An AI agent skill that teaches coding assistants how to build real-time features in Laravel using Reverb (WebSocket server) and Echo (client-side library).

## What This Skill Does

This skill provides procedural knowledge for AI agents implementing real-time bidirectional communication in Laravel applications. It covers the full stack — from server-side broadcast events to client-side channel subscriptions — using Laravel's first-party Reverb WebSocket server.

### Topics Covered

- **Installation & Configuration** — Reverb setup, environment variables, Echo client initialization
- **Broadcast Events** — `ShouldBroadcast`, `broadcastAs()`, `broadcastWith()`, `broadcastOn()`
- **Channel Types** — Public, private, and presence channels with proper authorization
- **Channel Authorization** — `routes/channels.php` for private and presence channel access control
- **Presence Channels** — Online user tracking with `here()`, `joining()`, `leaving()`
- **Client Events** — Whisper for ephemeral data like typing indicators
- **Model Broadcasting** — `BroadcastsEvents` trait for automatic model event broadcasting
- **Notifications** — Broadcasting notifications via `ShouldBroadcast` on notification classes
- **Production Deployment** — Supervisor config, Nginx reverse proxy with WebSocket upgrade headers, SSL/TLS
- **Scaling** — Horizontal scaling with Redis pub/sub, sticky sessions, OS-level tuning
- **Health Checks** — WebSocket ping monitoring via scheduled commands
- **Vue.js Integration** — Echo channel subscriptions in Vue components with cleanup

## Installation

### Via skills CLI (skills.sh)

```bash
npx skills add neuraliq/laravel-reverb-websockets
```

### Via Laravel Boost (skills.laravel.cloud)

```bash
php artisan boost:add-skill neuraliq/laravel-reverb-websockets
```

### Manual

Copy the `skills/SKILL.md` file into your project's `.cursor/skills/`, `.claude/skills/`, or equivalent agent skills directory.

## Compatibility

| Agent | Supported |
|-------|-----------|
| Claude Code | Yes |
| Cursor | Yes |
| Windsurf | Yes |
| GitHub Copilot | Yes |

## Requirements

- Laravel 11+
- Laravel Reverb
- Laravel Echo (`@laravel-echo`)
- Pusher.js (`pusher-js` — Reverb uses the Pusher protocol)
- Redis (for horizontal scaling)
- Supervisor (for production)

## File Structure

```
├── skills/
│   └── SKILL.md          # Main skill definition
├── references/
│   ├── production.md     # Supervisor, Nginx, SSL, Sanctum auth, deployment checklist
│   └── scaling.md        # Connection limits, Redis pub/sub, load balancing, model broadcasting
└── README.md
```

## Key Rules

1. Always implement `ShouldBroadcast` (not `ShouldBroadcastNow`) for non-critical events
2. Always define `broadcastAs()` to use custom event names — clients listen with a `.` prefix
3. Always authorize private/presence channels in `routes/channels.php`
4. Always clean up channels with `Echo.leave()` in `onUnmounted()`
5. Keep broadcast payloads small with `broadcastWith()`
6. Always run Reverb as a separate Supervisor process

## License

MIT
