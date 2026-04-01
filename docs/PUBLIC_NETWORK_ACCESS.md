# Accessing OpenClaw Through the Public Network

This guide explains the exact approaches available in this project for accessing OpenClaw Gateway when your local machine does **not** have a public IP address or when you need to receive messages from DingTalk's cloud services.

## TL;DR — No Public IP Required

**Good news:** neither integration scheme in this project requires you to expose your OpenClaw Gateway to the public internet. Both schemes work seamlessly behind NAT, corporate firewalls, and home routers with no port forwarding.

| Scheme | Setup difficulty | How it avoids the public IP problem |
|--------|-----------------|-------------------------------------|
| [Scheme 1: DingTalk Robot](#scheme-1-dingtalk-robot-easiest) | ⭐ Easy (~15 min) | Outbound WebSocket (Stream) — your machine calls DingTalk |
| [Scheme 2: DingTalk DEAP Agent](#scheme-2-dingtalk-deap-agent-local-device-control) | ⭐⭐ Medium (~30 min) | Reverse tunnel — Connector binary creates an outbound tunnel to DEAP cloud |
| [Scheme 3: Custom Public URL](#scheme-3-custom-public-gateway-url-advanced) | ⭐⭐⭐ Advanced | You expose the Gateway yourself (nginx, Cloudflare Tunnel, ngrok, etc.) |

---

## Scheme 1: DingTalk Robot (Easiest)

### How it works

```
DingTalk App  ──►  DingTalk Stream WebSocket (cloud)
                          ▲
                          │  outbound WebSocket
                          │  (your machine initiates)
                   DingTalk OpenClaw Connector
                          │
                          ▼
                   OpenClaw Gateway (127.0.0.1:18789)
```

The connector opens an **outbound** WebSocket connection to DingTalk's cloud (`Stream` protocol). DingTalk sends incoming messages down that connection. Your Gateway never listens on the public internet — all traffic flows through the established outbound tunnel.

### Why no public IP is needed

- Your machine is the WebSocket **client**, not the server.
- DingTalk cloud is the server; it simply pushes messages to your open socket.
- NAT, firewalls, and home routers all allow outbound TCP connections without any configuration.

### Setup (≈15 minutes)

1. **Install the plugin**

   ```bash
   openclaw plugins install @dingtalk-real-ai/dingtalk-connector
   ```

2. **Create a DingTalk Robot** at [DingTalk Open Platform](https://open.dingtalk.com/) and note the `clientId` and `clientSecret`.

3. **Configure** `~/.openclaw/openclaw.json`:

   ```json5
   {
     "channels": {
       "dingtalk-connector": {
         "enabled": true,
         "clientId": "dingXXXXXXXX",
         "clientSecret": "your_secret_here"
       }
     }
   }
   ```

4. **Enable the Chat Completions endpoint**:

   ```bash
   openclaw config set gateway.http.endpoints.chatCompletions.enabled true
   ```

5. **Restart the Gateway**:

   ```bash
   openclaw gateway restart
   ```

That's it. Your local OpenClaw Gateway is now reachable from DingTalk without any public IP or port forwarding.

---

## Scheme 2: DingTalk DEAP Agent (Local Device Control)

### How it works

```
DingTalk App  ──►  DEAP Agent (cloud)
                       │
                       │  HTTP over reverse tunnel
                       │
                   DingTalk OpenClaw Connector  ◄── outbound connection (your machine initiates)
                          │
                          ▼
                   OpenClaw Gateway (127.0.0.1:18789)
                          │
                          ▼
                   Local PC / device operations
```

A small Connector **binary** (Go executable, ~10 MB) runs on your machine and dials out to DingTalk's DEAP infrastructure, establishing a persistent reverse tunnel. DEAP routes your natural-language commands through that tunnel to your local Gateway. As with Scheme 1, you initiate every connection; nothing listens on the public internet.

### Why no public IP is needed

- The Connector binary makes an **outbound** TCP connection to DingTalk DEAP.
- DEAP multiplexes HTTP requests back through that single connection.
- Your Gateway stays on `127.0.0.1` and is never reachable from outside.

### Setup (≈30 minutes)

1. **Start OpenClaw Gateway** and configure it:

   ```bash
   openclaw gateway start
   ```

   - Visit `http://127.0.0.1:18789/config` → **Auth** tab → set a Gateway Token.
   - Visit **Http** tab → enable _OpenAI Chat Completions Endpoint_.
   - Click **Save**.

2. **Get your DingTalk `corpId`** from the [DingTalk Developer Platform](https://open-dev.dingtalk.com/).

3. **Get a DEAP `apiKey`** from [DingTalk DEAP Platform](https://deap.dingtalk.com/) → **Security & Permissions** → **API-Key Management**.

4. **Download and run the Connector binary** from the [Releases page](https://github.com/hoskii/dingtalk-openclaw-connector/releases/tag/v0.0.1):

   ```bash
   # macOS example
   unzip connector-mac.zip
   ./connector-darwin -deapCorpId YOUR_CORP_ID -deapApiKey YOUR_API_KEY
   ```

5. **Create and configure a DEAP Agent** at [deap.dingtalk.com](https://deap.dingtalk.com/):
   - Create a new Agent.
   - Add the **OpenClaw** skill and set:
     - `apikey` — your DEAP API Key.
     - `apihost` — `127.0.0.1:18789` (or `localhost:18789` on Windows).
     - `gatewayToken` — the token you set in step 1.
   - Publish the Agent.

6. **Chat with your Agent** in DingTalk to control your local PC via natural language.

---

## Scheme 3: Custom Public Gateway URL (Advanced)

If you explicitly want the Gateway to be reachable over the public internet (e.g., to integrate with other HTTP clients or webhooks), you can expose it yourself and point the connector at it with `gatewayBaseUrl`.

### Typical approaches

| Tool | Description |
|------|-------------|
| **Nginx / Caddy** | Reverse proxy on a VPS with TLS termination |
| **Cloudflare Tunnel** | Free, no public IP needed on origin server |
| **ngrok** | Quick local tunnel, free tier available |
| **frp** | Self-hosted reverse proxy for stable production use |

### Configuration

```json5
{
  "channels": {
    "dingtalk-connector": {
      "enabled": true,
      "clientId": "dingXXXXXXXX",
      "clientSecret": "your_secret_here",
      "gatewayBaseUrl": "https://your-gateway.example.com"  // ← custom public URL
    }
  }
}
```

The connector will call `https://your-gateway.example.com/v1/chat/completions` instead of the local `http://127.0.0.1:18789` default.

> **Security note:** When exposing the Gateway publicly, always set a `gatewayToken` or `gatewayPassword` in your Gateway config to prevent unauthorized access.

---

## Summary

| Question | Answer |
|----------|--------|
| Do I need a public IP? | **No.** Schemes 1 and 2 require only outbound internet access. |
| Is it easy to set up? | **Yes.** Scheme 1 takes ~15 minutes; Scheme 2 takes ~30 minutes. |
| Can I use HTTPS? | **Yes.** Use `gatewayBaseUrl` to point at any public or TLS-terminated URL. |
| Does it work behind a corporate firewall? | **Yes**, as long as outbound HTTPS (port 443) and WSS (WebSocket Secure) are permitted. |
