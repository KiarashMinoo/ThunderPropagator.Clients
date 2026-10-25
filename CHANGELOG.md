# Changelog

All notable changes to this project will be documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased] — 2026-10-25

### ⚠️ Known divergences from the server (spec not yet changed — breaking if resolved server-side)
- §4 Handshake: spec says the first frame is a `ConnectionResponse` JSON with `RequestResponseConfiguration` / `PushMessageConfiguration`; the current server sends `Key=Value` lines (`ConnectionId`, `ClientIPAddress`, `ClientPort`, `ConnectionDateTime`, `IsAvailable`, `User`) and neither configuration object.
- §7/§8 Request/Response: server parses requests as `Key=Value` (`RequestId`, `Route`, `Content-Type`, `Accept`, `FormData`) and pushes responses as `Key=Value` (`ResponseCode`, `ResponseContent`, ...).
- §5.4/§10 Push format: no client-side `PushMessageFormat` negotiation exists; the server uses its per-transport `SerializerType` (default NJson; Proprietary = "FastSerializer").
- §18 Codes: bad credentials return 403 (not 401); missing credential fields 406; duplicate `RequestId` 406.
- Additive server features not in the spec: `ClusterPeers` on Subscribe responses, 307/503 leader handling, `traceparent`/`tracestate`/`baggage` request keys, WebTransport and MQTT transports.
- Tracked in the .NET client repo (`.github/issue-drafts/`). Decide the contract before releasing the next spec version.

## [Unreleased] — 2026-05-13

### 📝 Documentation
- Expands and clarifies protocol spec documentation `(82adfb7)` — Kiarash Minoo
- Documents push/request format negotiation and new push formats `(e81b8c5)` — Kiarash Minoo
- Add protocol specification files `(2df2da0)` — Kiarash Minoo

### 🏠 Chores
- Initial commit `(ba459e6)` — Kiarash Minoo
