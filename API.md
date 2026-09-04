# API

Route Handlers under `/api`. The web app mostly uses Server Actions; these
endpoints exist because the React Native app will need them, so build them as the
real interface and have the web call the same logic.

All money in **kobo**. All timestamps **ISO 8601 UTC**.

## Conventions

```
Authorization: Bearer <jwt>          # except public endpoints
Idempotency-Key: <uuid>              # required on POST /orders and payments
```

Errors are consistent and user-readable:

```json
{ "error": { "code": "ITEM_SOLD_OUT",
             "message": "Terkimbi's has run out of egusi.",
             "details": { "menuItemId": "..." } } }
```

Never return a raw stack trace. The `message` is shown to the user as-is, so
write it in product voice.

---

## Auth

```
POST   /api/auth/request-code      { phone, channel? }  → { retryAfter, attemptsLeft }
POST   /api/auth/verify-code       { phone, code }      → { token, user, roles[], isNewUser }
POST   /api/auth/request-voice     { phone }            → { retryAfter }
POST   /api/users/me               { name }             → { user }
GET    /api/users/me                                    → { user, roles[] }
```

**Rules**
- Max 3 code requests per phone per hour, enforced server-side
- Codes expire after 10 minutes, 5 wrong attempts locks for 15 minutes
- `roles[]` drives the "Continue as" screen. One role → skip it entirely.
- Never return whether a phone already exists. Same response either way.

## Geography

```
GET    /api/areas                              → [{ id, name, slug, isLive }]
GET    /api/areas/resolve?lat=&lng=            → { area | null, covered }
POST   /api/areas/requests    { rawText, lat?, lng? }  → { ok }
GET    /api/delivery/quote?restaurantId=&addressId=    → { distanceKm, feeKobo, etaMinutes }
```

`resolve` does a PostGIS point-in-polygon against `areas.boundary`. If no match,
`covered: false` and the app shows the "we don't reach Gboko yet" screen.

## Addresses

```
GET    /api/addresses                          → [address]
POST   /api/addresses  { landmark, riderNote?, lat?, lng?, label? } → { address }
PATCH  /api/addresses/:id
DELETE /api/addresses/:id
```

`landmark` is required free text. `lat`/`lng` optional — many users deny location.
Resolve `area_id` server-side from coordinates when present, otherwise by fuzzy
matching the landmark text against known areas, and fall back to asking.

## Discovery

```
GET /api/restaurants?areaId=&lat=&lng=&q=      → [restaurantCard]
GET /api/restaurants/:slug                     → { restaurant, sections[], items[] }
GET /api/search?q=&areaId=                     → { dishes[], restaurants[] }
```

`restaurantCard` includes `distanceKm`, `deliveryFeeKobo`, `etaMinutes`,
`isOpen`, `opensAt`, `ratingAvg`. **Sorted by distance ascending, not by
"featured".** Closed restaurants are returned but flagged, never hidden.

A restaurant is open when: `is_approved` AND `is_open` AND now is inside today's
`restaurant_hours` AND (`paused_until` IS NULL OR now > `paused_until`).

## Cart

```
GET    /api/cart                                        → { cart, issues[] }
POST   /api/cart/items  { menuItemId, quantity, optionIds[], note? }
                                                        → { cart } | 409 DIFFERENT_RESTAURANT
PATCH  /api/cart/items/:id  { quantity }
DELETE /api/cart/items/:id
POST   /api/cart/clear
POST   /api/cart/validate                               → { cart, issues[] }
```

**`issues[]` is the important part.** Called on cart open and again at checkout:

```json
[{ "type": "SOLD_OUT",         "menuItemId": "...", "name": "Pounded yam & egusi" },
 { "type": "PRICE_CHANGED",    "menuItemId": "...", "oldKobo": 280000, "newKobo": 300000 },
 { "type": "RESTAURANT_CLOSED","opensAt": "2026-09-05T09:00:00Z" },
 { "type": "BELOW_MINIMUM",    "shortfallKobo": 120000 }]
```

Adding an item from a different restaurant returns `409 DIFFERENT_RESTAURANT`
with the current restaurant's name, so the client can show the "start a new cart?"
sheet. **Never silently clear a cart.**

## Orders

```
POST   /api/orders  { addressId | address, paymentMethod, promoCode?, note? }
                                             → { order, paymentUrl? }
GET    /api/orders                           → [orderSummary]
GET    /api/orders/:id                       → { order, items[], events[] }
POST   /api/orders/:id/cancel                → { order }
GET    /api/track/:publicToken               → { order, riderLocation? }   # PUBLIC
```

`/api/track/:publicToken` requires **no auth** — that's the shareable link. Return
only what a stranger should see: status, ETA, item names, total, rider first name
and position. Never the customer's phone or full address.

**Order creation must, in one transaction:**
1. Re-validate every item's availability and price
2. Re-check the restaurant is open
3. Re-check the minimum order
4. Recompute the delivery fee from live distance
5. Snapshot address, phone, item names and prices onto the order
6. Create the payment intent
7. Set status `awaiting_restaurant` and start the 3-minute timer

If any step fails, nothing is written and the user gets a specific message.

## Payments

```
POST   /api/payments/initialize  { orderId }   → { authorizationUrl, reference }
POST   /api/payments/webhook                   # Paystack → us
GET    /api/payments/verify/:reference         → { status }
```

Verify the Paystack signature on the webhook. **Trust the webhook, not the client
redirect** — users close browsers mid-payment. Cash orders skip this and go
straight to `awaiting_restaurant`.

## Restaurant

All require `restaurant_staff` for that specific `restaurantId`, checked
server-side against `user_roles`.

```
GET    /api/r/:id/orders?status=              → [order]
POST   /api/r/:id/orders/:orderId/accept      { prepMinutes }  → { order }
POST   /api/r/:id/orders/:orderId/reject      { reason }       → { order }
POST   /api/r/:id/orders/:orderId/ready
GET    /api/r/:id/menu
PATCH  /api/r/:id/menu/items/:itemId          { soldOut? , priceKobo?, isActive? }
POST   /api/r/:id/open                        { isOpen }
POST   /api/r/:id/pause                       { minutes }      # 15 | 30 | 60
GET    /api/r/:id/payouts
```

**`reject` reasons** are an enum: `sold_out`, `too_busy`, `closing_early`,
`other`. When `sold_out`, the client also sends `menuItemIds[]` and the server
marks them sold out immediately — that's what closes the loop with customer carts.

**Sold-out reset:** a scheduled job runs hourly and clears
`sold_out_until_open` for any restaurant whose opening time has passed since the
flag was set. Do not rely on the restaurant remembering.

## Realtime

Use Supabase Realtime or SSE. Minimum viable set:

| Channel | Who listens | Events |
|---|---|---|
| `order:{id}` | customer, tracking link | status changes, ETA changes, rider position |
| `restaurant:{id}` | restaurant staff | new order, cancellation |

**The restaurant notification problem is not solved by realtime.** A browser tab
cannot reliably ring a locked Android phone. See the open question in CLAUDE.md.

## Rate limits

| Endpoint | Limit |
|---|---|
| `auth/request-code` | 3 / phone / hour |
| `auth/verify-code` | 5 / phone / 15 min |
| `orders` POST | 10 / user / hour |
| `areas/requests` | 5 / user / day |
| everything else | 100 / user / min |
