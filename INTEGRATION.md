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

Privy config that matters on your side: `embeddedWallets.ethereum.createOnLogin`
so a wallet exists immediately after login. The backend reads the user's
embedded Ethereum wallet address out of their Privy account; if a user has no
embedded wallet, every authenticated request fails, so don't make that optional.

**Retry the first `/api/me` after a brand-new sign-in.** Privy creates the
wallet as part of logging in and for a moment afterwards its own API still
reports the account without one, so the server correctly answers "no embedded
wallet" and a cached failure strands the session permanently. `SessionProvider`
now retries four times over about four seconds.

---

## 3. Endpoints

```
GET    /api/games                       catalog: search, tag, sort, freeOnly, cursor, limit
GET    /api/games/:idOrSlug             detail + studio + splits + media (+ owned, liked when signed in)
POST   /api/games                       upload, multipart — see §5
POST   /api/games/:id/publish           locks splits, mints the token, writes the HCS listing
GET    /api/games/:id/download          x402-gated — see §4
GET    /api/games/:id/build.zip         the build itself, ownership-checked
POST   /api/games/:id/pay/prepare       server builds and freezes the transfer — see §4
POST   /api/games/:id/pay/complete      browser's signatures go back, server settles
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
GET    /api/me                          who you are, wallet balances, your studio — see §6
GET    /api/me/library                  every game you actually hold a key for — see §6
GET    /api/me/earnings                 what you've earned, across every studio — see §6.2
GET    /api/studios/:id/earnings        what the studio made, team only — see §6.2
POST   /api/me/withdraw/prepare         build a transfer out of the wallet — see §6.1
POST   /api/me/withdraw/complete        sign it in the browser, server submits
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
{ "buildPath": "/api/games/<id>/build.zip", "buildCid": "bafy…",
  "tokenId": "0.0.998877", "keyStatus": "free" | "owned" }
```

**There is no `playUrl`, and IPFS is not where you fetch the build from.**
Pinata refuses to serve HTML through its public gateway (`403 ERR_ID:00023`),
and public gateways time out on freshly pinned content. So `buildPath` is an
ownership-checked URL on this API that returns the zip, and the client unpacks
it onto its own isolated build origin — the same pipeline the publish preview
already uses. `buildCid` is still there because IPFS is what makes a build
verifiable by someone who doesn't trust us; it just isn't the delivery route.

Because the build now comes from this origin, the second origin **is** needed
for purchased builds, not only the local preview. That reverses what this
section used to say.

**It returns `402`** when payment is required, with the payment terms:

```json
{ "x402Version": 2,
  "resource": { "url": "…", "description": "…", "mimeType": "application/json" },
  "accepts": [{ "scheme": "exact", "network": "hedera:testnet",
                "amount": "4500000", "asset": "0.0.429274",
                "payTo": "0.0.10375438", "maxTimeoutSeconds": 180,
                "extra": { "feePayer": "0.0.7162784" } }] }
```

You do **not** build a Hedera transaction from this. It takes two calls:

```
POST /api/games/:id/pay/prepare    requireAuth, no body
  -> { status: "prepared", intentId, hashes, expiresAt, amountUnits, asset }
  -> or { status: "granted", …grant }  when it's free or already owned

POST /api/games/:id/pay/complete   requireAuth  { intentId, signatures }
```

**The browser signs, not the server.** The earlier version of this doc said the
browser can't do raw-hash signing. It can: `secp256k1_sign` is supported on
Privy's embedded wallet provider and signs a hash with no Ethereum prefix,
which is exactly Hedera's format. Signing server-side would have required every
buyer to delegate their wallet to the store first, which is standing permission
to move their money and a much larger thing to ask than one game. So the server
builds and freezes (it needs the 402 terms and a Hedera client), the browser
signs, the server settles.

`hashes` is a list because a frozen Hedera transaction carries one body per
node it may go to, each needing its own signature. Send them back as
`[{ hash, signature }]` — they're matched by hash, not position, so order can't
corrupt a payment.

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


### `GET /api/me` gains two balance fields

```json
{ "balanceUnits": "10000000", "balanceUsd": 10.00, "balanceAssetDecimals": 6,
  "hbarUnits": "100000000", "hbar": 1.0 }
```

`hbarUnits` is tinybars, `hbar` is the same thing for display. It is reported
separately from the settlement asset because **HBAR is not spending money
here**: the x402 facilitator covers the fee on a purchase and the operator
covers it on a withdrawal, so HBAR is only ever what opened the account. A
wallet holding 0 USDC and some HBAR is funded with nothing to spend, and
without this field that reads identically to a wallet with nothing at all.

### 6.1 Taking money out

Same two-step shape as a purchase, and for the same reason: the server builds
and freezes the transfer because that needs a Hedera client, and the browser
signs because the key is the person's. Reuse `useWalletSigner`.

```http
POST /api/me/withdraw/prepare    requireAuth   { to, asset?, amountUnits?, memo? }
```
`to` is **either** a Hedera account id (`0.0.x`) **or** an EVM address —
someone copying an address out of their own wallet has no reason to know which
one we wanted. `asset` defaults to the settlement asset; pass `"0.0.0"` for
HBAR. Omit `amountUnits` to send the whole balance, which is what "take my
money out" usually means.

```json
{ "intentId": "…", "hashes": ["0x…"], "to": "0.0.512345",
  "asset": "0.0.429274", "amountUnits": "10000000", "amountDisplay": 10.0,
  "assetDecimals": 6, "expiresAt": "…" }
```

```http
POST /api/me/withdraw/complete   requireAuth   { intentId, signatures }
```
`signatures` is `[{ hash, signature }]`, exactly as `/pay/complete` takes them.
Returns `{ status: "sent", transactionId, to, asset, amountUnits }`.

**The user does not need HBAR to withdraw.** The operator pays the network fee,
because otherwise a wallet holding only USDC would be a wallet you cannot
empty. Verified on testnet: the full balance leaves and the sender's HBAR is
untouched.

**`memo` matters for any off-ramp.** Exchange deposit addresses are pooled
accounts that identify the depositor by memo, the same mechanic as an XRP tag.
Sending to one without it means the money is credited to nobody. If a withdraw
UI ever points at an exchange, it has to offer this field.

Two failures worth handling by name, both arriving as `VALIDATION_FAILED` with
the field in `details`: `to` when the destination has no Hedera account yet, or
when it cannot receive the token (it needs associating in that wallet first);
and `intentId` when the intent expired, which is a "start it again", not an
error to show as a failure.

### 6.2 Earnings

Two views of the same numbers, computed by one function so they cannot
disagree.

```http
GET /api/me/earnings          requireAuth
```
**Cross-studio, deliberately.** Someone added to a split by email is often
credited on games from several teams, so framing this as "your studio" would
hide money from exactly the people the splits feature exists for.

```json
{ "totals": { "earned": {"units":12000,"display":0.012,"assetDecimals":6},
              "gross": {...}, "sales": 2, "games": 1,
              "held": {...}, "failed": {...}, "asset": "0.0.429274" },
  "games": [ { "gameId":"…", "slug":"…", "title":"…", "status":"published",
               "studio": {...}, "sales": 2, "gross": {...},
               "yours": { "pct": 60, "role": "code", "earned": {...} },
               "plays": 8, "likes": 3, "reviews": 1, "rating": 5 } ],
  "held": [ { "gameTitle":"…", "amount": {...}, "reason":"…", "since":"…" } ],
  "failed": [] }
```

```http
GET /api/studios/:id/earnings    requireAuth, owner or accepted member
```
Same shape plus `people` — every handle on the studio's splits, what they
earned, and whether they've claimed their invite yet. That's what lets an owner
see *"your artist hasn't claimed theirs, 12.50 is waiting"*. Anyone outside the
studio gets `NOT_OWNER`.

Every money value is `{ units, display, assetDecimals }`. `units` is the truth;
`display` is there so you don't derive it, and nothing should compute with it.

**Held money settles by itself now.** A share that couldn't be paid (the person
has no Hedera account yet) is held, and it goes out the moment that account
first appears — triggered on `GET /api/me`, so simply opening the site is what
releases it. You don't need a "claim" button and shouldn't build one.

### Funding a new wallet: no HBAR step

A Hedera account doesn't exist until value first lands on the address, but
**a token transfer creates it too** — HIP-542 charges the creation fee to the
sender rather than deducting it from what's sent. Verified on testnet by
sending only USDC to an untouched address: the account was created holding the
USDC and **zero HBAR**.

So the funding hint is simply "send USDC here". Nobody has to acquire HBAR
first, and the profile menu says so.

One caveat worth knowing before pointing anyone at a funding route: **not every
wallet can send to an EVM address.** HashPack can, and it works. Circle's
testnet faucet cannot — it requires a `0.0.x`, so it's only usable once the
account already exists. Exchanges generally reject EVM addresses outright.
After the first transfer the account has a `0.0.x` and all of that stops
mattering.

### `/api/me` also gains `studios`

An array of every studio the person owns or has accepted an invite to, each
with `role`. `studio` stays as the primary one so nothing existing moves. Worth
using wherever a picker makes sense: a person on two teams could only ever see
one before.

### Studio members see their team's drafts

`GET /api/studios/:idOrSlug` returns unpublished games to the owner **and to
accepted members**, and only published ones to everyone else. Ownership was the
wrong line — a collaborator credited on a game couldn't see the game they
helped make. Member email addresses are still owner-only.

### Invites now actually send

`POST /api/studios/:id/members`, and naming someone by `email` on a split in
`POST /api/games`, both create the membership row that *is* the invite. They
now also email that person. Nothing about the request or response changed —
this is the half that was missing, since an invitee has no account and so no
notification row could ever reach them.

Mail is best-effort by design: a send never fails the request that caused it.
**With no verified domain, Resend only delivers to the account's own address**,
so an invite to anyone else is refused and logged. That is configuration, not a
bug, and it changes nothing on the client.

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
