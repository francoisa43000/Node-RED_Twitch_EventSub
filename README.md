# Node-RED Twitch EventSub

Node-RED flow to subscribe, receive and respond to Twitch EventSub webhook events.

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Dashboard.png">
<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/EventSubFlow.PNG">

## Note
- My [Twitch channel](https://www.twitch.tv/ioodyme)
- To [support me](https://paypal.me/ioodyme)
- It is **HIGHLY** recommended to [secure](https://nodered.org/docs/user-guide/runtime/securing-node-red#editor--admin-api-security) your Node-RED instance (your server will be exposed to the internet)!
- [EventSub documentation](https://dev.twitch.tv/docs/eventsub)
- This flow is compatible with the [Twitch Helix API](https://dev.twitch.tv/docs/api/reference)

## What's New (Modernization)

- **Dashboard v2 (FlowFuse Dashboard)**: UI migrated from node-red-dashboard (v1) to `@flowfuse/node-red-dashboard` (v2). All `ui_*` nodes replaced with `ui-*` equivalents.
- **Native crypto for webhook signature verification**: The `node-red-contrib-crypto-js-dynamic` `hmac` node dependency has been removed. Signature verification now uses the Node.js built-in `crypto` module with `timingSafeEqual` for timing-attack resistance.
- **Replay attack prevention**: Webhook messages with a timestamp older than 10 minutes are automatically rejected with HTTP 403.
- **TTL-based message deduplication**: Duplicate EventSub message IDs are filtered with a 10-minute TTL cache (replaces the previous hourly reset strategy).
- **Bug fixes**: Fixed `flow.get('AppID')` → `flow.get('ClientID')` in token validation functions.

## Prerequisites

- [Node-RED](https://nodered.org/) **v3.0+** running with [HTTPS](https://nodered.org/docs/user-guide/runtime/securing-node-red#enabling-https-access) enabled **OR** using an [ngrok](https://ngrok.com/) tunnel
  - Twitch EventSub requires a publicly accessible HTTPS endpoint
- A [Twitch Developer Application](https://dev.twitch.tv/console/apps/) (Client ID + Client Secret)
- Node-RED version compatibility: tested on Node-RED 3.x / Node.js 18+

## Dependencies

Install the following via the Node-RED Palette Manager:

| Package | Purpose |
|---------|---------|
| `@flowfuse/node-red-dashboard` | Dashboard v2 UI (replaces node-red-dashboard v1) |
| *(optional)* `node-red-contrib-ngrok` | Expose local Node-RED via ngrok tunnel |

> **Note:** `node-red-contrib-crypto-js-dynamic` is no longer required. Webhook signature verification now uses Node.js built-in `crypto`.

### Installing Dashboard v2

In Node-RED, go to **Menu → Manage Palette → Install** and search for:

```
@flowfuse/node-red-dashboard
```

Or via npm in your Node-RED user directory:

```bash
npm install @flowfuse/node-red-dashboard
```

The dashboard UI will be available at `/dashboard` (e.g. `https://your.domain.name/dashboard`).

## For ngrok Users

1. Create a free account on [ngrok](https://ngrok.com/) and save your AuthToken
2. Install the [ngrok node](https://flows.nodered.org/node/node-red-contrib-ngrok) on Node-RED
3. Drag and drop a ngrok node on your flow
4. Set up the node using your AuthToken, open a tunnel, and save the tunnel URL

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/NgrockURL.png" width="50%">

## Twitch Developer App Setup

1. Create an [Application](https://dev.twitch.tv/console/apps/) and save your Client ID and Client Secret
2. Add the following OAuth redirect URLs using your domain name or ngrok tunnel URL:

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Redirect.png" width="80%">

> **Note:** If your Node-RED is secured with HTTP basic auth, use the format:
> `https://admin:password@your.domain.name` or `https://admin:password@xxxxxxxx.ngrok.io`

## Node-RED Setup

### 1. Import the Flow

- [Import](https://nodered.org/docs/user-guide/editor/workspace/import-export) `EventSub-Twitch-Flow.json` into Node-RED

### 2. Configure Settings (Dashboard)

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Settings1.png" width="75%">

Navigate to the dashboard at `https://your.domain.name/dashboard` and fill in the **Settings** form:

| Field | Description |
|-------|-------------|
| **Client ID** | Your Twitch Application Client ID |
| **Client Secret** | Your Twitch Application Client Secret |
| **Sub Secret** | A password of your choice used to validate EventSub webhook signatures |
| **Channel** | Default Twitch channel username to subscribe to |
| **Sub URI** | Your server hostname **without** `https://` (e.g. `your.domain.name` or `xxxx.ngrok.io`) |
| **Scopes** | Space-separated OAuth scopes (see recommendations below) |

> **Security:** The settings form stores credentials in Node-RED flow context. It is strongly recommended to secure your Node-RED editor with authentication and HTTPS.

*Recommended Scopes:*
```
bits:read channel:manage:broadcast channel:manage:polls channel:manage:predictions channel:manage:redemptions channel:read:polls channel:read:predictions channel:read:redemptions channel:read:subscriptions moderation:read user:read:follows user:read:subscriptions channel:moderate channel:read:hype_train
```

### 3. Authorize Tokens

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Auth.png" width="30%">

After submitting the Settings form:

1. Click **Authorize APP** to generate an App Access Token
2. Click **Authorize USER** to log in with your Twitch account and accept the requested scopes (User Access Token)

> Tokens can be validated using the **Test token** button in the Settings group.

### 4. Subscribe to Events

- Enter the channel username in the **Username** field
- Select the desired event from the **Subscription list** dropdown
- Click **Subscribe**

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Sub.png" width="50%">

### 5. Unsubscribe

- Click **Refresh** to load active subscriptions
- Select the **Channel** from the channels dropdown
- Select the **Active sub** you want to remove
- Click **Unsubscribe**

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Unsub.png" width="30%">

## Handling Events

When an event is received at `/webhook`, it is:
1. **Signature-verified** using HMAC SHA256 (with timestamp anti-replay check — messages older than 10 minutes are rejected with HTTP 403)
2. **Deduplicated** using the EventSub message ID (10-minute TTL cache)
3. **Routed** by message type: `webhook_callback_verification`, `notification`, or `revocation`

If you subscribed to multiple channels, you can filter events by broadcaster channel ID:

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Subfilter.png" width="25%">

Connect your own function/node to the annotated output of each event type handler in the **Response** tab.

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Responses.png" width="40%">

<img src="https://github.com/n3odym3/Node-RED_Twitch_EventSub/blob/main/pictures/Annotation.png" width="40%">

> The **Inject** nodes and **Fake event** functions in the Response tab can be removed — they exist for testing/debug purposes only.

## Security Recommendations

- Always run Node-RED behind HTTPS (reverse proxy with Caddy/Nginx, or Let's Encrypt)
- Enable [Node-RED editor authentication](https://nodered.org/docs/user-guide/runtime/securing-node-red#editor--admin-api-security)
- Use a strong, random **Sub Secret** (at least 16 characters)
- Restrict access to your Node-RED instance to trusted IPs where possible

**HAVE FUN** 🎮
