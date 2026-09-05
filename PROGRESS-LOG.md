# Progress log

Shared status between both devs. Read the top before starting a session, append at the end. Newest first.

---

## Status

**Stage:** Foundations
**Backend:** scaffolded, no running code yet
**Frontend:** in progress on mock data
**Deployed:** no

**Next — backend:** Express + Drizzle + Postgres up, Hedera client, Mirror Node helper, create the three HCS topics.
**Next — frontend:** teammate to fill in.

## Blockers

| Who | Blocked on | Needs |
|---|---|---|
| — | — | — |

---

## Decisions

| Decision | Choice | Why |
|---|---|---|
| Hedera SDK | `@hiero-ledger/sdk` | `@x402/hedera` depends on it directly. Two SDK copies would break `instanceof`. |
| Privy SDK | `@privy-io/node` | Has both `secp256k1_sign` and `verifyAccessToken`. `server-auth` is deprecated and stale. |
| Auth | Privy access token, no sessions | Bearer token, verified locally. No cookies, no CSRF. |
| Settlement asset | USDC `0.0.429274` (6dp) | Matches the USD pricing in the frontend types. HBAR is the fallback. |
| GameKey treasury | Platform operator, no wipe/freeze/pause keys | Studio treasury would mean a sleeping dev blocks their own sales. Missing keys = we can't claw back a purchase, checkable on HashScan. |
| GameKey mint | On demand, async after payment settles | Keeps chain latency off the instant-play moment. |
| Agent watcher | Mirror Node polling, 5s, saved cursor | Restart-safe and easy to demo. |
| Delisted games | Owners keep access | Delisting hides from catalog only. |
| Shared types package | None | Three repos, not a monorepo. Not worth the packaging overhead. |

---

## For the frontend

Endpoints:

```
GET   /api/games                 catalog: search, tag, sort, freeOnly, cursor
GET   /api/games/:idOrSlug
POST  /api/studios
POST  /api/studios/:id/members   invite by email
POST  /api/games                 upload (multipart)
POST  /api/games/:id/publish     locks splits, writes the HCS listing
GET   /api/games/:id/download    x402-gated
GET   /api/games/:id/owned
GET   /api/games/:id/reviews
POST  /api/games/:id/reviews     ownership-gated
PATCH /api/reviews/:id
POST  /api/agents                returns a wallet address to fund
GET   /api/agents/:id
POST  /api/reports
```

Four things that affect client code:

- Auth is `Authorization: Bearer <privy access token>`. No cookie, so no CSRF handling.
- Catalog endpoints work signed out. They add an `owned` flag when a token is present. Don't gate browsing behind login.
- Prices come as `priceUsd` for display and `priceUnits` (integer, 6dp) for math. Don't do money math on the float.
- Never hardcode `payTo`, `feePayer`, or `asset`. All three come back in the 402 response.

`Game`, `Studio`, `Review` and `SplitMember` match your `src/mocks/types.ts` — the backend model was extended to fit those rather than the other way round. `coverSeed` stays alongside a real `coverCid` so the placeholder art keeps working.

Errors are always `{ error: { code, message, details? } }`. Codes worth handling: `UNAUTHENTICATED`, `WALLET_NOT_FUNDED`, `NOT_OWNER`, `GAME_NOT_PUBLISHED`, `VALIDATION_FAILED`, `RATE_LIMITED`.

---

## Validation

Every real external interaction goes here. It's evidence for the 15% validation score, not bookkeeping.

| Date | What | Who | Outcome |
|---|---|---|---|
| | | | |

---

## Log

### 2026-09-05

Split into three repos. Settled the open stack decisions (see table above) by checking the actual packages rather than the docs — the SDK one turned out to be forced by an x402 dependency, not a preference. Wrote the API contract and setup notes. Added a `.gitignore` to CGS-server