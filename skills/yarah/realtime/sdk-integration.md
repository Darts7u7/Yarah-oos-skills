# Real-time SDK Integration

Use Yarah SDK for WebSocket pub/sub messaging in your frontend application.

## Setup

First, ensure your `.env` file is configured with your Yarah URL and anon key. Get the anon key with `npx -y @yarahdev/cli secrets get ANON_KEY`. See the main [SKILL.md](../SKILL.md) for framework-specific variable names and full setup steps.

```javascript
import { createClient } from '@yarahdev/sdk'

const yarah = createClient({
  baseUrl: process.env.NEXT_PUBLIC_YARAH_URL,       // adjust prefix for your framework
  anonKey: process.env.NEXT_PUBLIC_YARAH_ANON_KEY   // adjust prefix for your framework
})
```

For Next.js / SSR Client Components, use `@yarahdev/sdk/ssr` so the browser WebSocket can authenticate with the access-token cookie:

```typescript
import { createBrowserClient } from '@yarahdev/sdk/ssr'

const yarah = createBrowserClient()
```

Use `createBrowserClient()` for authenticated browser Realtime connections. It reads `yarah_access_token`, refreshes through `/api/auth/refresh`, and applies same-user refreshes with `AuthChangeEvent.TOKEN_REFRESHED` internally so active Realtime sockets stay connected; the fresh JWT is used on the next handshake.

## Backend Setup

If the task needs channel patterns, database triggers, or channel/message RLS, use the **yarah-cli** skill's [realtime](../../yarah-cli/references/realtime.md) reference. This SDK guide covers frontend connection, subscription, publishing, and presence handling.

## Usage Examples

### Connect

```javascript
await yarah.realtime.connect()
console.log('Connected:', yarah.realtime.isConnected)
```

### Subscribe to Channel

```javascript
const response = await yarah.realtime.subscribe('order:123')

if (!response.ok) {
  console.error('Failed:', response.error?.message)
} else {
  console.log('Subscribed to:', response.channel)
  console.log('Members already present:', response.presence.members)
}

// Auto-connects if not connected
```

### Presence Snapshot on Subscribe

A successful `subscribe()` always returns a `presence` snapshot:

```javascript
const subscribeResponse = {
  ok: true,
  channel: 'order:123',
  presence: {
    members: [
      {
        type: 'user',
        presenceId: 'user-123',
        joinedAt: '2026-04-25T17:00:00.000Z'
      }
    ]
  }
}
```

Use this snapshot to seed local participant state before listening for live deltas.

- `presence.members` is the initial source of truth for who is already in the channel
- `presenceId` is the stable key for a member: user ID for `type: 'user'`, socket ID for `type: 'anonymous'`
- Authenticated users are deduplicated into one logical member across multiple sockets or tabs
- Anonymous connections are tracked per socket, so multiple tabs show up as separate members
- Initialize the current user's presence from the `subscribe()` response; your own presence is already represented there

### Listen for Events

```javascript
// Listen for events
yarah.realtime.on('status_changed', (payload) => {
  console.log('Status:', payload.status)
  console.log('Meta:', payload.meta.messageId, payload.meta.timestamp)
})

// Presence deltas for other members in the channel
yarah.realtime.on('presence:join', (message) => {
  console.log('Member joined:', message.member.presenceId, message.member.type)
})

yarah.realtime.on('presence:leave', (message) => {
  console.log('Member left:', message.member.presenceId)
})

// Listen once
yarah.realtime.once('order_completed', (payload) => {
  console.log('Completed:', payload)
})

// Remove listener
yarah.realtime.off('status_changed', handler)
```

### Integrate Presence into UI State

```javascript
const channel = `chat:${roomId}`
const response = await yarah.realtime.subscribe(channel)

if (!response.ok) throw new Error(response.error?.message || 'Subscribe failed')

let members = response.presence.members
renderMembers(members)

const handleJoin = ({ member, meta }) => {
  if (meta.channel !== channel) return

  const exists = members.some((current) => current.presenceId === member.presenceId)
  members = exists ? members : [...members, member]
  renderMembers(members)
}

const handleLeave = ({ member, meta }) => {
  if (meta.channel !== channel) return

  members = members.filter((current) => current.presenceId !== member.presenceId)
  renderMembers(members)
}

yarah.realtime.on('presence:join', handleJoin)
yarah.realtime.on('presence:leave', handleLeave)
```

### Publish Messages

```javascript
// Must be subscribed to channel first
await yarah.realtime.publish('chat:room-1', 'new_message', {
  text: 'Hello!',
  sender: 'Alice'
})
```

### Unsubscribe and Disconnect

```javascript
yarah.realtime.unsubscribe('order:123')
yarah.realtime.disconnect()
```

### Connection Events

```javascript
yarah.realtime.on('connect', () => console.log('Connected'))
yarah.realtime.on('disconnect', (reason) => console.log('Disconnected:', reason))
yarah.realtime.on('connect_error', (err) => console.error('Error:', err))
yarah.realtime.on('error', ({ code, message }) => console.error(code, message))
```

Error codes: `UNAUTHORIZED`, `NOT_SUBSCRIBED`, `INTERNAL_ERROR`

### Properties

```javascript
yarah.realtime.isConnected           // boolean
yarah.realtime.connectionState       // 'disconnected' | 'connecting' | 'connected'
yarah.realtime.socketId              // string
yarah.realtime.getSubscribedChannels() // string[]
```

### Message Metadata

All messages include `meta`:

```javascript
const message = {
  meta: {
    messageId: 'uuid',
    channel: 'order:123',
    senderType: 'system' | 'user',
    senderId: 'user-uuid',  // if user
    timestamp: 'ISO string'
  },
  // ...payload fields
}
```

### Complete Example

```javascript
await yarah.realtime.connect()

const channel = `order:${orderId}`
const response = await yarah.realtime.subscribe(channel)

if (!response.ok) throw new Error(response.error?.message || 'Subscribe failed')

let members = response.presence.members
renderPresence(members)

yarah.realtime.on('status_changed', (payload) => {
  updateUI(payload.status)
})

yarah.realtime.on('presence:join', ({ member, meta }) => {
  if (meta.channel !== channel) return
  const exists = members.some((current) => current.presenceId === member.presenceId)
  members = exists ? members : [...members, member]
  renderPresence(members)
})

yarah.realtime.on('presence:leave', ({ member, meta }) => {
  if (meta.channel !== channel) return
  members = members.filter((current) => current.presenceId !== member.presenceId)
  renderPresence(members)
})

// Client can also publish
await yarah.realtime.publish(channel, 'viewed', {
  viewedAt: new Date().toISOString()
})
```

---

## Best Practices

1. **Ensure channel pattern exists before subscribing**
   - The frontend can only subscribe to channel names that match an enabled backend channel pattern.
   - If no channel pattern exists, finish backend setup with the **yarah-cli** skill first.

2. **Seed presence from `subscribe()` before processing deltas**
   - Treat `response.presence.members` as the initial source of truth
   - Apply `presence:join` and `presence:leave` as incremental updates after the subscribe call succeeds
   - Use `presenceId` as your stable UI key

3. **Handle connection events and rebuild presence after reconnect**
   - Listen for `connect`, `disconnect`, and `connect_error` events
   - Presence is ephemeral and tracked in-memory on a single backend instance, so reconnect by subscribing again and rebuilding from the returned snapshot

4. **Gate user-dependent side effects on auth hydration**
   - Webhook-backed events can arrive before a cold-load `getCurrentUser()` refresh finishes
   - If an event branches on the current user, wait for `authLoading === false` before running it or flipping a "first event wins" guard
   - See [../auth/sdk-integration.md#gate-user-dependent-side-effects-during-auth-loading](../auth/sdk-integration.md#gate-user-dependent-side-effects-during-auth-loading)

5. **Design for presence visibility rules**
   - Authenticated subscribers expose their user ID through `presenceId` to other channel members
   - Use non-presence channels when subscriber identity should stay opaque

6. **Clean up subscriptions**
   - Unsubscribe from channels when no longer needed
   - Disconnect when leaving the page/component

### Recommended Workflow

```text
1. Confirm backend setup          → channel pattern exists and is enabled
2. Connect to realtime            → await yarah.realtime.connect()
3. Subscribe and seed presence    → const response = await yarah.realtime.subscribe('channel')
4. Listen for events and deltas   → on('event', handler) + on('presence:join'/'presence:leave')
5. Clean up on unmount            → unsubscribe() and disconnect()
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Subscribing before backend channel setup exists | Configure channel patterns, database triggers, and RLS before subscribing |
| Waiting for a self `presence:join` event | Initialize local presence state from `subscribe()` response |
| Treating presence as durable/global state | Treat presence as single-instance, in-memory state and resubscribe after reconnects |
| Missing connection error handling | Listen for `connect_error` and `disconnect` events |
| Leaving subscriptions active after unmount | Unsubscribe on component unmount |
| Publishing before subscribing | Subscribe to the channel before publishing |
