# ThunderPropagator Client Protocol Specification

> **This is the authoritative specification for all ThunderPropagator client libraries.**
> Every client — regardless of language or platform — **must** implement exactly this protocol.
> Deviations require an approved RFC against this repository.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Supported Protocols](#2-supported-protocols)
3. [Protocol Negotiation & IDS Fallback](#3-protocol-negotiation--ids-fallback)
4. [Connection Handshake](#4-connection-handshake)
5. [Authentication](#5-authentication)
6. [Channel Metadata](#6-channel-metadata)
7. [Request Format](#7-request-format)
8. [Response Format](#8-response-format)
9. [Subscription Model](#9-subscription-model)
10. [Subscription Push Message Formats](#10-subscription-push-message-formats)
11. [Record Status Codes](#11-record-status-codes)
12. [Splitter Escaping](#12-splitter-escaping)
13. [Heartbeat](#13-heartbeat)
14. [Reconnection & State Recovery](#14-reconnection--state-recovery)
15. [Configuration Reference](#15-configuration-reference)
16. [Mobile & Background Lifecycle](#16-mobile--background-lifecycle)
17. [Encryption Support](#17-encryption-support)
18. [Error Handling](#18-error-handling)
19. [Compliance Checklist](#19-compliance-checklist)
20. [Client Library Implementations](#20-client-library-implementations)

---

## 1. Overview

ThunderPropagator is a real-time data streaming platform. Clients connect to a server, request channel metadata, subscribe to data streams using key/field filters, and receive live push updates in one of the supported push message formats.

Control-plane requests and responses may be encoded as `Json`, `Yaml`, or `Xml`. The default push message format is the ThunderPropagator fast serializer format. Servers may also expose additional push formats such as JSON, YAML, and XML, and clients must be able to request and parse any format they declare support for.

The platform supports three transport protocols in priority order:

| Priority | Protocol | Transport | Works everywhere? |
|----------|----------|-----------|-------------------|
| 1 | **WebSocket** | TCP (ws:// / wss://) | Most environments |
| 2 | **QUIC** | UDP (HTTP/3) | Modern environments |
| 3 | **IDS** (InfiniteDataStream) | HTTP/1.1 (http:// / https://) | **Always — universal fallback** |

IDS is the backup protocol. **All clients must implement IDS** and must activate it automatically when higher-priority protocols fail. IDS works through all corporate proxies, firewalls, and NATs that support HTTP.

---

## 2. Supported Protocols

### 2.1 WebSocket

- **URL scheme:** `ws://` or `wss://`
- **Path:** same root path as HTTP server (no subpath needed)
- **Frame type:** Text frames only (UTF-8 structured request/response payloads and the negotiated push message format for subscription updates)
- **Library examples:** native browser API, `ws` (Node.js), `websockets` (Python), `gorilla/websocket` (Go), `tungstenite` (Rust), `URLSessionWebSocketTask` (Swift)

### 2.2 QUIC

- **URL scheme:** `quic://` (maps to same host/port as HTTPS endpoint)
- **Stream type:** Unidirectional inbound streams for server-push; bidirectional streams for request/response
- **Library examples:** `System.Net.Quic` (.NET), `quinn` (Rust), `quic-go` (Go), `aioquic` (Python), `Network.framework` (Swift/Apple), `msquic` (Windows/C++)

### 2.3 InfiniteDataStream (IDS)

IDS is a pure HTTP/1.1 streaming protocol. It uses two separate HTTP operations:

#### Downstream (server → client) — subscribe and receive push data

```
POST /thunderPropagator/{channelName}/subscribe
Content-Type: depends on negotiated request format:
  - `application/json; charset=utf-8` for `Json`
  - `application/yaml; charset=utf-8` for `Yaml`
  - `application/xml; charset=utf-8` for `Xml`
Body: { ...SubscribeRequest }

Response: 200 OK
Content-Type: depends on negotiated push message format:
  - `text/plain; charset=utf-8` for `FastSerializer`
  - `application/x-ndjson; charset=utf-8` for `Json`
  - `application/yaml; charset=utf-8` for `Yaml`
  - `application/xml; charset=utf-8` for `Xml`
Transfer-Encoding: chunked
[newline-delimited messages stream indefinitely until client disconnects]
```

Each message is a single line terminated by `\n`. The client reads lines indefinitely from the response body and parses each line using the negotiated push message format.

> **Note:** The channel can also be addressed by its `Guid` key:
> `POST /thunderPropagator/{channelKey:guid}/subscribe`

#### Channel metadata (IDS only — WebSocket/QUIC use the request protocol)

```
GET /channel/{channelName}/metadata
Accept: depends on negotiated request/response format:
  - `application/json` for `Json`
  - `application/yaml` for `Yaml`
  - `application/xml` for `Xml`
Response: 200 OK, Content-Type: same as negotiated request/response format
Body: ChannelMetadata in the negotiated structured format
```

---

## 3. Protocol Negotiation & IDS Fallback

### 3.1 Startup negotiation

Clients attempt protocols in the configured priority order. The first successful connection wins.

```
Start
  │
  ├─ Try WebSocket (probe timeout: configurable, default 3 s)
  │   → Success: use WebSocket
  │   → Fail: next
  │
  ├─ Try QUIC (probe timeout: configurable, default 2 s)
  │   → QUIC available in runtime? Yes → try it
  │   → Success: use QUIC
  │   → Fail or QUIC unavailable: next
  │
  └─ IDS (HTTP POST — always succeeds if server is reachable)
      → Connect and stream
```

**Rule:** IDS is always the final fallback regardless of the configured `protocols` list. Setting `idsAsLastResort: false` is explicitly not supported — IDS must always be available.

### 3.2 Pre-warmed shadow IDS connection

When a client is connected via WebSocket or QUIC, it **must** maintain a lightweight background IDS connection (probe-only, not subscribed). This ensures that IDS is available **instantly** when the primary protocol fails — no new TCP handshake or TLS negotiation is needed at the moment of failure.

### 3.3 Active failover

Protocol failure is detected via missed heartbeat (see §13). On failure:

```
WebSocket missed heartbeat
  │
  ├─ Activate pre-warmed IDS connection
  ├─ Re-apply all active subscriptions over IDS
  ├─ Log: "Primary protocol (WebSocket) timed out. Switched to IDS."
  └─ Fire onProtocolChanged(from: "WebSocket", to: "IDS")
```

Failover **must complete within 100 ms** of declaring the primary protocol dead.

### 3.4 Automatic upgrade on recovery (autoUpgrade)

When `autoUpgrade` is enabled (recommended default: `true`), the client periodically probes the preferred protocol:

```
Every upgradeCheckIntervalMs (default: 30 000 ms):
  If currently on IDS:
    → Probe WebSocket endpoint
    → Success: migrate subscriptions, switch to WebSocket, fire onProtocolChanged
    → Fail: stay on IDS, log "Primary still unavailable"
```

Upgrade is seamless — subscriptions are silently re-applied over the new protocol before the old one is closed.

---

## 4. Connection Handshake

### 4.1 WebSocket / QUIC handshake

1. Client opens a connection.
2. **Server sends the first message** — a JSON `ConnectionResponse`:

```json
{
  "ConnectionId": "550e8400-e29b-41d4-a716-446655440000",
  "IsAvailable": true,
  "RequestResponseConfiguration": {
    "MessageFormat": "Json",
    "SupportedMessageFormats": ["Json", "Yaml", "Xml"]
  },
  "PushMessageConfiguration": {
    "MaxPushSize": 65536,
    "MessageFormat": "FastSerializer",
    "SupportedMessageFormats": ["FastSerializer", "Json", "Yaml", "Xml"]
  }
}
```

- `ConnectionId` — opaque string; must be stored and included in all subsequent requests.
- `IsAvailable` — if `false`, the server is at capacity; client should disconnect and retry.
- `RequestResponseConfiguration.MessageFormat` — the effective structured format for requests, responses, and metadata payloads after the bootstrap handshake.
- `RequestResponseConfiguration.SupportedMessageFormats` — the structured formats the server accepts for requests and emits for responses.
- `PushMessageConfiguration.MaxPushSize` — maximum bytes the server will send in a single push chunk. The client uses this for buffer sizing.
- `PushMessageConfiguration.MessageFormat` — the effective push message format for this connection. Clients must use this value when decoding subscription updates.
- `PushMessageConfiguration.SupportedMessageFormats` — the formats the server can emit on this connection.

3. The client **must ignore** any message that starts with `PROBE`. These are server-side liveness probes.

4. All subsequent messages from the server are either:
  - **Response messages** (in the negotiated structured message format, in response to a client request)
  - **Push messages** (emitted in the negotiated push message format, unsolicited from feeder channels)

### 4.2 IDS handshake

IDS has no persistent connection. Each subscription is a separate long-running HTTP POST. There is no `ConnectionResponse`. The client generates its own `ConnectionId` (UUID v4) for IDS sessions.

---

## 5. Authentication

Authentication credentials are included **per-request** in the request JSON body — not in HTTP headers. The server enforces authentication at the channel subscription level.

### 5.1 No authentication

Omit `Token`, `Username`, and `Password` from the request body.

### 5.2 OAuth2 Bearer Token

```json
{
  "Token": "eyJhbGciOiJSUzI1NiJ9...",
  "Username": null,
  "Password": null
}
```

### 5.3 Basic Authentication (RSA-encrypted)

Credentials are RSA-encrypted using the server's public key before transmission. The channel metadata response includes `Authentication.IsEnabled`, `Authentication.AuthenticationType`, and the public key details.

```json
{
  "Token": null,
  "Username": "<RSA-encrypted-base64-username>",
  "Password": "<RSA-encrypted-base64-password>"
}
```

**Client must:**
1. Request channel metadata first (see §6).
2. Read `Authentication.AuthenticationType` from metadata.
3. Encrypt credentials using the provided RSA public key and key size.
4. Include encrypted values in all subsequent subscription requests.

### 5.4 Push message format negotiation

Push message format is negotiated per subscription. Clients request a preferred format in the subscribe request, and the server either accepts it or falls back to `FastSerializer`.

Rules:
1. Every client **must** support `FastSerializer`.
2. Clients **should** support `Json`.
3. Clients **may** support `Yaml` and `Xml` when their target platform benefits from human-readable or schema-oriented payloads.
4. If the requested format is unavailable, the server must use `FastSerializer`.
5. The effective format is reported in `PushMessageConfiguration.MessageFormat` and applies to every push message for that subscription/connection.

### 5.5 Request/response message format negotiation

Structured control-plane payloads use a separate negotiated format from push payloads.

Rules:
1. Requests, responses, and metadata payloads must support `Json`.
2. Servers may additionally support `Yaml` and `Xml`.
3. `FastSerializer` must not be used for requests, responses, or metadata.
4. Clients choose a preferred structured format via configuration and HTTP headers where applicable.
5. The effective structured format is reported in `RequestResponseConfiguration.MessageFormat`.

---

## 6. Channel Metadata

Before subscribing, the client must request channel metadata. Metadata describes the channel's field schema, authentication requirements, and encryption settings.

### 6.1 Via WebSocket / QUIC

Send a `MetadataRequest` (see §7). The server responds with a standard `Response` (see §8) where `ResponseContent` contains the `Metadata` payload in the negotiated structured format.

### 6.2 Via IDS

```
GET /channel/{channelName}/metadata
Accept: depends on negotiated request/response format:
  - `application/json` for `Json`
  - `application/yaml` for `Yaml`
  - `application/xml` for `Xml`
```

Response body is the raw `ChannelMetadata` payload in the negotiated structured format.

### 6.3 Metadata structure

```json
{
  "ChannelName": "Clock",
  "Authentication": {
    "IsEnabled": false,
    "AuthenticationType": "None"
  },
  "MessageEncryption": {
    "IsEnabled": false
  },
  "ChannelProgramsDescriptors": {
    "0": { "Index": 0, "Name": "UserId", "IsSubscribingKey": true, "Table": "UserId" },
    "1": { "Index": 1, "Name": "Date",   "IsSubscribingKey": false, "Type": "DateTime" },
    "2": { "Index": 2, "Name": "Time",   "IsSubscribingKey": false, "Type": "TimeSpan" }
  }
}
```

**Key fields:**
- `ChannelProgramsDescriptors` — maps field indices (integers as strings) to field definitions.
- `IsSubscribingKey: true` — this field is used as a subscription key (filter).
- `Index` — the integer index used in `FastSerializer` push values (see §10).

---

## 7. Request Format

All requests sent by the client (WebSocket and QUIC only; IDS uses HTTP) use the negotiated structured message format. Supported values are `Json`, `Yaml`, and `Xml`. `FastSerializer` is not valid for requests.

The logical request shape is:

```json
{
  "RequestId": "c3d4e5f6-a7b8-...",
  "Route": {
    "Channel": "Clock",
    "RequestType": "Metadata"
  },
  "Token": null,
  "Username": null,
  "Password": null
}
```

JSON example:

```json
{
  "RequestId": "c3d4e5f6-a7b8-...",
  "Route": {
    "Channel": "Clock",
    "RequestType": "Metadata"
  },
  "Token": null,
  "Username": null,
  "Password": null
}
```

YAML example:

```yaml
RequestId: c3d4e5f6-a7b8-...
Route:
  Channel: Clock
  RequestType: Metadata
Token: null
Username: null
Password: null
```

XML example:

```xml
<Request>
  <RequestId>c3d4e5f6-a7b8-...</RequestId>
  <Route>
    <Channel>Clock</Channel>
    <RequestType>Metadata</RequestType>
  </Route>
  <Token />
  <Username />
  <Password />
</Request>
```

### 7.1 RequestId

A unique identifier (UUID v4) for this request. The server echoes it back in the corresponding `Response.RequestId`. Clients use this to correlate responses to requests.

### 7.2 Route

- `Channel` — the channel name (e.g., `"Clock"`, `"Chat"`, `"ResourceMonitoring"`).
- `RequestType` — one of the request types below.

### 7.3 Standard request types

| RequestType | Description |
|-------------|-------------|
| `Metadata` | Request channel metadata (§6) |
| `Ping` | Heartbeat ping to server |
| `Subscribe` | Subscribe to a data stream (§9) |
| `Unsubscribe` | Cancel a subscription |

### 7.4 Subscribe request (additional fields)

```json
{
  "RequestId": "...",
  "Route": { "Channel": "Clock", "RequestType": "Subscribe" },
  "SubscribingKeys": [{ "UserId": "alice" }],
  "SubscribingFields": ["Date", "Time"],
  "SubscriptionMode": "Full"
}
```

- `SubscribingKeys` — array of key-value dictionaries. Each entry subscribes to one key combination.
- `SubscribingFields` — array of field names to receive. Empty array = all fields.
- `SubscriptionMode` — `"Full"` (all field values on every update) or `"Changes"` (only changed fields).
- `PushMessageFormat` — preferred push message format. Supported values: `"FastSerializer"`, `"Json"`, `"Yaml"`, `"Xml"`. If omitted, the default is `"FastSerializer"`.

Example:

```json
{
  "RequestId": "...",
  "Route": { "Channel": "Clock", "RequestType": "Subscribe" },
  "SubscribingKeys": [{ "UserId": "alice" }],
  "SubscribingFields": ["Date", "Time"],
  "SubscriptionMode": "Full",
  "PushMessageFormat": "Json"
}
```

The same logical subscribe payload may also be encoded as YAML or XML when the negotiated structured format is `Yaml` or `Xml`.

### 7.5 Unsubscribe request (additional fields)

```json
{
  "RequestId": "...",
  "Route": { "Channel": "Clock", "RequestType": "Unsubscribe" },
  "SubscribingKeys": [{ "UserId": "alice" }]
}
```

### 7.6 Pipeline requests (channel-specific)

Channels may expose custom `RequestType` values via receive pipelines (e.g., `"Users/Login"`, `"Messages/Send"` for the Chat channel). These follow the same base structure with additional channel-specific fields.

---

## 8. Response Format

The server sends responses using the negotiated structured message format. Supported values are `Json`, `Yaml`, and `Xml`. `FastSerializer` is not valid for responses.

The logical response shape is:

```json
{
  "ConnectionId": "550e8400-...",
  "RequestId":    "c3d4e5f6-...",
  "ResponseCode": 200,
  "ResponseContent": "...",
  "Route": {
    "Channel": "Clock",
    "RequestType": "Metadata"
  }
}
```

- JSON, YAML, and XML responses carry the same logical fields.
- `ResponseCode` — HTTP-style status code (200 OK, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, etc.).
- `ResponseContent` — structured payload or plain string payload; meaning depends on `RequestType`.
- For `Metadata` requests, `ResponseContent` contains the metadata payload in the negotiated structured format.
- For `Ping` requests, `ResponseContent` contains server info in the negotiated structured format.
- For `Subscribe` / `Unsubscribe`, `ResponseContent` is `"Subscribed"` or `"Unsubscribed"`.

---

## 9. Subscription Model

### 9.1 Subscription lifecycle

```
createSubscription(keys, fields, mode)
  → subscribe()       → server acknowledges
  → [push messages arrive]
  → unsubscribe()     → server acknowledges
```

### 9.2 Multiple keys

A client may subscribe to multiple key combinations in a single `createSubscriptions()` call. The server treats each key combination as an independent subscription internally, but the client API groups them.

### 9.3 Snapshot on subscribe

Upon successful subscription, the server immediately sends the current snapshot — all existing data matching the subscription keys. These push messages have `FromSnapshot: true` (see §10).

### 9.4 Subscription state machine

Clients must maintain a canonical `SubscriptionRegistry` independent of the transport protocol. On protocol switch, all entries are re-submitted as fresh subscribe requests over the new transport. The server will re-send the snapshot, and callbacks fire normally.

```
States: Initiated → Subscribing → Subscribed → Unsubscribing → Unsubscribed
                                             ↘ Error
```

---

## 10. Subscription Push Message Formats

Push messages are transport-agnostic — identical whether received over WebSocket, QUIC, or IDS. The format is selected during subscription creation.

### 10.1 Supported formats

All clients **must** support the following baseline push message format:

| Format | Required | Description |
|--------|----------|-------------|
| `FastSerializer` | Yes | Proprietary compact text format optimized for throughput and payload size |
| `Json` | Recommended | Line-delimited JSON payloads optimized for interoperability and debugging |
| `Yaml` | Optional | Line-delimited YAML payloads optimized for human readability and configuration-style consumers |
| `Xml` | Optional | Line-delimited XML payloads optimized for schema-driven and enterprise integrations |

If a client requests an unsupported format, the server falls back to `FastSerializer`.

### 10.2 FastSerializer format

This is the existing ThunderPropagator compact text format.

```
{requestId},{fromSnapshot},{recordStatus},{key1|key2|...},{index=value;index=value;...}
```

### 10.3 FastSerializer fields

| Position | Name | Type | Description |
|----------|------|------|-------------|
| 1 | `requestId` | string | The `RequestId` from the original Subscribe request. Routes the message to the correct subscription. |
| 2 | `fromSnapshot` | `"1"` or `"0"` | `"1"` = from initial snapshot; `"0"` = live update |
| 3 | `recordStatus` | char | Record change type (see §11) |
| 4 | `keys` | pipe-separated strings | Subscription key values in the same order as the subscribing keys |
| 5 | `values` | semicolon-separated `index=value` pairs | Field values keyed by their descriptor index (integer from ChannelProgramsDescriptors) |

### 10.4 FastSerializer parsing regex

```
^(.+?),(.+?),(.+?),(.+?),(.*)$
```

Groups:
- `[1]` = requestId
- `[2]` = fromSnapshot (`"1"` or `"0"`)
- `[3]` = recordStatus
- `[4]` = pipe-separated keys
- `[5]` = semicolon-separated index=value pairs

### 10.5 FastSerializer parsing values (group 5)

Split group 5 by `;`, then split each part by `=`:

```
"1=2026-05-10;2=14:30:00" → { 1: "2026-05-10", 2: "14:30:00" }
```

Map integer indices to field names using `ChannelProgramsDescriptors`.

### 10.6 FastSerializer example

```
sub-req-uuid-1234,0,M,alice,1=2026-05-10;2=14:30:00.123
```

Decoded:
- `requestId` = `sub-req-uuid-1234`
- `fromSnapshot` = `false` (live update)
- `recordStatus` = `M` (Modified)
- `keys` = `["alice"]`
- `values` = `{ "Date": "2026-05-10", "Time": "14:30:00.123" }`

### 10.7 Json format

When `PushMessageFormat` is `Json`, each pushed line is a single JSON object with the following shape:

```json
{
  "RequestId": "sub-req-uuid-1234",
  "FromSnapshot": false,
  "RecordStatus": "M",
  "Keys": ["alice"],
  "Values": {
    "Date": "2026-05-10",
    "Time": "14:30:00.123"
  }
}
```

Rules:
- `RequestId` maps to the original subscribe request.
- `FromSnapshot` is a boolean.
- `RecordStatus` uses the same codes defined in §11.
- `Keys` preserves the subscribing-key order defined by the channel metadata.
- `Values` uses field names, not descriptor indices.
- Each JSON object is newline-delimited when transported over IDS.

### 10.8 Yaml format

When `PushMessageFormat` is `Yaml`, each pushed line is a single YAML document with the following shape:

```yaml
RequestId: sub-req-uuid-1234
FromSnapshot: false
RecordStatus: M
Keys:
  - alice
Values:
  Date: 2026-05-10
  Time: "14:30:00.123"
```

Rules:
- `Yaml` carries the same logical fields as `Json`.
- Field names are case-sensitive and must match the JSON property names exactly.
- Each YAML document is newline-delimited when transported over IDS.
- Clients must use a safe YAML parser and must not permit custom tags or object instantiation.

### 10.9 Xml format

When `PushMessageFormat` is `Xml`, each pushed line is a single XML document with the following shape:

```xml
<PushMessage>
  <RequestId>sub-req-uuid-1234</RequestId>
  <FromSnapshot>false</FromSnapshot>
  <RecordStatus>M</RecordStatus>
  <Keys>
    <Key>alice</Key>
  </Keys>
  <Values>
    <Date>2026-05-10</Date>
    <Time>14:30:00.123</Time>
  </Values>
</PushMessage>
```

Rules:
- `Xml` carries the same logical fields as `Json`.
- The root element must be `PushMessage`.
- Repeated key values must be emitted as `<Key>` child elements under `<Keys>`.
- Values are emitted using field names as XML element names under `<Values>`.
- Each XML document is newline-delimited when transported over IDS.

---

## 11. Record Status Codes

| Code | Constant | Meaning |
|------|----------|---------|
| `A` | Added | Record newly created |
| `M` | Modified | One or more field values changed |
| `D` | Deleted | Record removed (values array will be empty) |
| `N` | Neutral | No change — included for informational purposes |

---

## 12. Splitter Escaping

Splitter escaping applies only to the `FastSerializer` push message format. That format uses `,`, `|`, and `;` as delimiters. If a **field value** contains any of these characters, the server replaces them with escape sequences before transmission. Clients **must** restore them after parsing.

| Escape Sequence | Original Character |
|-----------------|--------------------|
| `<C>` | `,` (comma) |
| `<PI>` | `\|` (pipe) |
| `<SC>` | `;` (semicolon) |

**Apply restoration to all string values in groups [1], [4], and [5] after splitting.**

Example:
```
<C>  → ,
<PI> → |
<SC> → ;
```

---

## 13. Heartbeat

The client is responsible for detecting connection liveness.

### 13.1 Ping request

Every `heartbeatIntervalMs` (default: 15 000 ms), the client sends a `Ping` request:

```json
{ "RequestId": "ping-uuid", "Route": { "Channel": "Clock", "RequestType": "Ping" } }
```

The server responds with a standard Response containing server diagnostics.

### 13.2 Timeout detection

If no response is received within `heartbeatTimeoutMs` (default: 5 000 ms), the primary protocol is declared dead and IDS failover activates.

### 13.3 IDS heartbeat

On IDS, the client sends `GET /thunderPropagator/ping` every `heartbeatIntervalMs`. A non-200 response triggers IDS reconnection.

---

## 14. Reconnection & State Recovery

When any transport connection drops:

1. **Detect** — heartbeat timeout or TCP/WebSocket close event.
2. **Failover** — activate IDS within 100 ms (pre-warmed).
3. **Re-subscribe** — for every entry in the `SubscriptionRegistry`, send a fresh `Subscribe` request over the new transport.
4. **Snapshot** — server resends the full snapshot; callbacks fire normally. `FromSnapshot` will be `true` for snapshot entries.
5. **Resume** — `onProtocolChanged` fires; application code is unaware of the transport switch.

**Exponential backoff** applies to reconnection attempts:
- Base delay: 1 s
- Max delay: 60 s
- Formula: `min(baseDelay * 2^attempt, maxDelay)`

---

## 15. Configuration Reference

All client libraries must expose the following configuration options with these exact names (adapted to language conventions — e.g., `camelCase` in JS, `snake_case` in Python):

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `url` | string | required | Server URL (ws://, wss://, https://) |
| `protocols` | string[] | `["WebSocket","QUIC","IDS"]` | Protocol preference order |
| `idsAsLastResort` | bool | `true` | IDS always available as final fallback |
| `autoUpgrade` | bool | `true` | Probe preferred protocol and upgrade when available |
| `upgradeCheckIntervalMs` | int | `30000` | How often to probe for upgrade |
| `heartbeatIntervalMs` | int | `15000` | How often to send heartbeat ping |
| `heartbeatTimeoutMs` | int | `5000` | Timeout before declaring protocol dead |
| `probeTimeoutMs` | int | `3000` | Timeout for protocol probe during negotiation |
| `reconnectBaseDelayMs` | int | `1000` | Base delay for reconnection backoff |
| `reconnectMaxDelayMs` | int | `60000` | Maximum reconnection delay |
| `messageFormat` | string | `"Json"` | Preferred structured format for requests, responses, and metadata. Supported values: `"Json"`, `"Yaml"`, `"Xml"` |
| `pushMessageFormat` | string | `"FastSerializer"` | Preferred push message format for subscriptions. Supported values: `"FastSerializer"`, `"Json"`, `"Yaml"`, `"Xml"` |
| `auth` | object | `null` | Authentication — `{ type: "Bearer", token }` or `{ type: "Basic", username, password }` |
| `backgroundBehaviour` | enum | `SwitchToIDS` | Mobile: what to do when app is backgrounded (`SwitchToIDS`, `Disconnect`, `KeepAlive`) |
| `backgroundHeartbeatIntervalMs` | int | `60000` | Heartbeat interval when app is in background |

---

## 16. Mobile & Background Lifecycle

Mobile clients (iOS, Android, Flutter) **must** handle the application lifecycle:

### 16.1 Entering background (app minimised / screen off)

1. Stop sending WebSocket/QUIC heartbeats.
2. Switch to IDS transport (HTTP works through OS background restrictions).
3. Increase heartbeat interval to `backgroundHeartbeatIntervalMs`.
4. Register background task with the OS if the platform supports it (iOS `BGProcessingTask`, Android `WorkManager`).

### 16.2 Returning to foreground

1. Immediately probe the preferred protocol.
2. Upgrade back from IDS to WebSocket/QUIC if available.
3. Resume normal `heartbeatIntervalMs`.

### 16.3 Network transition (WiFi → Cellular)

On network interface change, re-negotiate the protocol from scratch. Do not assume the existing connection survived the transition.

### 16.4 watchOS / tvOS / visionOS

- **watchOS**: IDS only after 30 s in background. Short subscription lifetimes.
- **tvOS**: No background restrictions — behaves like macOS.
- **Widgets / App Clips**: IDS only; no long-lived connections.

---

## 17. Encryption Support

If `MessageEncryption.IsEnabled` is `true` in channel metadata:

1. Read the encryption key and key size from `MessageEncryption` metadata.
2. All received push messages are RSA-encrypted. Decrypt using the provided key before parsing the negotiated push message format (§10).
3. All authentication credentials are RSA-encrypted before transmission (§5.3).

Clients that do not implement encryption must reject channels with `MessageEncryption.IsEnabled: true` and throw a `FeatureNotSupportedException`.

---

## 18. Error Handling

### 18.1 Response codes

| Code | Meaning | Client action |
|------|---------|---------------|
| 200 | OK | Process response |
| 400 | Bad request | Log error; do not retry same request |
| 401 | Unauthorized | Re-authenticate; retry once |
| 403 | Forbidden | Log; do not retry |
| 404 | Channel not found | Throw `ChannelNotFoundException` |
| 405 | Method not allowed | Log; file bug |
| 422 | Validation error | Extract message from `ResponseContent`; throw |
| 500 | Server error | Exponential backoff retry |
| 503 | Unavailable | Exponential backoff retry |

### 18.2 Connection errors

- **DllNotFoundException / native library absent** (applies to .NET client): catch, log warning, treat all features as unavailable. Do not crash.
- **Protocol probe failure**: move to next protocol silently; log at `Debug` level.
- **IDS body interrupted**: reconnect IDS immediately; re-apply subscriptions.

### 18.3 Invalid push message payload

If a received push message cannot be parsed according to the negotiated format in §10, the client must:
1. Log the raw message at `Warning` level.
2. Skip the message (do not crash the subscription).
3. Increment a metric counter `thunderpropagator.receive.parse_errors_total`.

---

## 19. Compliance Checklist

Every client library version must pass all of the following before release:

### Connection
- [ ] Implements WebSocket transport
- [ ] Implements IDS transport (HTTP POST streaming)
- [ ] Implements QUIC transport (or explicitly declares it as unsupported for the platform)
- [ ] Protocol negotiation follows the priority order in §3.1
- [ ] IDS is always available as final fallback (cannot be disabled)
- [ ] Pre-warmed shadow IDS connection maintained when on WebSocket/QUIC
- [ ] Failover to IDS completes within 100 ms of heartbeat timeout
- [ ] All subscriptions re-applied automatically after protocol switch
- [ ] `onProtocolChanged` fires on every protocol switch
- [ ] `autoUpgrade` probe restores preferred protocol when available

### Handshake
- [ ] First message from server is parsed as `ConnectionResponse` JSON
- [ ] `RequestResponseConfiguration.MessageFormat` stored and used for control-plane payload serialization
- [ ] `PROBE` messages are silently ignored
- [ ] `ConnectionId` stored and included in all requests
- [ ] `IsAvailable: false` causes disconnection with retry

### Authentication
- [ ] No-auth mode supported
- [ ] Bearer token mode supported
- [ ] Basic auth with RSA encryption supported (or declared unsupported)
- [ ] Auth credentials sent per-request in request body, not HTTP headers

### Metadata
- [ ] Channel metadata requested before first subscription
- [ ] Metadata cached and reused for the session
- [ ] `ChannelProgramsDescriptors` parsed and used for `FastSerializer` decoding

### Subscriptions
- [ ] Subscribe request includes `SubscribingKeys`, `SubscribingFields`, `SubscriptionMode`
- [ ] Subscribe request supports `PushMessageFormat`
- [ ] Request payloads support `Json` and optionally `Yaml` / `Xml`
- [ ] `RequestId` is a unique UUID v4 per request
- [ ] Response correlated to request via `RequestId`
- [ ] Snapshot messages (`FromSnapshot: true`) delivered to callbacks
- [ ] `onFieldUpdated` callback fires for each changed field
- [ ] `onTableUpdated` callback fires with table, key, items, status
- [ ] Unsubscribe request sent on `unsubscribe()` call

### Push message formats
- [ ] `FastSerializer` supported
- [ ] `Json` supported or explicitly declared unsupported for the platform/version
- [ ] `Yaml` supported or explicitly declared unsupported for the platform/version
- [ ] `Xml` supported or explicitly declared unsupported for the platform/version
- [ ] Requested `PushMessageFormat` is sent during subscription creation
- [ ] Effective push message format read from `PushMessageConfiguration.MessageFormat`
- [ ] `FastSerializer` regex `^(.+?),(.+?),(.+?),(.+?),(.*)$` applied correctly
- [ ] `FastSerializer` `fromSnapshot` decoded as boolean (`"1"` = true)
- [ ] `FastSerializer` `recordStatus` decoded as A/M/D/N enum
- [ ] `FastSerializer` keys split by `|`
- [ ] `FastSerializer` values split by `;` then by `=` as index=value pairs
- [ ] `FastSerializer` splitter escapes `<C>`, `<PI>`, `<SC>` restored after splitting
- [ ] `FastSerializer` field indices mapped to names via `ChannelProgramsDescriptors`
- [ ] `Json` payloads decoded with `RequestId`, `FromSnapshot`, `RecordStatus`, `Keys`, `Values`
- [ ] `Yaml` payloads decoded with `RequestId`, `FromSnapshot`, `RecordStatus`, `Keys`, `Values`
- [ ] `Xml` payloads decoded with `RequestId`, `FromSnapshot`, `RecordStatus`, `Keys`, `Values`
- [ ] Invalid push payload logged and skipped (no crash)

### Request/response formats
- [ ] `Json` request/response payloads supported
- [ ] `Yaml` request/response payloads supported or explicitly declared unsupported for the platform/version
- [ ] `Xml` request/response payloads supported or explicitly declared unsupported for the platform/version
- [ ] `FastSerializer` never used for requests/responses
- [ ] Effective structured message format read from `RequestResponseConfiguration.MessageFormat`

### Heartbeat & reconnection
- [ ] Ping sent every `heartbeatIntervalMs`
- [ ] Heartbeat timeout triggers IDS failover
- [ ] Reconnection uses exponential backoff
- [ ] Reconnection re-applies all subscriptions

### Mobile (if applicable)
- [ ] Background event switches to IDS
- [ ] Foreground event triggers `autoUpgrade` probe immediately
- [ ] Network interface change triggers protocol re-negotiation

### Configuration
- [ ] All options in §15 are supported with the specified defaults
- [ ] Language-idiomatic naming used (camelCase, snake_case, PascalCase as appropriate)

### Observability
- [ ] `activeProtocol` property exposed (string: `"WebSocket"`, `"QUIC"`, `"IDS"`)
- [ ] `onProtocolChanged(from, to)` callback/event
- [ ] Error logging via platform-standard logger (structured where possible)

---

## 20. Client Library Implementations

| Language / Platform | Package | Repository | Status |
|---------------------|---------|------------|--------|
| **.NET** (C#) | `ThunderPropagator.Clients.DotNet` | [Clients.DotNet](https://github.com/KiarashMinoo/ThunderPropagator.Clients.DotNet) | ✅ Reference implementation |
| **JavaScript / TypeScript** | `@thunderpropagator/client` (npm) | Clients.JS | 🔲 Planned |
| **Python** | `thunderpropagator-client` (PyPI) | Clients.Python | 🔲 Planned |
| **Java** | `com.thunderpropagator:client` (Maven) | Clients.Java | 🔲 Planned |
| **Go** | `github.com/KiarashMinoo/thunderpropagator-go` | Clients.Go | 🔲 Planned |
| **Rust** | `thunderpropagator-client` (crates.io) | Clients.Rust | 🔲 Planned |
| **C++** | `thunderpropagator-cpp` (vcpkg) | Clients.Cpp | 🔲 Planned |
| **Swift / Objective-C** | SPM package | Clients.Swift | 🔲 Planned |
| **Flutter / Dart** | `thunderpropagator_client` (pub.dev) | Clients.Flutter | 🔲 Planned |

---

## Contributing

To propose changes to this specification:

1. Open an issue in this repository describing the change and the motivation.
2. Changes that break wire-format compatibility require a major version bump.
3. All client library maintainers must acknowledge the change before merging.
4. After merge, client libraries have 30 days to implement and release the change.

---

*© ThunderPropagator — MIT License*
