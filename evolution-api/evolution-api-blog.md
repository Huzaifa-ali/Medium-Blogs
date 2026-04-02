# Evolution API: How to Run WhatsApp Automation Without the Business API

**Keywords:** Evolution API, WhatsApp automation, Baileys library, WhatsApp Web API, self-hosted WhatsApp, WhatsApp bot, Evolution API tutorial, WhatsApp without Business API, WhatsApp QR code authentication, open source WhatsApp API, WhatsApp Node.js, WhatsApp multi-device, Evolution API setup, WhatsApp REST API, WhatsApp messaging automation, Baileys WhatsApp library, WhatsApp linked device API, Evolution API Docker, WhatsApp chatbot backend, send WhatsApp messages programmatically

---

I was building a notification system and needed to send WhatsApp messages from my backend. Like most developers, I started by Googling "WhatsApp API" — and immediately ran into the Meta Business API wall.

Apply for access. Verify your business. Submit every message template for review. Wait days for approval. Pay per conversation.

For an MVP? That's a non-starter.

Then I stumbled onto something called Evolution API, and it completely changed how I think about WhatsApp integration.

---

## So What Exactly Is Evolution API?

At its core, Evolution API is an open-source REST API that sits on top of the Baileys library — a Node.js implementation of the WhatsApp Web protocol. Think of it this way: when you open WhatsApp Desktop and scan a QR code, your computer becomes a "linked device." Evolution API does the same thing, but instead of a desktop app, it's a Docker container that exposes REST endpoints.

You scan a QR code once, and suddenly your backend can send and receive WhatsApp messages through simple HTTP calls. No Meta approval. No template reviews. No per-message fees.

[IMAGE: Insert the **System Architecture** diagram here — shows Your Backend → Evolution API (REST API + Baileys) → WhatsApp Servers → Customer, with PostgreSQL and Docker Volume for persistence. Export from `diagrams/evolution-01-architecture.drawio`]

Here's what makes it interesting:

- It uses Baileys under the hood — a lightweight Node.js library, not a headless browser
- Completely self-hosted. You own the container, you own the data
- Simple REST calls to send text messages, native WhatsApp polls, manage sessions
- Sessions survive container restarts through a combination of Docker volumes and PostgreSQL
- Webhooks fire back to your backend whenever a customer replies or the connection state changes

---

## Getting It Running

I'll show you two approaches — a Docker Compose setup for when Evolution is part of a bigger stack, and a standalone `docker run` command that I actually use in production.

### Docker Compose (When You Have Multiple Services)

This is the cleaner approach if you're running Evolution alongside your backend, Redis, etc.:

```yaml
services:
  evolution:
    image: evoapicloud/evolution-api:latest
    ports:
      - "8080:8080"
    environment:
      AUTHENTICATION_API_KEY: your-secret-api-key
      WEBHOOK_GLOBAL_URL: http://api:8000/api/v1/evolution/webhook
      WEBHOOK_GLOBAL_ENABLED: "true"
      WEBHOOK_EVENTS_MESSAGES_UPSERT: "true"
      WEBHOOK_EVENTS_CONNECTION_UPDATE: "true"
      WEBHOOK_EVENTS_QRCODE_UPDATED: "true"
      DATABASE_ENABLED: "true"
      DATABASE_PROVIDER: postgresql
      DATABASE_CONNECTION_URI: "postgresql://user:pass@db:5432/evolution?schema=evolution"
      DATABASE_CONNECTION_CLIENT_NAME: evolution_v2
      DATABASE_SAVE_DATA_INSTANCE: "true"
      DATABASE_SAVE_DATA_NEW_MESSAGE: "false"
      DATABASE_SAVE_MESSAGE_UPDATE: "false"
      DATABASE_SAVE_DATA_CONTACTS: "false"
      DATABASE_SAVE_DATA_CHATS: "false"
      DATABASE_SAVE_DATA_LABELS: "false"
      DATABASE_SAVE_DATA_HISTORIC: "false"
      LOG_LEVEL: ERROR
    volumes:
      - evolution_instances:/evolution/instances
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: evolution
    volumes:
      - pg_data:/var/lib/postgresql/data

volumes:
  evolution_instances:
  pg_data:
```

One thing worth calling out: see all those `DATABASE_SAVE_DATA_*` flags set to `false`? Evolution can store every message, contact, chat, and label in PostgreSQL. You almost certainly don't want that. Your own backend should be the source of truth for messages — let Evolution just be the transport layer. I only enable `INSTANCE` (the config and auth state) and turn everything else off.


### Standalone Docker Run (What I Actually Use)

When I'm running Evolution as a standalone container pointing at an external PostgreSQL database (Supabase in my case), this is the command:

```bash
docker run -d --name evolution \
  -p 8080:8080 \
  --add-host=host.docker.internal:host-gateway \
  -e SERVER_URL=http://localhost:8080 \
  -e AUTHENTICATION_API_KEY=your-secret-api-key \
  -e WEBHOOK_GLOBAL_ENABLED=true \
  -e WEBHOOK_GLOBAL_URL=http://host.docker.internal:8000/api/v1/evolution/webhook \
  -e WEBHOOK_EVENTS_MESSAGES_UPSERT=true \
  -e WEBHOOK_EVENTS_CONNECTION_UPDATE=true \
  -e WEBHOOK_EVENTS_QRCODE_UPDATED=true \
  -e DATABASE_ENABLED=true \
  -e DATABASE_PROVIDER=postgresql \
  -e DATABASE_CONNECTION_URI="postgresql://user:pass@your-db-host:5432/dbname?schema=evolution" \
  -e DATABASE_CONNECTION_CLIENT_NAME=evolution_v2 \
  -e DATABASE_SAVE_DATA_INSTANCE=true \
  -e DATABASE_SAVE_DATA_NEW_MESSAGE=false \
  -e DATABASE_SAVE_MESSAGE_UPDATE=false \
  -e DATABASE_SAVE_DATA_CONTACTS=false \
  -e DATABASE_SAVE_DATA_CHATS=false \
  -e DATABASE_SAVE_DATA_LABELS=false \
  -e DATABASE_SAVE_DATA_HISTORIC=false \
  -e CACHE_LOCAL_ENABLED=true \
  -e CACHE_REDIS_ENABLED=false \
  -e RABBITMQ_ENABLED=false \
  -e CHATWOOT_ENABLED=false \
  -e OPENAI_ENABLED=false \
  -e DIFY_ENABLED=false \
  -e S3_ENABLED=false \
  -e WEBSOCKET_ENABLED=false \
  -e LOG_LEVEL=ERROR,WARN \
  -e DEL_INSTANCE=false \
  -v evolution_instances:/evolution/instances \
  evoapicloud/evolution-api:latest
```

That's a lot of flags, so let me explain the ones that actually matter:

The `--add-host` flag is easy to miss but critical — it lets the container reach your backend running on the host machine. Without it, `host.docker.internal` doesn't resolve and your webhooks go nowhere.

The `DATABASE_*` flags connect Evolution to PostgreSQL so your instance config survives even if the container gets nuked. The `CACHE_LOCAL_ENABLED=true` with `CACHE_REDIS_ENABLED=false` keeps things simple — Redis is overkill for a single instance.

All those `*_ENABLED=false` flags at the bottom? Evolution ships with built-in integrations for Chatwoot, OpenAI, Dify, S3, RabbitMQ, and WebSockets. If you're not using them, turn them off. Less memory, faster startup, smaller attack surface.

And `DEL_INSTANCE=false` is important — without it, Evolution might auto-delete your instance when it disconnects, which means you'd have to recreate it from scratch instead of just reconnecting.

---

## Why PostgreSQL Matters (Not Just Docker Volumes)

This took me a while to figure out. Evolution API has two layers of persistence, and you need to understand both or you'll lose your session at the worst possible time.

[IMAGE: Insert the **PostgreSQL Persistence Model** diagram here — shows Docker Volume (session keys, device identity) vs PostgreSQL (instance config, auth metadata), with three recovery scenarios. Export from `diagrams/evolution-02-postgres-persistence.drawio`]

The Docker volume (`evolution_instances`) stores the raw WhatsApp session files — encryption keys, device identity, auth credentials. This is what keeps you connected without re-scanning the QR code.

PostgreSQL stores the instance configuration — the instance name, settings, webhook URLs, and auth state metadata. This is what lets Evolution know the instance *exists* even if the container restarts.

Here's why this matters in practice:

<table>
  <thead>
    <tr>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Scenario</th>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">What Happens</th>
    </tr>
  </thead>
  <tbody>
    <tr style="background:#d4edda;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Both intact</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Auto-reconnects on restart. You don't even notice.</td>
    </tr>
    <tr style="background:#fff3cd;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Volume wiped, DB intact</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Evolution knows the instance exists but lost the session keys. You need to re-scan QR, but you don't have to recreate the instance from scratch.</td>
    </tr>
    <tr style="background:#f8d7da;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Both wiped</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Everything's gone. Full recreate + re-scan QR.</td>
    </tr>
  </tbody>
</table>

The connection string uses a `?schema=evolution` parameter, which tells Evolution to create its tables in a separate PostgreSQL schema. I share the same Supabase database between my backend and Evolution — they each get their own schema and never step on each other's tables.

```
postgresql://user:password@host:5432/dbname?schema=evolution
```

---

## Creating an Instance and Connecting

An "instance" is just a WhatsApp session — one phone number, one connection. Creating one is a single API call:

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: your-secret-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "my-store",
    "integration": "WHATSAPP-BAILEYS",
    "qrcode": true,
    "groupsIgnore": true,
    "alwaysOnline": false,
    "readMessages": false,
    "syncFullHistory": false
  }'
```

A few things worth noting: `groupsIgnore` skips group messages (you almost certainly want this for a bot). `alwaysOnline` shows your number as permanently online — I keep it off because it's a red flag for WhatsApp's detection systems. And `syncFullHistory` pulls every old message when you connect, which is slow and unnecessary.

Once the instance exists, you need to link it to a WhatsApp account. There are two ways to do this:

[IMAGE: Insert the **QR Code vs Pairing Code** diagram here — shows both authentication paths leading to the Connected state. Export from `diagrams/evolution-03-qr-vs-pairing.drawio`]

**QR Code** — call `GET /instance/connect/my-store` and you get back a base64-encoded PNG. Display it somewhere, scan it with WhatsApp → Linked Devices, done.

**Pairing Code** — call `POST /instance/connect/my-store` with `{"number": "923001234567"}` and you get an 8-digit code. Enter it in WhatsApp under "Link with phone number." I prefer this for server setups where there's no screen to show a QR.

---

## Sending Messages

This is the fun part. Once connected, sending a WhatsApp message is literally one HTTP call:

```bash
curl -X POST http://localhost:8080/message/sendText/my-store \
  -H "apikey: your-secret-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "923001234567@s.whatsapp.net",
    "text": "Hi! Your order #1001 has been confirmed.",
    "delay": 3000
  }'
```

You can also send native WhatsApp polls:

```bash
curl -X POST http://localhost:8080/message/sendPoll/my-store \
  -H "apikey: your-secret-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "923001234567@s.whatsapp.net",
    "name": "How was your experience?",
    "values": ["Great", "Good", "Could be better"],
    "selectableCount": 1,
    "delay": 2000
  }'
```

One gotcha: Evolution expects phone numbers without the `+` prefix and with `@s.whatsapp.net` appended. If you're storing numbers in E.164 format (which you should be), the conversion is trivial:

```python
def e164_to_evolution(phone: str) -> str:
    return phone.lstrip("+") + "@s.whatsapp.net"

# +923001234567 → 923001234567@s.whatsapp.net
```

---

## The Typing Simulation Trick

This is the part that most tutorials completely skip, and it's arguably the most important thing in this entire post.

WhatsApp actively monitors for bot-like behavior. If your number sends messages with zero delay — bang, bang, bang, one after another with no pause — you will get flagged. And flagged means banned.

Evolution API has a built-in `delay` parameter that simulates typing before the message is sent. The recipient actually sees the "typing..." indicator, just like they would if a real person was composing the message.

[IMAGE: Insert the **Typing Simulation** diagram here — shows the sequence: Backend calculates delay → sends to Evolution with delay parameter → Evolution shows "typing..." → message delivered. Export from `diagrams/evolution-04-typing-simulation.drawio`]

The formula I use:

```python
def calculate_delay_ms(message: str) -> int:
    delay_ms = len(message) * 50   # 50ms per character
    max_delay_ms = 25 * 1000       # cap at 25 seconds
    return min(delay_ms, max_delay_ms)
```

A short 100-character message gets a 5-second typing delay. A longer 500-character message hits the 25-second cap. It looks completely natural from the recipient's side.

This one parameter is the difference between your number lasting months and getting banned in days. Don't skip it.

---

## Handling Incoming Messages

Evolution doesn't just send messages — it also receives them. When a customer replies, Evolution fires a webhook to your backend. There are three events you'll actually care about:

[IMAGE: Insert the **Webhook Events** diagram here — shows Customer message → MESSAGES_UPSERT, Connection change → CONNECTION_UPDATE, QR refresh → QRCODE_UPDATED, all flowing to your backend. Export from `diagrams/evolution-05-webhook-events.drawio`]

<table>
  <thead>
    <tr>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Event</th>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">What Triggers It</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>MESSAGES_UPSERT</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">A customer sends you a WhatsApp message</td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>CONNECTION_UPDATE</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Your WhatsApp connection state changes (connected, disconnected, reconnecting)</td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>QRCODE_UPDATED</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">The QR code refreshes — they expire roughly every 60 seconds</td>
    </tr>
  </tbody>
</table>

Here's what an incoming message webhook looks like:

```json
{
  "event": "MESSAGES_UPSERT",
  "instance": "my-store",
  "data": {
    "key": {
      "remoteJid": "923001234567@s.whatsapp.net",
      "fromMe": false,
      "id": "3EB0A0B1C2D3E4F5"
    },
    "message": {
      "conversation": "What's the status of my order?"
    },
    "pushName": "Ahmed"
  }
}
```

And a connection status change:

```json
{
  "event": "CONNECTION_UPDATE",
  "instance": "my-store",
  "data": {
    "state": "open",
    "statusReason": 200
  }
}
```

The `state` field is either `open` (connected and ready), `close` (disconnected), or `connecting` (trying to reconnect).

One thing I'd strongly recommend: verify webhook signatures. Evolution supports HMAC-SHA256 verification — set `WEBHOOK_GLOBAL_HMAC_SECRET` in your environment and check the `X-Webhook-Signature` header on every incoming request. Without this, anyone who knows your webhook URL can send fake events.

```python
import hmac, hashlib

def verify_signature(body: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)
```

---

## Session Lifecycle: What Happens When Things Go Wrong

WhatsApp sessions aren't permanent. Phones disconnect, containers crash, WhatsApp occasionally drops linked devices for no apparent reason. If you're running this in production, you need to understand the state machine.

[IMAGE: Insert the **Session State Machine** diagram here — shows the states: STOPPED → STARTING → SCAN_QR_CODE → WORKING, with transitions to FAILED on ban/timeout, and reconnect paths. Export from `diagrams/evolution-06-session-state-machine.drawio`]

The happy path is straightforward: you start a session, it moves to STARTING, Evolution generates a QR code (SCAN_QR_CODE), you scan it, and it moves to WORKING. From there, messages flow in both directions.

The unhappy paths are where it gets interesting. If WhatsApp rejects the connection (status codes 401, 408, 428, or 440 — usually meaning the number is banned or rate-limited), the session moves to FAILED. A graceful disconnect moves it to STOPPED, from where you can reconnect.

### Startup Recovery

Here's something I built that saved me a lot of headaches: a reconciliation check that runs every time my backend starts up.

[IMAGE: Insert the **Startup Recovery Flowchart** diagram here — shows the backend querying active sessions, checking each against Evolution API, with four outcomes: already connected, trigger reconnect, recreate instance, or mark disconnected. Export from `diagrams/evolution-07-startup-recovery.drawio`]

The idea is simple. On startup, I query every session that should be active (status WORKING, STARTING, or SCAN_QR_CODE) and check it against the live Evolution API:

```python
def reconcile_sessions():
    for session in get_active_sessions():
        status = evolution_api.get_status(session.instance_name)
        
        if status == "NOT_FOUND":
            # Evolution lost this instance — recreate it
            evolution_api.create_instance(session.instance_name)
            session.status = "SCAN_QR_CODE"
            
        elif status == "WORKING":
            pass  # Still connected, nothing to do
            
        elif status in ("STOPPED", "STARTING"):
            evolution_api.connect(session.instance_name)
            session.status = "STARTING"
```

If Evolution's container crashed and lost its data, my backend automatically recreates the instances with the same names. The only manual step is re-scanning the QR code. Without this, you'd wake up to a dead system and have to manually recreate everything.

---

## Duplicate Phone Protection

If you're building a multi-tenant system where each tenant gets their own WhatsApp number, there's a subtle problem: you can't know which phone number is being connected until *after* the QR scan succeeds. The phone comes from the `CONNECTION_UPDATE` webhook in the `ownerJid` field.

So the duplicate check has to happen at connection time. When a `CONNECTION_UPDATE` arrives with `state=open`, I extract the phone number, check if any other active session is already using it, and if there's a conflict, I immediately delete the new instance. It's a race condition you have to handle — two people scanning QR codes for the same phone number at roughly the same time.

---

## When You Shouldn't Use This

I want to be honest about the tradeoffs, because Evolution API isn't the right choice for everyone:

<table>
  <thead>
    <tr>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Factor</th>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Evolution API (Baileys)</th>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Official Business API</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Setup time</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">5 minutes</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Days to weeks</td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Cost</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Free (self-hosted)</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Per-conversation pricing</td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Template approval</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">None needed</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Required for outbound</td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Reliability</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Depends on your infra</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Meta's SLA</td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Ban risk</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Real (bot detection)</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Zero</td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Scale</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Hundreds per day safely</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Millions per day</td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><strong>Support</strong></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">GitHub community</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Meta support</td>
    </tr>
  </tbody>
</table>

If you're building an MVP, a side project, or serving a market where WhatsApp is the dominant communication channel (South Asia, Latin America, parts of Africa), Evolution API is a great fit. You get full control, zero cost, and you can move fast.

If you're at enterprise scale, in a regulated industry, or can't afford any downtime — use the official Business API. The approval process is annoying but the reliability is worth it.

---

## Quick Reference

### Endpoints You'll Use Most

<table>
  <thead>
    <tr>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Action</th>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Method</th>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Endpoint</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Create instance</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>POST</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>/instance/create</code></td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Get QR code</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>GET</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>/instance/connect/{instance}</code></td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Get pairing code</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>POST</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>/instance/connect/{instance}</code></td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Check status</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>GET</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>/instance/connectionState/{instance}</code></td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Send text</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>POST</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>/message/sendText/{instance}</code></td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Send poll</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>POST</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>/message/sendPoll/{instance}</code></td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Disconnect</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>DELETE</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>/instance/logout/{instance}</code></td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Delete instance</td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>DELETE</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>/instance/delete/{instance}</code></td>
    </tr>
  </tbody>
</table>

### Environment Variables Worth Knowing

<table>
  <thead>
    <tr>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">Variable</th>
      <th style="text-align:left;padding:12px 16px;background:#f8f9fa;border-bottom:2px solid #dee2e6;">What It Does</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>AUTHENTICATION_API_KEY</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">API key for authenticating all requests</td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>SERVER_URL</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Public URL of your Evolution instance</td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>WEBHOOK_GLOBAL_URL</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Where Evolution sends webhook events</td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>DATABASE_ENABLED</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Enable external database persistence (<code>true</code> or <code>false</code>)</td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>DATABASE_PROVIDER</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Database type — <code>postgresql</code> or <code>mongodb</code></td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>DATABASE_CONNECTION_URI</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Full connection string (use <code>?schema=evolution</code>)</td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>DATABASE_SAVE_DATA_INSTANCE</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">The one flag you should set to <code>true</code></td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>CACHE_LOCAL_ENABLED</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">In-memory caching — keep it simple</td>
    </tr>
    <tr>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>DEL_INSTANCE</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;">Set to <code>false</code> so instances survive disconnects</td>
    </tr>
    <tr style="background:#f8f9fa;">
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>LOG_LEVEL</code></td>
      <td style="padding:10px 16px;border-bottom:1px solid #e9ecef;"><code>ERROR,WARN</code> for production, <code>INFO</code> for debugging</td>
    </tr>
  </tbody>
</table>

---

## Final Thoughts

Evolution API turned what would have been a weeks-long approval process into a 5-minute Docker setup. It's not without its risks — you're responsible for anti-detection, session recovery, and keeping the infrastructure running — but for the right use case, it removes a massive barrier.

The Baileys library underneath is actively maintained, and the Evolution API community is solid. If you need WhatsApp integration and you're not at enterprise scale yet, this is worth a serious look.

---

*Follow me for more deep-dives into the tools and patterns I discover while building real products.*