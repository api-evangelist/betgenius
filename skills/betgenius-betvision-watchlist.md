---
name: Drive the BetVision in-play watchlist and betslip
description: >-
  Wire a sportsbook's own bet state and betslip into the BetVision interactive video experience over
  the geniussportsmessagebus CustomEvent bus. Covers the requestUserBets/setUserBets handshake, the
  stateless overwrite semantics that silently drop markets, the multibet command set, and betslip
  positioning on rotation.
api: components/betgenius-components.yml
operations: []
surface: browser CustomEvent bus (geniussportsmessagebus)
---

# Drive the BetVision in-play watchlist and betslip

BetVision renders the operator's prices, markets and open bets over live video. It has **no REST
API** — the entire contract is a bidirectional DOM `CustomEvent` bus on `window` called
`geniussportsmessagebus`. There is no OpenAPI to ground this in; the TypeScript interfaces below are
the published contract, quoted from the provider's integration guide.

> Interactive Betting mode only unlocks "once your prices and markets are fully integrated into our
> system and you have completed your Betslip integration." Until then these events are inert.

## The envelope

Every message, in both directions, has the same shape:

```ts
type MessageBusEvent = {
  correlationId: string   // uuid
  routingKey: {}
  type: string
  body: any
}
```

Listen:

```js
window.addEventListener('geniussportsmessagebus', (event) => {
  switch (event.detail.type) {
    case 'player_ready': /* see betgenius-stream-live-fixture.md */ break
    case 'requestUserBets': /* below */ break
    case 'multibet-event': /* below */ break
    case 'betslip-container-dimensions': /* below */ break
  }
})
```

Dispatch:

```js
window.dispatchEvent(new CustomEvent('geniussportsmessagebus', {
  detail: { correlationId: 'guid', routingKey: {}, type: 'setUserBets', body: markets }
}))
```

## 1. Wait for `requestUserBets` before you push anything

BetVision emits `requestUserBets` (body `null`) when it is ready to receive the viewer's bets.

**Ordering matters and the failure is silent:** if you call `setUserBets` before `player_ready` or
before the first `requestUserBets`, "the BetVisionUI will not be populated as it has not finished
loading." Nothing errors. Queue your state and flush it on the event.

## 2. Push the watchlist — `setUserBets`

Body is `CustomerMarket[]`:

```ts
type CustomerMarket = {
  id: string
  type: 'Single' | 'Multi'
  status: 'Open' | 'Settled'
  selections: Selection[]
  currency: string
  payout: number
  stakePerUnit: number
  cashOutAvailable: boolean
  cashOutPrice: number
  addedAt: number          // timestamp
}

type Selection = {
  sport: string
  competitionName: string
  selectionId: string
  status: 'Won' | 'Lost' | 'Open'
  selectionName: string
  marketId: string
  marketType: string
  fixtureId: string
  fixtureName: string
  fixtureStartTime: number  // timestamp
  price: {
    numerator: number
    denominator: number
    decimal: number
    american: string
    outcome?: string        // 'Under' | 'Over' — over/under bets only
    handicap?: string       // the over/under line — over/under bets only
  }
}
```

**The single thing that breaks integrations here:** the UI is stateless and `setUserBets` is a full
overwrite. "The setUserBets body completely overwrites the state of the Watchlist... if a market is
sent on one event dispatch but it is not sent on another, the UI will remove it from the Watchlist."

Never send a delta. Always send the viewer's complete current set of markets, every time — including
settled ones you still want shown. Re-send on every change to your own bet state.

Note the price object requires **three** representations of the same odds (fractional, decimal,
American). Compute all three; the player does not convert for you.

## 3. Handle betting commands — `multibet-event`

The player asks your sportsbook to act. `detail.body.command` is one of:

| Command | What you do |
|---|---|
| `addToBetslip` | Add the selection to the operator's betslip. |
| `placeBet` | Place the bet currently on the betslip. |
| `openBetslip` | Show your betslip UI. |
| `closeBetslip` | Hide it. |

`addToBetslip` carries the identifiers you need to resolve the selection in **your own** catalogue,
not just Genius ids:

```ts
interface BetslipEvent {
  command: 'addToBetslip'
  sportsbookFixtureId: string
  marketId: string
  sportsbookMarketId: string
  sportsbookMarketContext: string
  sportsbookSelectionId: string
  decimalPrice: number
}
```

Validate `decimalPrice` against your own live price before accepting — the event is a request from
a rendered overlay, not an authoritative price.

## 4. Position your betslip — `betslip-container-dimensions`

When Interactive Betting is enabled and the device rotates, BetVision tells you where and how big
your betslip may be:

```ts
interface BetsliptContainerDimensions { height: number; left: number; top: number; width: number }
```

`top`/`left` are the betslip's top-left corner relative to the viewport; `height`/`width` are the
space available to draw in. It is emitted only when the values actually change — the player tracks
the last values sent, so a rotation that produces identical geometry sends nothing. Cache the last
value; do not assume an event on every rotation.

## Correlation and debugging

`detail.correlationId` is a uuid present on every message and is the **only** correlation identifier
BetGenius exposes anywhere — none of the REST surfaces carry a request-id header. Log it on both
sides; it is what support will ask for.

## References

- Handling events: https://dap-docs.betstream.betgenius.com/bet-vision/handling-events
- User Bets API: https://dap-docs.betstream.betgenius.com/bet-vision/User-Bets/api
- Betslip in iframe: https://dap-docs.betstream.betgenius.com/bet-vision/integrating-betslip-inside-iframe
- Event surface: `asyncapi/betgenius-event-surface.yml`
- Components: `components/betgenius-components.yml`
