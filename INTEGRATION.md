# Integration guide — frontend ↔ backend

For Suparno. This is the doc for replacing `src/mocks/` with the real API.

**Short answer on ownership:** the integration is **frontend work**. The backend
exposes the API and hands you one helper for the payment step; wiring the screens
to it is yours, the same way Privy in the browser is yours. Everything below is
what you need so nothing has to be guessed.

---

## 1. Who does what

| Piece | Whose | Why |
|---|---|---|
| Calling the API, replacing mocks | **Frontend** | It's client code in your repo. The seams are already marked `TODO(integration)`. |
| `@privy-io/react-auth` — login, embedded wallet, funding UI | **Frontend** | Privy's browser SDK only runs in the browser. The onramp/funding UI is a sponsor requirement and it's a UI surface. |
| Verifying Privy tokens, agent wallets, signing | **Backend** | Done. `@privy-io/node` server-side, already built. |
| The x402 payment step | **Backend helper, frontend calls it** | See §4. You do not build Hedera transactions in the browser. |

Nothing about the chain leaks into your code. You send a bearer token and read JSON.

---

## 2. Auth — three steps, no sessions

There is no session cookie, no CSRF token, no login endpoint on our side.

```ts
const token = await getAccessToken();          // from usePrivy()
const res = await fetch(`${API}/api/games`, {
  headers: token ? { Authorization: `Bearer ${token}` } : {},
});
```

That's the whole integration. The backend verifies the token locally on every
request, and creates the user row itself the first time it sees a new one — you
never call a "register" endpoint.

**Browsing endpoints work without a token.** Catalog, game detail, studio pages,
reviews and the invite screen all return real data signed out. Send the token
when you have one and you additionally get an `owned` flag on game detail. Do
not gate browsing behind login — that's a product rule, not a preference.

Privy config that matters on your side: `embeddedWallets.createOnLogin` so a
wallet exists immediately after login. The backend reads the user's embedded
Ethereum wallet address out of their Privy account; if a user somehow has no
embedded wallet, every authenticated request fails, so don't make that optional.

---

## 3. Endpoints

```
GET    /api/games                       catalog: search, tag, sort, freeOnly, cursor, limit
GET    /api/games/:idOrSlug             detail + studio + splits + media (+ owned, liked when signed in)
POST   /api/games                       upload, multipart — see §5
POST   /api/games/:id/publish           locks splits, mints the token, writes the HCS listing
GET    /api/games/:id/download          x402-gated — see §4
POST   /api/games/:id/pay               signs + settles payment server-side — see §4
GET    /api/games/:id/owned             authoritative ownership check
GET    /api/games/:id/reviews
POST   /api/games/:id/reviews           ownership-gated
PATCH  /api/reviews/:id
POST   /api/games/:id/like              toggle — see §6
GET    /api/games/:id/comments
POST   /api/games/:id/comments          no ownership gate — see §6
PATCH  /api/comments/:id
POST   /api/games/:id/sessions          call when play actually starts — see §6
PATCH  /api/games/:id/sessions/:id      call when play ends
POST   /api/studios
GET    /api/studios/ens-availability    ?name=
GET    /api/studios/:idOrSlug
POST   /api/studios/:id/members         invite by email
GET    /api/invites/:id                 public — the emailed link lands here
POST   /api/invites/:id/accept
POST   /api/agents                      returns a wallet address to fund
GET    /api/agents/:id                  status, balance, trigger
GET    /api/me                          who you are, wallet balance, your studio — see §6
GET    /api/me/library                  every game you actually hold a key for — see §6
GET    /api/notifications
POST   /api/notifications/:id/read
POST   /api/reports
GET    /health
```

Errors are always the same shape:

```json
{ "error": { "code": "NOT_OWNER", "message": "…", "details": {} } }
```

Codes worth branching on: `UNAUTHENTICATED` (401, show sign-in),
`WALLET_NOT_FUNDED` (409, show the funding step), `NOT_OWNER` (403),
`GAME_NOT_PUBLISHED` (409), `MODERATION_BLOCKED` (422, upload rejected),
`VALIDATION_FAILED` (422, `details` has the field errors), `RATE_LIMITED` (429).

---

## 4. The payment step — the only non-REST call

`GET /api/games/:id/download` is the one endpoint that doesn't behave like
ordinary REST. It has three outcomes and you only handle two of them:

**It returns `200` immediately** when the game is free, or when this wallet
already owns it. Body:

```json
{ "playUrl": "https://ipfs.io/ipfs/<cid>/index.html",
  "tokenId": "0.0.998877", "keyStatus": "free" | "owned" }
```

Point the player iframe at `playUrl` and you're done. Note this is already a
different origin from the app, so `allow-same-origin` on that iframe is safe by
construction — you don't need your `VITE_PREVIEW_ORIGIN` trick for purchased
builds, only for the pre-publish local preview.

**It returns `402`** when payment is required, with the payment terms:

```json
{ "x402Version": 2,
  "resource": { "url": "…", "description": "…", "mimeType": "application/json" },
  "accepts": [{ "scheme": "exact", "network": "hedera:testnet",
                "amount": "4500000", "asset": "0.0.429274",
                "payTo": "0.0.10375438", "maxTimeoutSeconds": 180,
                "extra": { "feePayer": "0.0.7162784" } }] }
```

You do **not** build a Hedera transaction from this. Call the helper instead:

```
POST /api/games/:id/pay      requireAuth, no body
```

Signs the payment with the logged-in buyer's own Privy wallet server-side (the
browser can't hold a signing key) and returns the same
`{ playUrl, tokenId, keyStatus }` shape as the `200` case above. Use it whenever
`GET /:id/download` came back `402` and the buyer confirms they want to pay.

**`keyStatus: "pending"`** on a successful purchase is deliberate and it changes
your UI. Payment has settled and the buyer is entitled to the game *right now* —
`playUrl` is live, boot it immediately. The GameKey NFT mints in the background
a few seconds later. So: start the game, and let a small "GameKey minting…"
indicator resolve on its own. Poll `GET /api/games/:id/owned` if you want to
show it landing. **Do not block the player on the key.** Blocking there would
put about six seconds of chain round-trips in front of the single moment the
whole demo rests on.

Never hardcode `payTo`, `feePayer`, `asset`, or `amount`. All four come from
the 402 response, and the facilitator's fee-payer account is theirs to change,
not ours.

---

## 5. Upload

`POST /api/games` is `multipart/form-data`:

| Field | Type | Notes |
|---|---|---|
| `build` | file | the zip. Must contain `index.html` at its root — a single wrapper folder is stripped automatically, matching your local preview |
| `media` | file[] | up to 8 images/videos |
| `coverMediaIndex` | number | which `media` entry is the cover. Omit it and the generated cover art is used |
| `splits` | string | JSON array: `[{ wallet, handle, role, pct }]`, must total exactly 100 |
| plus | | `studioId`, `title`, `tagline`, `description`, `tags`, `priceUnits` |

Upload creates a **draft**. `POST /api/games/:id/publish` is a separate call
that locks the splits, creates the token, and publishes. That split is
deliberate — splits are only editable while a game is a draft.

**Right now every upload fails with `MODERATION_BLOCKED`.** That's not a bug in
your code. A CSAM (child sexual abuse material) hash-check runs before anything
reaches storage, and no scanning provider has been chosen yet, so it fails
closed. Build the screen against the error path; it'll start passing once a
provider is wired.

---

## 6. Identity, library, likes, comments, playtime

New. `session.ts`'s mocked fields (`email`, `balanceUsd`, `studioId`,
`ownedGameIds`) and the "achievements, profiles, likes, comments" line in the
frontend brief's "deliberately not building" list are both superseded by
this — the cut was for time reasons that no longer apply, so it's reversed.

```http
GET /api/me            requireAuth
```
```json
{ "id": "...", "email": "dev@example.com", "evmAddress": "0x71C7…",
  "hederaAccountId": "0.0.512345", "balanceUnits": "4500000",
  "balanceAsset": "0.0.429274",
  "studio": { "id": "...", "name": "Tin Roof", "slug": "tin-roof", "role": "owner" } }
```
This is `session.ts`'s real backing. `balanceUnits` is an integer in
`balanceAsset`'s smallest units — same rule as every other price in this doc,
divide for display, never do the arithmetic on a float. `studio` is `null`
until this user owns one or has an **accepted** invite into one; `role` tells
you which.

```http
GET /api/me/library     requireAuth
```
```json
{ "games": [ { "id": "...", "slug": "...", "title": "...", "tagline": "...",
  "studio": { "id": "...", "name": "...", "ens": "tinroof.eth", "slug": "..." },
  "coverCid": "...", "coverSeed": 8412, "status": "published",
  "serial": 2, "myPlayCount": 3, "myPlaytimeSeconds": 5400 } ] }
```
The real answer to `/library`'s "keys you hold" — checked live against the
Mirror Node, not a local flag, so it's correct even for a key minted outside
this app. `myPlayCount`/`myPlaytimeSeconds` are this user's own sessions on
that game (see below) — the concrete "how much you've played this."

```http
POST /api/games/:id/like              requireAuth   → { liked, likeCount }
```
Toggle — call it again to unlike. No ownership needed, so it's safe to show
on a listing the buyer hasn't purchased yet.

```http
GET  /api/games/:id/comments          → { comments, nextCursor }
POST /api/games/:id/comments          requireAuth   { body }
PATCH /api/comments/:id               requireAuth (author)
```
Ordinary discussion, no ownership gate and no rating — the deliberate
difference from a review. Same shape and pagination as
`GET /api/games/:id/reviews`, so one component can likely render both.

```http
POST  /api/games/:id/sessions              requireAuth   → { sessionId, startedAt }
PATCH /api/games/:id/sessions/:sessionId   requireAuth   { } → session
```
Call the first the moment the player actually boots — right after
`/download` or `/pay` hands back a `playUrl`, not before; it re-checks
ownership server-side, so it'll reject a game this wallet doesn't hold. Call
the second when play ends: component unmount, or a `beforeunload` handler if
you can reach one before the tab actually closes. It's fine if that call
sometimes never fires (a crashed tab, a hard-killed browser) — the session
still counts toward `plays` on the listing, it just contributes no duration
to `myPlaytimeSeconds`. This is the piece the player surface (`GameStage`)
needs wired; nothing else in this doc depends on it.

`Game` gains `liked` (only when signed in, next to `owned`) and real
`plays`/`likeCount` — both existed in `types.ts` and the contract already,
neither was ever actually computed before this. `mocks/types.ts` will need a
`Comment` type (mirror `Review` minus `rating`) and a `PlaySession`-shaped
concept for whatever calls the two session endpoints.

---

## 7. Money

Every price arrives twice:

- `priceUsd` — a float, **display only**
- `priceUnits` — an integer in the asset's smallest units, **all arithmetic**

`priceAsset` names the asset (`0.0.429274` is testnet USDC, 6 decimals;
`0.0.0` is HBAR, 8 decimals). Never do money math on `priceUsd`.

---

## 8. Suggested order

1. **Catalog + detail** — no auth, no chain, immediate payoff. Proves the fetch
   layer and the shapes.
2. **Privy login** — `usePrivy()`, `getAccessToken()`, send the header. Now
   `owned`/`liked` start appearing, and `GET /api/me` replaces the mocked
   session fields.
3. **Library + notifications** — `GET /api/me/library` plus the existing
   notification routes, both plain authenticated GETs.
4. **Publish** — the biggest form. Expect `MODERATION_BLOCKED` until the CSAM
   provider lands; everything up to that point is real.
5. **Checkout** — needs the helper and a funded wallet.
6. **Likes, comments, play sessions** — no particular order, none of the
   other five steps depend on them; the session calls are the only ones tied
   to a specific moment (see §6).

Steps 1–4 need nothing from the backend that isn't already live and tested.
Step 5 is the one to sync on before starting.

---

## 9. Local setup

Backend runs on `:3000`. Set `CORS_ORIGIN` in the backend `.env` to your Vite
origin (defaults to `http://localhost:5173`) — if requests fail with a CORS
error, that's the variable, tell me rather than working around it.

The database is shared Neon, so whatever you publish locally is visible to
both of us. Handy for testing against real rows; worth knowing before you
wonder where someone else's test game came from.
