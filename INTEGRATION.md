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
PATCH  /api/games/:id                   edit the listing, including price — see §10
POST   /api/games/:id/builds            ship a new version, multipart — see §10
GET    /api/games/:id/builds            public version history
GET    /api/games/:id/price-history     public, every row checkable on HCS
POST   /api/games/:id/unpublish         take it off the catalog; buyers keep it
POST   /api/games/:id/relist            put it back
DELETE /api/games/:id                   drafts only
POST   /api/games/:id/promotions        put a game on sale — see §17
GET    /api/games/:id/promotions        public: the running sale + history
PATCH  /api/games/:id/promotions/:pid   extend the end date
DELETE /api/games/:id/promotions/:pid   end it early
POST   /api/games/:id/media             add screenshots
PATCH  /api/games/:id/media             reorder
DELETE /api/games/:id/media/:mediaId
GET    /api/games/:id/manage            everything the studio's own screen needs
GET    /api/games/:id/download          x402-gated — see §4
GET    /api/games/:id/build.zip         the build itself, ownership-checked
POST   /api/games/:id/pay/prepare       server builds and freezes the transfer — see §4
POST   /api/games/:id/pay/complete      browser's signatures go back, server settles
GET    /api/games/:id/owned             authoritative ownership check
GET    /api/games/:id/reviews
POST   /api/games/:id/reviews           ownership-gated
PATCH  /api/reviews/:id
DELETE /api/reviews/:id                 the reviewer only — see §15
POST   /api/reviews/:id/reply           the developer's reply — see §15
DELETE /api/reviews/:id/reply
POST   /api/games/:id/like              toggle — see §12
POST   /api/games/:id/wishlist          add (idempotent) — see §12
DELETE /api/games/:id/wishlist          remove
GET    /api/games/:id/demand            public wishlist count — see §12
GET    /api/me/wishlist                 your list, with what changed — see §12
GET    /api/games/:id/comments
POST   /api/games/:id/comments          no ownership gate — see §6
PATCH  /api/comments/:id
DELETE /api/comments/:id                the author, or the game's manager — see §15
GET    /api/games/:id/saves             cloud saves — see §13
GET    /api/games/:id/saves/:slot
PUT    /api/games/:id/saves/:slot
DELETE /api/games/:id/saves/:slot
POST   /api/games/:id/sessions          call when play actually starts — see §6
PATCH  /api/games/:id/sessions/:id      call when play ends
POST   /api/studios
GET    /api/studios/ens-availability    ?name=
GET    /api/studios/:idOrSlug
POST   /api/studios/:id/members         invite by email
DELETE /api/studios/:id/members/:mid    remove — see §14
PATCH  /api/studios/:id/members/:mid    { role } — promote/demote a manager
POST   /api/studios/:id/members/:mid/resend-invite
POST   /api/studios/:id/leave           leave a studio you're on
POST   /api/studios/:id/transfer        { toMemberId } — founder only
GET    /api/invites/:id                 public — the emailed link lands here
POST   /api/invites/:id/accept
POST   /api/me/agent                    create — see §18
GET    /api/me/agent                    status, live balance
PATCH  /api/me/agent                    change mode / timeout / expiry
DELETE /api/me/agent                    retire, refund
GET    /api/me/agent/decisions          audit trail, newest first
POST   /api/me/agent/decisions/:id/respond   answer an ask-first question — see §18
PATCH  /api/games/:id/wishlist          set or clear a want — see §18
GET    /api/users/:handle               public profile — see §11
GET    /api/users/handle-availability   ?handle=
PATCH  /api/me/profile                  display name, handle, bio, library visibility
POST   /api/me/avatar                   multipart, field `avatar`
DELETE /api/me/avatar
GET    /api/me/purchases                receipts, with the settlement tx id
GET    /api/me                          who you are, wallet balances, your studio — see §6
GET    /api/me/library                  every game you actually hold a key for — see §6
GET    /api/me/earnings                 what you've earned, across every studio — see §6.2
GET    /api/studios/:id/earnings        what the studio made, team only — see §6.2
POST   /api/me/withdraw/prepare         build a transfer out of the wallet — see §6.1
POST   /api/me/withdraw/complete        sign it in the browser, server submits
GET    /api/notifications
POST   /api/notifications/:id/read
POST   /api/notifications/read-all      one request, not one per row
POST   /api/reports
POST   /api/reports/content             report a review or comment — see §16
POST   /api/dev/faucet                  dev only, 404s unless DEV_FAUCET=on
GET    /health
```

Errors are always the same shape:

```json
{ "error": { "code": "NOT_OWNER", "message": "…", "details": {} } }
```

Codes worth branching on: `UNAUTHENTICATED` (401, show sign-in),
`WALLET_NOT_FUNDED` (409, show the funding step), `NOT_OWNER` (403),
`GAME_NOT_PUBLISHED` (409), `MODERATION_BLOCKED` (422, upload rejected),
`VALIDATION_FAILED` (422, `details` has the field errors), `RATE_LIMITED` (429),
`MODERATION_HOLD` (409, a relist that moderation won't allow), `GAME_HAS_SALES`
and `GAME_IS_WATCHED` (409, why a draft delete was refused), `HANDLE_TAKEN`
(409, someone else has that handle), `SAVE_CONFLICT` (409, a cloud save changed
elsewhere), `PAYLOAD_TOO_LARGE` (413) and `MALFORMED_JSON` (400) — those last
two used to come back as `500`. `IS_FOUNDER` (409, trying to remove, demote, or
leave-as the studio's actual founder — see §14), `PROMOTION_EXISTS` and
`PROMOTION_ACTIVE` (409, see §17).

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

---

## 10. Managing a game after it's published

This is new, and it's the largest thing the frontend is currently missing. A
published game used to be frozen: no price change, no typo fix, no new build,
no way for a developer to take their own work down. All of that works now.

Everything here needs the caller to be the game's studio owner, or a member
promoted to the `owner` role. Anyone else gets `403 NOT_OWNER`.

### Editing

```
PATCH /api/games/:id
{ "title": "…", "tagline": "…", "description": "…",
  "tags": ["puzzle"], "priceUnits": 250000, "coverMediaId": "<uuid|null>" }
```

Every field is optional; send only what changed. `priceUnits` is integer
smallest-units like everywhere else — `250000` is $0.25 at 6 decimals.

Two things to surface in the UI:

- **The slug never changes**, even when the title does. Existing links keep
  working. Don't re-route after a rename.
- The response carries **`announced`** when the price changed. `false` means the
  new price is live here but hasn't reached the public HCS topic yet, so
  wishlist agents can't see it. Worth showing — it's the difference between "I
  put it on sale" and "the sale is public."

`coverMediaId` picks an existing image as the cover; it does not upload one.
Uploading is `POST /api/games/:id/media` (multipart, field `media`, up to 8).
`PATCH /api/games/:id/media` takes `{ "mediaIds": [...] }` and reorders —
anything you leave out keeps its relative position at the end, it isn't deleted.

### Shipping a new build

```
POST /api/games/:id/builds       multipart: build=<zip>, label?, notes?
```

A game has versions now. Uploading a new build makes it the one everyone plays,
including people who already bought the game — that's the point of owning a key
rather than a file, and every owner gets a `build_updated` notification. The old
version's CID stays in the history permanently.

`label` is the developer's own name for it ("v19", "1.0.2"), `notes` are patch
notes. Both optional, both shown as-is.

`GET /api/games/:id/builds` is public and takes a slug too:

```json
{ "current": 2,
  "builds": [ { "version": 2, "label": "v19", "notes": "fixed the jump",
                "buildCid": "bafy…", "buildSizeKb": 58141,
                "hcsTxId": "0.0.x@…", "createdAt": "…" } ] }
```

### Price history

`GET /api/games/:id/price-history` — public, slug or id:

```json
{ "currentUnits": 250000, "currentUsd": 0.25,
  "asset": "0.0.429274", "assetDecimals": 6,
  "lowestEverUnits": 250000,
  "history": [
    { "fromUnits": 500000, "toUnits": 250000,
      "fromUsd": 0.5,     "toUsd": 0.25,
      "asset": "0.0.429274", "assetDecimals": 6,
      "at": "2026-09-07T18:22:11.402Z",
      "hcsTxId": "0.0.10375438@1788784716.529183364",
      "topicId": "0.0.10380868" } ] }
```

Newest first. **A row describes a change, not a price** — there is no
`priceUnits`/`priceUsd` on it, only the `from`/`to` pair. Render the movement.

The exact same row shape appears as `priceHistory` inside
`GET /api/games/:id/manage`, so one component can render both.

`hcsTxId` is the point: **every row names the HCS message that announced it**,
so a visitor can verify the whole history on the Mirror Node without trusting
us. No other storefront can offer that, because they all own the database their
price history lives in. `hcsTxId` is `null` only when an announcement failed and
`npm run listings:retry` hasn't caught up yet.

### Unlisting

`POST /api/games/:id/unpublish` takes a game off the catalog. The response
includes `ownersKeepAccess: true` — say it in the confirmation dialog, because
it's the thing a developer is actually worried about. `POST /api/games/:id/relist`
puts it back, and returns `409 MODERATION_HOLD` if the game was delisted by
moderation rather than by its developer.

`DELETE /api/games/:id` works on **drafts only**. A published game returns `422`
with a message pointing at unlisting; one that has sold returns `409
GAME_HAS_SALES`.

### The manage screen

`GET /api/games/:id/manage` is one call for the whole developer view:

```json
{ "game": { … }, "media": [ … ], "builds": [ … ], "priceHistory": [ … ],
  "stats": { "sales": 4, "grossUnits": 12000000, "grossUsd": 12,
             "owners": 5, "plays": 23, "playtimeSeconds": 8140,
             "reviewCount": 2, "rating": 4.5, "unsettledSplits": 0 } }
```

`owners` counts distinct wallets holding a key, so it's higher than `sales` for
a free game. `unsettledSplits` is sales whose money never reached the
collaborators — surface it, because a team otherwise finds out when someone asks
where their money is.

### Also new on every game

`GET /api/games/:idOrSlug` now returns `status`, `buildVersion`, `updatedAt` and
`delistedBy` alongside everything it already returned. Nothing was removed.

---

## 11. Profiles

Everyone has a handle and a page at it. This is what the truncated addresses all
over the site were standing in for.

### Someone else's page

`GET /api/users/:handle` — public, no auth needed, and it never returns an email
address. Case-insensitive.

```json
{ "handle": "kai", "displayName": "Kai", "label": "Kai",
  "bio": "…", "avatarUrl": "https://…", 
  "address": "0x…", "addressShort": "0x67…aCfD", "hederaAccountId": "0.0.x",
  "joinedAt": "…", "isSelf": false, "libraryPublic": true,
  "studios":  [ { "id": "…", "name": "…", "slug": "…", "ens": "…", "role": "owner" } ],
  "credits":  [ { "gameId": "…", "slug": "…", "title": "…", "coverUrl": "…",
                  "studio": { … }, "role": "art", "pct": 30 } ],
  "reviews":  [ { "id": "…", "rating": 5, "body": "…", "game": { … } } ],
  "library":  [ { "gameId": "…", "slug": "…", "title": "…", "playtimeSeconds": 900 } ],
  "stats": { "gamesCredited": 3, "gamesOwned": 12, "reviewCount": 4,
             "wishlistCount": 7, "playCount": 22, "playtimeSeconds": 8140 } }
```

`label` is the one to print: display name, else handle, else a truncated
address. Use it everywhere rather than reimplementing the fallback.

**`credits` is the interesting one.** It is every game this person has a share
of, with the share. Steam names a publisher and itch names an uploader — this
names everyone who made a thing and proves what each of them is paid. Worth a
real section on the page rather than a list of links.

`library` is empty and `stats.gamesOwned` is `null` when the person has set
`libraryPublic: false` (unless it's their own page). Reviews and credits are
public either way.

### Your own page

```
PATCH /api/me/profile
{ "displayName": "Kai", "handle": "kai", "bio": "…", "libraryPublic": true }
```

All optional. **The handle gets normalised** — lowercased, non-URL-safe
characters dropped — so "Kai Saha" becomes `kaisaha`. Show the normalised value
back before saving; `GET /api/users/handle-availability?handle=…` returns
`{ normalised, available, reason }` where reason is `taken`, `reserved`,
`unusable`, or null. A `409 HANDLE_TAKEN` on save means someone got there first.

`POST /api/me/avatar` is multipart, field `avatar`, images only, 5MB cap. It
goes through the same moderation gate as every other uploaded image.
`DELETE /api/me/avatar` clears it.

`GET /api/me` now also returns `handle`, `displayName`, `label`, `bio`,
`avatarUrl` and `libraryPublic`, so the header needs no second request.

**Default handles come from the email's local part.** Consider prompting for a
real one on first visit — `displayName === null` is a reasonable trigger for a
"finish your profile" step.

### Receipts

`GET /api/me/purchases` — newest first, one row per purchase, each with
`settlementTxId` (look it up on the Mirror Node), the price paid at the time,
the game, and the `key` (`tokenId`, `serial`) it minted.

### Authors on reviews and comments

Both lists now carry `authorProfile` alongside the existing `author` string:

```json
{ "author": "kai", "authorIsEns": false,
  "authorProfile": { "handle": "kai", "displayName": "Kai",
                     "avatarUrl": "…", "address": "0x…", "label": "Kai" } }
```

`author` is unchanged in shape, so nothing breaks — it just says a name now
instead of `0x0000…0000`.

### Credits on a game

`GET /api/games/:idOrSlug` — each entry in `splits` gains `profile`, the same
shape as `authorProfile`, or `null` for a collaborator who was invited by email
and has never signed in. That null is a real state, not a gap: they are on the
splits and paid from the first sale regardless.

### One fix worth knowing

**Every `/api/games/:id/…` route now accepts a slug.** They used to return
`500` for one — so `/api/games/deadzone/reviews` was a server error while
`/api/games/deadzone` worked. Both work now, and an unknown game returns `404`
where reviews and comments previously returned an empty list.

---

## 12. The wishlist

`likes` is the wishlist now. **Nothing breaks** — every response still carries
`liked` and `likeCount`, and `POST /api/games/:id/like` still toggles exactly as
it did. What is new sits beside them.

Every response from all three routes has the same body:

```json
{ "wishlisted": true, "wishlistCount": 12, "liked": true, "likeCount": 12 }
```

`POST /api/games/:id/wishlist` adds (201 the first time, 200 after — adding
twice is not an error). `DELETE /api/games/:id/wishlist` removes. Prefer these
over the toggle for a button that says "on your wishlist": a toggle undoes
itself on a double click.

### The list

`GET /api/me/wishlist`:

```json
{ "onSale": 2,
  "items": [ { "addedAt": "…", "notifyOnDrop": true,
               "game": { "id": "…", "slug": "…", "title": "…", "coverUrl": "…",
                         "studio": { … }, "priceUnits": 250000, "priceUsd": 0.25 },
               "savedAtUnits": 500000, "savedAtUsd": 0.5,
               "changeUnits": -250000, "percentOff": 50,
               "stillForSale": true,
               "agentMaxUnits": 300000, "agentNote": "if it's still fun-looking" } ] }
```

`percentOff` is the headline — it's the price now against what it cost when
they saved it, which is the comparison a wishlist exists to make. `onSale` is
the count of items with `percentOff > 0`, so a "3 games on your list are
cheaper" banner needs no client-side arithmetic.

`stillForSale: false` means the game was unlisted. It stays on the list on
purpose — someone who saved it should learn what happened to it rather than find
a gap. Only a `removed` game disappears.

`agentMaxUnits` is non-null when this wishlist row is also a **want** — see §18.
There's one agent per person now, not one per game, so this replaced the old
per-row `agent` object.

### Price drops

Lowering a game's price notifies and emails everyone who has it wishlisted,
except people who already own it and people who muted that row. The notification
type is `price_drop` and its payload carries `fromUnits`, `priceUnits`,
`savedAtUnits` and `percentOff` (plus the `…Usd` versions, like every other
notification).

### Public demand — worth building something for

`GET /api/games/:id/demand` (public, slug or id):

```json
{ "gameId": "…", "wishlistCount": 12, "announcedMilestone": 10,
  "topicId": "0.0.10380868" }
```

Every time the count crosses a milestone (1, 5, 10, 25, 50, 100…) it is written
to the public HCS listings topic. Wishlist counts are private platform data on
every other storefront — it's one of the things Steam won't give away. Here
anyone can verify the number on the Mirror Node. That's a real differentiator
and it currently has no UI at all.

The developer's own view (`GET /api/games/:id/manage`) gains
`stats.wishlisted` — how many people are waiting, which is the number that
decides whether a discount is worth running.

---

## 13. Cloud saves

A build runs on its own sandboxed origin, so whatever it writes to
`localStorage` lives in that browser on that machine. Clear site data or open
the game on a phone and progress is gone. These four routes are the
somewhere-else it can live.

Three slots per person per game, **512KB each**. The data is opaque — send
whatever you dumped out of the game's storage, we never parse it.

```
GET    /api/games/:id/saves        -> { slots: [...], maxSlots: 3, maxBytes: 524288 }
GET    /api/games/:id/saves/:slot  -> the slot, including `data`
PUT    /api/games/:id/saves/:slot  <- { data, label?, device?, baseVersion? }
DELETE /api/games/:id/saves/:slot
```

The list carries metadata only, no payloads:

```json
{ "slot": 0, "label": "Chapter 3", "sizeBytes": 812, "checksum": "…",
  "device": "laptop", "version": 4, "updatedAt": "…" }
```

Same gate as play sessions: paid games need ownership, free ones don't.

### The one thing to get right: `baseVersion`

Send the `version` you last read. If the slot changed elsewhere since, you get
`409 SAVE_CONFLICT` instead of silently overwriting it, and the error `details`
describe the other side so you can show both:

```json
{ "error": { "code": "SAVE_CONFLICT", "message": "…",
  "details": { "currentVersion": 5, "updatedAt": "…", "device": "phone",
               "sizeBytes": 940, "checksum": "…" } } }
```

**Don't auto-merge.** There's no general way to merge two opaque blobs and
guessing loses progress — show both and let the player pick. Omitting
`baseVersion` means "overwrite, I know", which is right for a first write.

`checksum` is a sha256 of the data, so you can verify a round trip.

### The bridge

The backend half is done; the browser half can't be. The build is on an
isolated origin, so the page can't read its `localStorage` directly — it needs a
small script injected into the build's frame that reads and writes storage on
request and `postMessage`s it out. Roughly:

1. On boot: `GET /saves`, pick the newest slot, `GET /saves/:slot`, post the
   payload into the frame **before** the game starts reading storage.
2. On exit, and on an interval: read storage back out, `PUT` it with the last
   `version` you saw.
3. On `409`: show both sides, let the player choose.

---

## 14. Studio management

A studio used to be write-once, same as a game was before §10: no way to
remove someone, fix a role, resend a lost invite, leave, or hand it off. All
five now exist, all under `/api/studios`.

**The roster (`studioMembers`) and the credit ledger (`splits`) are separate.**
`splits` is permanent — every share ever paid or held stays exactly where it
is, no matter what happens to someone's membership. Removing a member is safe
to build at all only because of that separation: it changes whether someone is
"on the team" right now, never what they earned.

### Removing, leaving, promoting

```
DELETE /api/studios/:id/members/:memberId     manager-gated
POST   /api/studios/:id/leave                 self, no body
PATCH  /api/studios/:id/members/:memberId     { role: "owner" | "member" }
POST   /api/studios/:id/members/:memberId/resend-invite
```

Remove and leave return the same shape:

```json
{ "outcome": "deleted", "member": { "id": "…", "handle": "…", "active": false } }
```

`outcome` is `"deleted"` (nobody had credited them on anything — the row is
just gone), `"deactivated"` (they're on a split or a held payout somewhere —
the row survives, but they drop off every roster and permission check), or
`"already-inactive"` (idempotent — calling remove twice isn't an error). Show
the difference if you want to; both mean the same thing from the UI's point of
view — they're no longer on the active team.

`role: "owner"` here means **manager**, not the studio's actual founder — see
below. A manager can edit the listing, invite people, and manage the roster,
the same as the founder, short of transferring the studio itself.

### The founder is special

Every one of the routes above returns `409 IS_FOUNDER` if targeted at the
studio's actual founder (`studios.owner_user_id`'s own membership row) — it
can't be removed, demoted, or self-left. The only way out for a founder is:

```
POST /api/studios/:id/transfer
{ "toMemberId": "<an accepted, active member's id>" }
```

**Founder-only** — a promoted manager can't call this, even though they can do
almost everything else. The target has to already be an accepted, active
member. After transfer, `studios.owner_user_id` is the new person and their
role is set to `owner` (manager); the old founder keeps their existing role and
is now just a manager like anyone else — including being able to leave.

---

## 15. Developer replies, and taking your own words back

**A developer can reply to a review**, once, from the studio:

```
POST   /api/reviews/:id/reply    { "body": "…" }    manager-gated
DELETE /api/reviews/:id/reply                        clears it
```

The reply lands as `developerReply` / `developerReplyAt` on every review row
`GET /api/games/:id/reviews` already returns — no new field to fetch. Posting
again overwrites; there's no thread. The reviewer gets a `review_reply`
notification on the *first* reply only, not on an edit of it.

**Reviews and comments can be deleted:**

```
DELETE /api/reviews/:id      the reviewer only
DELETE /api/comments/:id     the author, or a manager of the game's studio
```

The second path on comments is moderation-lite for a developer's own page —
they can remove spam or abuse under their own listing without a global
moderator. There's no equivalent on reviews: those are gated by real
ownership already, and a developer silencing criticism of their own game is
a different thing from removing an off-topic comment.

---

## 16. Reporting reviews and comments

`POST /api/reports` (games) already existed. This is the same idea for the two
surfaces it can't cover:

```
POST /api/reports/content
{ "targetType": "review" | "comment", "targetId": "…", "reason": "…" }
```

**It does nothing automatically.** A game report delists on submission because
a false positive there is cheap to undo; hiding a review the instant it's
reported would hand any developer a one-click way to silence honest criticism
of their own game, so this only queues the report for a human. Nothing about
the review or comment changes until someone resolves it.

Refuses reporting your own review or comment, and `404`s on a target that
doesn't exist.

### Learning what happened

Both this and the original game-report path now notify the reporter when
their report is resolved — a `report_resolved` notification either way, not
only when something was removed:

```json
{ "type": "report_resolved",
  "payload": { "reportKind": "review", "targetId": "…",
               "gameId": "…", "slug": "…", "title": "…", "action": "removed" } }
```

`reportKind` is `"game"`, `"review"`, or `"comment"`. `action` is `"none"` or
`"removed"` for content, or the game-report actions (`"none"`, `"delisted"`,
`"removed_from_storage"`) for a game.

---

## 17. Sales

A sale is a **scheduled price** — it starts, it ends, and it puts the old price
back on its own. The developer never has to remember to change it back, which
is most of why sales barely existed before.

```
POST   /api/games/:id/promotions
{ "salePriceUnits": 200000, "endsAt": "2026-09-14T18:00:00Z",
  "startsAt": "2026-09-12T18:00:00Z" }     // omit startsAt to begin now
```

`salePriceUnits` must be **below** the current price, and `endsAt` must be in
the future. Returns `409 PROMOTION_EXISTS` if the game already has one
scheduled or running — one at a time, because overlapping sales have no
coherent price to return to.

```
GET    /api/games/:id/promotions     public, slug or id
PATCH  /api/games/:id/promotions/:promotionId   { "endsAt": "…" }  — later only
DELETE /api/games/:id/promotions/:promotionId   ends it early, price restored
```

```json
{ "active": { "id": "…", "status": "active",
              "salePriceUnits": 200000, "salePriceUsd": 0.2,
              "basePriceUnits": 600000, "basePriceUsd": 0.6,
              "percentOff": 67,
              "startsAt": "…", "endsAt": "…",
              "hcsStartTxId": "0.0.x@…", "hcsEndTxId": null },
  "history": [ … ] }
```

### Things that will bite if you don't know them

**`PATCH /api/games/:id` with a `priceUnits` returns `409 PROMOTION_ACTIVE`
while a sale is running.** The sale owns the price until it ends — editing
underneath it would be silently undone at `endsAt`. The error carries
`promotionId` and `endsAt`; send the person to the sale instead.

**Extending only moves the end later.** A pulled-in deadline would strand
anyone who read the original. To end early, `DELETE` — which is announced.

**`GET /api/games/:idOrSlug` now carries `promotion`** — the running sale, or
`null`. Render `endsAt`: a discount with a visible countdown is a different
thing from a cheap game, and it's the part people act on.

**A sale sends the same `price_drop` notifications** a manual price change
does. Nothing new to handle.

### Worth building a countdown for

Both the start *and* the end of every sale are written to the public HCS topic,
and the message carries `endsAt`. That means the deadline isn't our claim — it's
on a public ledger before the sale even matters. Same verifiability story as the
price history in §10.

---

## 18. The wishlist agent

**Rebuilt as one agent per person, not one per game.** One wallet, one shared
budget, several **wants** — a wishlist row upgraded with a max price and an
optional note. `POST /api/agents` and `/api/agents/:id` are gone; both 404 now.

```
POST /api/me/agent
{ "mode": "autonomous", "onTimeout": "buy", "expiresAt": "2026-12-01T00:00:00Z",
  "ensLabel": "kai-agent" }             // ensLabel optional
```

`mode` is `"autonomous"` (acts on its own, tells you after) or `"ask_first"`
(asks before an ambiguous or contested buy, but only when there's genuinely
time to — see below). `onTimeout` (`"buy"` or `"skip"`) is what happens if an
ask-first question goes unanswered past its deadline. `ensLabel` claims a
subname the same way a studio does; omit it and the agent has no name, just
an address. Returns a wallet address — fund it like any withdrawal
destination:

```json
{ "id": "…", "status": "draft", "agentAccountId": null,
  "agentEvmAddress": "0x…", "ensLabel": "kai-agent", "ensTxId": "0.0.x@…" }
```

`GET /api/me/agent` once funded:

```json
{ "id": "…", "status": "watching", "balanceUnits": 500000, "balanceUsd": 0.5,
  "mode": "autonomous", "expiresAt": "…", "ensLabel": "kai-agent" }
```

`status` moves `draft` → `funded` → `watching` on its own once money lands and
identity anchors — nothing to poll for beyond `GET`. `DELETE /api/me/agent`
retires it and refunds whatever's left, in one step, no separate withdrawal.

### Wants

```
PATCH /api/games/:id/wishlist
{ "agentMaxUnits": 300000, "agentNote": "if it's still fun-looking" }   // set
{ "agentMaxUnits": null }                                                // clear
```

Requires an agent to already exist (`422 NO_AGENT` if not — create one first).
Also checked live against the Mirror Node, not a cached balance: `agentMaxUnits`
can't exceed what the agent's wallet actually holds right now.

### What it does

The agent buys a want the moment the game's price drops to or below
`agentMaxUnits` **and** it's still unowned, picking whichever wants currently
fit its balance if several qualify at once (soonest-ending sale first). That
much never calls a model and never costs anything beyond the games themselves.

**When more is eligible than the balance covers**, a real model call decides
what to do with the rest — buy some now, hold others for a bounded wait (only
ever when there's a real sale deadline to bound it), or decline. This is the
only thing that spends on "thinking" rather than games, and it's small and
reported separately — see `GET /api/me/agent/decisions`'s `inferenceCostUnits`.

A purchase notifies once per decision, not once per game — buying three wanted
games in one pass is one `agent_purchased` notification and one email, not
three. `GET /api/me/agent/decisions` is the audit trail: what was considered,
what was chosen, `reasoning` (null for an obvious buy, real text for a
contested one), and `inferenceCostUnits`, newest first.

**The game lands in your library, not the agent's**, even though the agent's
wallet paid — that's the whole point of delegating the budget rather than the
taste. Nothing to build for this; it's the same `GET /api/me/library` as any
other purchase.

An agent past `expiresAt` retires and refunds itself automatically, the same
`agent_expired` notification/email shape as a manual `DELETE`.

### Ask-first

In `ask_first` mode, a contested round that's genuinely close **and** has
enough time before any sale ends sends an `agent_asked` notification and email
instead of buying immediately — the recommendation, the reasoning, and a
deadline. Answer it:

```
POST /api/me/agent/decisions/:id/respond
{ "action": "buy" }     // or "skip", "remove", "keep"
```

`buy` executes the recommendation (re-checked against the current price and
balance — it can still turn out to have become unaffordable or already owned
in the meantime, in which case nothing is charged). `skip` and `keep` both
decline this round without changing the want; `remove` also clears the want's
`agentMaxUnits`, same as `PATCH .../wishlist` with a null. Answering twice, or
answering after the deadline already resolved it automatically, returns `409
ALREADY_RESOLVED` — check for that rather than treating a slow double-tap as
a bug. `422 NOT_ASKABLE` means this decision was never a question — nothing to
answer.

**Nothing is ever left hanging.** A question's deadline always fires an
outcome (buy or skip, per `onTimeout`) whether or not anyone answers, and a
fresh price event on any of your wants cancels a still-open question before
deciding fresh — answering a stale one just returns `ALREADY_RESOLVED`.
