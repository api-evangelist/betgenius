---
name: Stream a live BetGenius fixture to a bettor
description: >-
  Take a viewer from "I want to watch this match" to a playing stream using the BetGenius Video
  Streaming API v3 and the Genius Live Player. Covers the player_ready handshake, why the API must
  never be called from the browser, multi-CDN failover, DRM, and the silent geo/DMA failure mode.
api: openapi/betgenius-video-v3-openapi.yml
operations:
  - getFixtures
  - getFixture
  - getRegions
  - createHLSStream
  - createDASHStream
  - createSRTStream
---

# Stream a live BetGenius fixture to a bettor

The video flow is a three-party handshake: the **player** in the bettor's browser decides what can
be played, your **backend** exchanges that decision for an authorised URL, and the player consumes
it. The API sits in the middle and is never touched by the browser.

> **Non-negotiable:** "This API must never be called from end-user browser/client code. Instead, the
> API should be integrated into server-side systems." Your API key and bearer token must never reach
> a client.

## Before you start

- Base URL: `https://api.geniussports.com/Video-v3/PRODPRM` (UAT spec at
  `https://explorer.api.geniussports.com/Video/v3/UAT/swagger-latest.json`).
- Auth: **two headers, both required** — `Authorization` (Cognito bearer token) and `x-api-key`
  (issued API key, which is what carries your quota).
- Errors are **RFC 7807** `application/problem+json` with `{type, title, status, detail, instance}`.

## 1. Know your regions — `getRegions`

`GET /regions` returns `{code, name, continent}` for every supported playback region. You need these
codes because the `region` you send when minting a stream **must** be one of them. Cache this; it
changes rarely.

## 2. Find streamable fixtures — `getFixtures` / `getFixture`

`GET /fixtures` accepts `sportIds`, `competitionIds`, `phases`, `bookingStatus`, `fromDate`,
`toDate`, `before`, `after`, `limit`, and `showPermissions`.

It returns a `FixturesPage`: `{data: Fixture[], paging}` where `paging.cursors.before` /
`paging.cursors.after` are opaque cursors and `paging.previous` / `paging.next` are ready-made
endpoint URLs. Follow `next` rather than constructing your own cursors; `previous` is absent on the
first page.

Each `Fixture` carries `liveStreams[]` and `vodStreams[]`. Each stream carries its **own
entitlement, inline**:

- `permissions[]` — `{device, region, maxPlayerSizePercentage}`. Note `maxPlayerSizePercentage`: some
  rights deals cap how large the player may be drawn.
- `dmas[]` — the US Designated Market Areas in which the stream may be shown.
- `deliveries` — `{hls[], dash[], srt[]}`. `srt` is omitted entirely when the stream has none.

Pass `showPermissions` when you need those blocks. `getFixture` (`GET /fixtures/{id}`) returns one
fixture by Genius Sports fixture id.

## 3. Embed the player and wait for `player_ready`

Drop the loader in with the fixture and the viewer's identity:

```html
<script src="https://genius-live-player-production.betstream.betgenius.com/widgetLoader?customerId=YOUR_CUSTOMER_ID&fixtureId=YOUR_FIXTURE_ID&deviceId=END_USER_DEVICE_ID&digest=END_USER_DIGEST"></script>
<div id="geniusLivePlayer"></div>
```

`digest` is an **HMAC-SHA256** of the end user's device MAID and a shared secret issued by the Genius
onboarding team. Compute it server-side.

Then listen on the message bus:

```js
window.addEventListener('geniussportsmessagebus', (event) => {
  if (event.detail.type === 'player_ready') {
    const { streamId, deliveryType, deliveryId, geniusSportsFixtureId, device, region } = event.detail.body
    // send these to YOUR backend, which calls the Video API
  }
})
```

The player picks the stream and delivery appropriate to the device; you do not choose.

## 4. Mint the stream — `createHLSStream` / `createDASHStream` / `createSRTStream`

From your **backend**, using the ids the event gave you:

```
POST /fixtures/{id}/live-streams/{streamId}/deliveries/hls/{deliveryId}
POST /fixtures/{id}/live-streams/{streamId}/deliveries/dash/{deliveryId}
POST /fixtures/{id}/live-streams/{streamId}/deliveries/srt/{deliveryId}
```

Pick the verb from `deliveryType` on the event (`HLS` or `DASH`). Body:

| Field | Required | Meaning |
|---|---|---|
| `endUserSessionId` | yes | Non-PII id for one viewer session, stable for the length of that session. Max 1024 chars. |
| `device` | yes | The `Device` value from the event. |
| `region` | no | Region the viewer is in — must match a code from `getRegions`. |

Response: `{url, token, expiresAt, drm}`. `drm` carries `widevine`, `playready`, `fairplay` and
`fairplayCertificate` URLs when the content is protected.

Hand the whole response straight back to the player:

```js
GeniusLivePlayer.player.start(data)
```

## 5. Handle the failure modes

- **`player_not_ready`** carries `detail.body.error[]` with codes `1001` (region or device not
  allowed), `1002` (DMA zone not allowed), `1003` (no deliveries found), `1004` (video cannot be
  played — transient, retry later).
- **Silence is also a failure.** If the viewer is outside a permitted DMA, the edge blocks them and
  the player *never dispatches `player_ready` at all*. Your UI must handle "no event ever arrived"
  as a distinct state, not wait forever.
- **`player_ready` can fire more than once for one fixture.** On multi-CDN failover the player
  re-emits with a different `deliveryId`, asking for a fresh URL. Your handler must be re-entrant —
  do not assume one event per session.
- **403 is entitlement, not authentication.** The client is authenticated but not licensed for that
  fixture, sport, source, region or DMA. Do not retry; do not re-auth.
- **Clean up.** Call `GeniusLivePlayer.player.close()` and `window.removeEventListener` when tearing
  down, or the player keeps downloading video segments in the background.

## VOD

`createHlsVod` (`POST /fixtures/{id}/vod-streams/{streamId}/deliveries/hls/{deliveryId}`) is the same
exchange for video-on-demand. VOD streams support HLS and DASH only — there is no SRT VOD.

## References

- Player integration: https://dap-docs.betstream.betgenius.com/video-player/web-browser-integration
- Error handling: https://dap-docs.betstream.betgenius.com/video-player/error-handling
- Restrictions: https://dap-docs.betstream.betgenius.com/bet-vision/restrictions
- Components: `components/betgenius-components.yml`
- Errors: `errors/betgenius-problem-types.yml`
