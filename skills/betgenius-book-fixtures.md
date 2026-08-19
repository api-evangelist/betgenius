---
name: Book and un-book BetGenius fixtures
description: >-
  Discover upcoming fixtures a sportsbook is entitled to, book the ones it wants feeds for, and
  un-book them again — using the BetGenius Booking API. Covers the per-sport cooldown that makes
  naive polling fail and the absence of any idempotency contract on the write calls.
api: openapi/betgenius-booking-v2-openapi.yml
operations:
  - BookingV2_Fixtures
  - BookingV2_Book
  - BookingV2_UnBook
---

# Book and un-book BetGenius fixtures

Booking is how a sportsbook tells BetGenius which fixtures it wants data and video for. It is a
commercial act, not a query: booking a fixture turns on the feeds the operator is billed for.

## Before you start

- Base URL: `https://dataservices.betgenius.com/BookingSystem` (UAT:
  `https://dataservices.uat.betgenius.com/BookingSystem`).
- Auth: **HTTP Basic**. The username and password are issued by the Genius Sports Support team to a
  contracted operator. There is no signup and no self-service key.
- Use the **V2** path prefix `/api/v2/booking/...`. V1 (`/api/booking/...`) is still served and
  exposes the same three operations, but its `Feed` object lacks the `Metadata` map.
- Everything is keyed on the Genius Sports **fixture id** (int32).

## 1. Find bookable fixtures — `BookingV2_Fixtures`

`GET /api/v2/booking/Fixtures`

Query parameters (all optional in the contract; the docs describe `SportId` as the one you should
always send):

| Parameter | Type | Meaning |
|---|---|---|
| `sportId` | int32 | Restrict to one sport. `10` is Football (soccer), `17` is American Football. |
| `from` | date-time | Only fixtures starting after this instant. |
| `until` | date-time | Only fixtures starting before this instant. |

Returns an array of `Fixture`:
`FixtureId`, `Name`, `Date`, `CompetitionId`, `CompetitionName`, `SportId`, `CustomerFixtureId`,
`IsBooked`, `AvailableFeeds[]` (`{Name, Type, IsLicensed, Metadata}`), `ExternalIds[]`
(`{Source, Id}`).

**Two hard limits that are not in the spec, only in the docs — respect both:**

1. **A 10-minute cooldown per sport.** If you call `Fixtures` again for the same sport inside the
   window, the response tells you how much time is left rather than returning data. Do not build a
   polling loop tighter than this, and do not retry on an empty result.
2. **A maximum of 7 days of fixtures per call.** Page through longer horizons by moving the
   `from`/`until` window, not by asking for more.

Read `IsBooked` to decide whether a fixture needs action, and `AvailableFeeds[].IsLicensed` to know
whether the operator is actually entitled to the feed before booking against it.

## 2. Book a fixture — `BookingV2_Book`

`POST /api/v2/booking/Book?fixtureId={fixtureId}`

`fixtureId` is a **required query parameter**, not a body. There is no request body.

## 3. Un-book a fixture — `BookingV2_UnBook`

`POST /api/v2/booking/UnBook?fixtureId={fixtureId}`

Same shape. Treat this as destructive: un-booking withdraws a live feed a trading desk may be
pricing against right now.

## Failure handling

- **There is no idempotency key on this API.** If a `Book` call times out you cannot tell whether it
  applied. Do not blind-retry. Re-read the fixture with `BookingV2_Fixtures` and check `IsBooked`
  before issuing the write again — and remember that the re-read is itself subject to the 10-minute
  per-sport cooldown, so the safe pattern is to record intent locally and reconcile on the next
  permitted read.
- **The contract declares no error responses.** Booking V1 and V2 both document only a `200` with
  `type: object`. Treat any non-2xx as opaque, log the raw body, and surface it to a human.
- The `Book` and `UnBook` operations are naturally idempotent *by state* — booking an already-booked
  fixture is a no-op — which is what makes reconcile-then-write safe here even though replay
  protection does not exist.

## Alternative: Auto-Booking

For bulk coverage, the InPlay Manager UI offers Auto-Booking templates that book every fixture
matching a competition/country rule, optionally gated on Match State or Trading State coverage
being available. It is an alternative to calling `BookingV2_Book` per fixture, not an API. Note that
disabling a template **automatically un-books every fixture it booked**.

## References

- Booking API docs: https://geniussports.atlassian.net/wiki/spaces/BID/pages/36084721/Booking+API
- Swagger UI: https://dataservices.betgenius.com/BookingSystem/swagger/ui/index
- Conventions: `conventions/betgenius-conventions.yml`
- Rate limits: `rate-limits/betgenius-rate-limits.yml`
