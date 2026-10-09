# How it works

```
Muse ──HTTPS POST /api/mcp──▶ auth (Bearer key) ──▶ MCP server ──▶ inventory backend ──▶ Supabase / CourtReserve
                                  │                                     │
                     production key → your venue              connector store: quotes (HMAC),
                     review key     → isolated review venue   idempotency keys, waitlist
```

## A booking, step by step

1. **Search.** Muse calls `check_availability` with the player's date and time window. Only open slots come back.
2. **Quote.** Muse calls `get_quote` for the chosen slot. The server stores the price with an HMAC signature and a 5-minute expiry.
3. **Confirm.** Muse shows the player the court, time and price. `create_booking` is a Sensitive write, so Muse asks "Allow once / Deny".
4. **Book.** `create_booking` checks the signature and expiry, then claims the slot and writes the booking in one database transaction. If two players try at once, exactly one wins. Retrying with the same idempotency key returns the same booking instead of a second one.
5. **Change or cancel.** `modify_booking` and `cancel_booking` are atomic as well: a move never leaves two slots booked, or none.

## Waitlist

1. If nothing matches, `join_waitlist` records the window and, with the player's consent, an email or phone number.
2. When a matching slot frees up (cancel, move, or an expired hold), it is held for the oldest matching entry, with a 15-minute quote.
3. The player is told by email, SMS and the next time they talk to Muse. They say "book it", and the normal `create_booking` confirm step runs.
4. Only that player can book the held slot. If the hold expires, it passes to the next player. Expired holds are released on the next tool call, and by a daily cron as a backstop.

## Review isolation

Meta's review needs a dedicated test account. Court Connector gives reviewers a second API key that resolves to a copy of your venue named "<venue> (Muse review)". Every tool is scoped by venue, so a review key can't see or change real bookings, and `npm run review:reset` returns the review venue to a clean schedule between review rounds.

## Backends

All booking logic runs against an `InventoryBackend` interface. Two implementations:

- **Supabase** (default): your courts, slots and bookings live in your own Supabase project, configured from `venue-config.yaml`.
- **CourtReserve** (preview): books against a club's existing CourtReserve schedule through its Integrations API. It is built from CourtReserve's public API docs and is not yet tested against a live club.

A shared contract test suite checks that every backend behaves the same way.
