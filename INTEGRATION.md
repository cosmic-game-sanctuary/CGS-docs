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
POST   /api/games/:id/pay/prepare       server quotes the price, hands back typed data — see §4
POST   /api/games/:id/pay/complete      browser's signature goes back, server settles
GET    /api/games/:id/owned             authoritative ownership check
GET    /api/games/:id/reviews
POST   /api/games/:id/reviews           ownership-gated, returns the author too
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
GET    /api/studios/ens-availability    ?name= — answers for agents too, see §18
GET    /api/studios/:idOrSlug
POST   /api/studios/:id/members         invite by email
DELETE /api/studios/:id/members/:mid    remove — see §14
PATCH  /api/studios/:id/members/:mid    { role } — promote/demote a manager
POST   /api/studios/:id/members/:mid/resend-invite
POST   /api/studios/:id/leave           leave a studio you're on
POST   /api/studios/:id/transfer        { toMemberId } — founder only
GET    /api/invites/:id                 public — the emailed link lands here
POST   /api/invites/:id/accept          must be the invited address — see §14
POST   /api/me/agent                    create — see §18
GET    /api/me/agent                    status, live balance
PATCH  /api/me/agent                    mode / timeout / expiry, and a name — see §18
DELETE /api/me/agent                    retire, refund
GET    /api/me/agent/decisions          audit trail, newest first
POST   /api/me/agent/decisions/:id/respond   answer an ask-first question — see §18
PATCH  /api/games/:id/wishlist          set or clear a want — see §18
GET    /api/games/:id/trial             config, and your own credit if signed in — see §19
POST   /api/games/:id/trial/chunks/prepare   see §19
POST   /api/games/:id/trial/chunks/complete  see §19
GET    /api/users/:handle               public profile — see §11
GET    /api/users/handle-availability   ?handle=
PATCH  /api/me/profile                  display name, handle, bio, library visibility
POST   /api/me/avatar                   multipart, field `avatar`
DELETE /api/me/avatar
GET    /api/me/purchases                receipts, with the settlement tx id
GET    /api/me                          who you are, wallet balances, your studio — see §6
GET    /api/me/library                  every game you actually hold a key for — see §6
GET    /api/me/earnings                 what you've earned, across every studio — see §6.2
POST   /api/me/claim/:gameId            take your share out of that game's vault — see §6.2
GET    /api/studios/:id/earnings        what the studio made, team only — see §6.2
POST   /api/me/withdraw/prepare         validate, then hand back the transaction to send — see §6.1
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

You do **not** build the authorization yourself. It takes two calls:

```
POST /api/games/:id/pay/prepare    requireAuth, no body
  -> { status: "prepared", intentId, typedData, payTo, expiresAt, amountUnits, asset }
  -> or { status: "granted", …grant }  when it's free or already owned

POST /api/games/:id/pay/complete   requireAuth  { intentId, signature }
```

**The browser signs, not the server.** Signing server-side would require every
buyer to delegate their wallet to the store first, which is standing permission
to move their money and a much larger thing to ask than one game. So the server
quotes the price and builds the authorization, the browser signs it, the server
settles it.

**What gets signed is now EIP-712 typed data, not a raw hash.** `typedData` is an
EIP-3009 `TransferWithAuthorization` — pass it **whole and unmodified** to
`eth_signTypedData_v4`:

```ts
const signature = await provider.request({
  method: 'eth_signTypedData_v4',
  params: [wallet.address, JSON.stringify(prepared.typedData)],
})
```

Do not rebuild, reorder or re-type any of it. The signature only verifies if
every byte of the domain and message matches what the server built, so
reassembling it client-side is just an opportunity to get it subtly wrong.

This is strictly better for the person signing than the old raw hash: a wallet
can *show* typed data, so "pay 0.30 USDC to this address" is legible where a
32-byte hash was not. `payTo` is the game's own `SplitVault` — the money never
passes through an account we control.

**The buyer pays no network fee at all.** Circle's facilitator submits the
transfer and covers the gas, so a wallet holding *exactly* the price of a game
can buy that game. Verified on testnet: a buyer funded with the price and
nothing else ended at a zero balance holding the key.

**An authorization is good for 30 minutes**, not the ~2 minutes a frozen Hedera
transaction allowed. A buyer who reads the page before signing is no longer a
buyer whose payment expired. `PAYMENT_INTENT_EXPIRED` is still possible and
still means nothing was charged.

**Two refusals worth handling by name.** `409 ALREADY_OWNED` — that wallet holds
the key already and nothing was charged. `409 PAYMENT_PENDING` — settlement did
not resolve inside the wait window; it is **not** a failure and the payment may
still land.

> ### ⚠ `PAYMENT_PENDING` changed status, and the old behaviour was dangerous
>
> **It was `202`. It is now `409`.** Changed 2026-10-03, and worth
> understanding rather than just patching, because the old shape broke any
> client that did the obvious thing.
>
> `202` is a 2xx. A client that checks `response.ok` — ours did — saw success,
> took the `{ error: … }` envelope as the `AccessGrant` payload, and carried on
> into the boot sequence with every field `undefined`, while the buyer's money
> may genuinely have been taken. A silent fake success is the worst possible
> answer to "did my payment work", so the status code now says no to every
> client, including ones that have never heard of this code.
>
> **Retrying is now actually possible, which it previously was not.** The
> advice here used to be "re-request rather than re-signing" — correct advice
> that the server made impossible: `/pay/complete` consumed the payment intent
> before submitting, so once a pending answer came back there was nothing left
> to re-request with, and a buyer's only route forward was the re-sign this
> paragraph warns against. The intent now survives a pending outcome.
>
> **So: on `409 PAYMENT_PENDING`, call `/pay/complete` again with the same
> `intentId`.** Not a new `prepare`, not a new signature. That resubmits the
> identical authorization under the identical idempotency key, so Circle
> converges on the one payment and the token's own nonce makes a second
> transfer impossible. Back off a second or two between tries and give up after
> a handful; the authorization is valid for 30 minutes, so there is no rush.
> **Never call `prepare` again in response to this** — a fresh signature is a
> genuinely new payment and is the one way to get charged twice.

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
until this user owns one or has an **accepted** invite into one.

`role` is the **membership role**, and `"owner"` here means *manager* in the
sense §14 uses: the founder, or someone promoted to it. It is the field to
read for "can this person change the listing". Before 11 Sep it was hardcoded
— `"owner"` for the studio you founded, `"member"` for every other — so a
promoted manager looked like a plain member on every client and nothing they
had been promoted to do became available. Same field, same values, now read
off the row.

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
POST /api/me/withdraw/prepare    requireAuth   { to, amountUnits? }
```
`to` is an EVM address. Omit `amountUnits` to send everything the wallet can
afford to send — a little is held back to pay for the transfer itself, reported
as `reservedForGasUnits`.

```json
{ "intentId": "…", "to": "0x…", "asset": "0x3600…0000",
  "amountUnits": "10000000", "amountDisplay": 10.0, "assetDecimals": 6,
  "reservedForGasUnits": "20000",
  "transaction": { "to": "0x…", "value": "10000000000000000000", "chainId": 5042002 },
  "expiresAt": "…" }
```

**Send `transaction` yourself**, with the owner's own wallet, then report the
hash back:

```http
POST /api/me/withdraw/complete   requireAuth   { intentId, txHash }
```
Returns `{ status: "sent", transactionId, to, asset, amountUnits }`, confirmed
against the chain rather than taken on your word — a hash that didn't move that
amount to that address is refused.

`value` is **wei** (18 decimals), because that is what a wallet's `value` field
expects and USDC is Arc's native token. `amountUnits` is the 6-decimal figure to
display. Those are the same amount written two ways; don't mix them up.

**Nobody needs a second asset to withdraw.** The fee is paid in the USDC being
withdrawn, so a wallet with money in it can always afford to move that money.
This is why the server no longer builds or submits the transfer at all. (On
Hedera the operator had to pay the fee, because a wallet holding only USDC would
be a wallet you cannot
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
              "claimable": {...}, "claimed": {...}, "asset": "0x3600…0000" },
  "games": [ { "gameId":"…", "slug":"…", "title":"…", "status":"published",
               "studio": {...}, "sales": 2, "gross": {...},
               "yours": { "pct": 60, "role": "code", "earned": {...} },
               "plays": 8, "likes": 3, "reviews": 1, "rating": 5 } ],
  "claims": [ { "gameId":"…", "gameTitle":"…", "gameSlug":"…",
                "vault": "0x…", "bps": 4750,
                "earned": {...}, "claimed": {...}, "claimable": {...} } ] }
```

**`held` and `failed` are gone, and so is the idea behind them.** There used to be
three states a share could be in — paid, held because the person had no account
yet, or failed — and all three existed because the server moved the money. It does
not any more: a sale credits the game's `SplitVault` directly and the contract
divides it, so a share is either still in the vault (`claimable`) or already
withdrawn (`claimed`). Nothing can get stuck, and nobody is owed a transfer that
failed.

Every figure in `claims` is read from the contract rather than from our tables, so
it is what the chain will actually pay and `vault` is checkable on the explorer.
`bps` is that person's share of each sale in basis points — 4750 is 47.5%.

```http
POST /api/me/claim/:gameId    requireAuth
```
Move your share of one game out of its vault.

```json
{ "gameId":"…", "gameTitle":"…", "vault":"0x…", "to":"0x…",
  "amount": {"units":190000,"display":0.19,"assetDecimals":6},
  "txHash":"0x…" }
```
`409 NOTHING_TO_CLAIM` when there is nothing waiting, which is an ordinary outcome
rather than an error worth alarming anyone about. One call per game, because each
game has its own vault and each claim is its own transaction.

**We pay the gas for this, and it is necessity rather than generosity.** Gas on
Arc is USDC, so paying for a transaction means already holding USDC — and a
developer whose first earnings are in the vault holds nothing. `SplitVault.claimFor`
lets anyone pay, and sends only to the payee, so paying for it buys us no say over
the money. A developer who would rather not involve us can call `claim()` from
their own wallet instead.

```http
GET /api/studios/:id/earnings    requireAuth, owner or accepted member
```
Same shape plus `people` — every handle on the studio's splits and what they
earned. Anyone outside the studio gets `NOT_OWNER`. `people[].claimed` is now
always `true` and kept only so the shape does not change under you: every payee
has a payout address from the moment they are invited, because one is generated
for them then and the game's vault names it permanently at publish.

Every money value is `{ units, display, assetDecimals }`. `units` is the truth;
`display` is there so you don't derive it, and nothing should compute with it.

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

**The same route gained `viewerMemberId`, as of 11 Sep.** The id of the
signed-in caller's own row in `members`, or `null` if they are not on the
team (or not signed in) — nobody else's `userId` is ever on this response,
this is the one exception, and it is always the viewer's own. Use it to
suppress manage-controls on someone's own roster row: showing "Remove" or
"Hand over" pointed at the person looking at it is a real bug the UI had
before this existed, not a hypothetical.

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
  new price is live here but hasn't reached `GameRegistry` yet, so wishlist agents
  can't see it. Worth showing — it's the difference between "I put it on sale" and
  "the sale is public."

`coverMediaId` picks an existing image as the cover; it does not upload one.
Uploading is `POST /api/games/:id/media` (multipart, field `media`, up to 8).
`PATCH /api/games/:id/media` takes `{ "mediaIds": [...] }` and reorders —
anything you leave out keeps its relative position at the end, it isn't deleted.

The same `PATCH /api/games/:id` also turns a paid trial on, off, or changes
it — `trialChunkPriceUnits`, `trialChunkMinutes`, `trialMaxChunks`. The first
and third must be sent together, both numbers to turn a trial on or change
it, or both `null` to turn it off — sending one without the other is a
validation error, not a half-applied config. See §19.

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
                "chainTxHash": "0x…", "createdAt": "…" } ] }
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
      "chainTxHash": "0x4839ffdeb7449ca81dbf1c8ed9e7073936c2c753b9cf2cac36ee8e4c7f546a59",
      "explorerUrl": "https://explorer.testnet.arc.io/tx/0x4839ff…" } ] }
```

Newest first. **A row describes a change, not a price** — there is no
`priceUnits`/`priceUsd` on it, only the `from`/`to` pair. Render the movement.

The exact same row shape appears as `priceHistory` inside
`GET /api/games/:id/manage`, so one component can render both.

`chainTxHash` is the point: **every row names the transaction that recorded it on
`GameRegistry`**, so a visitor can verify the whole history on the block explorer
without trusting us. No other storefront can offer that, because they all own the
database their price history lives in. `explorerUrl` is the link to render; both
are `null` only when the recording failed and `npm run listings:retry` hasn't
caught up yet.

> **Renamed from `hcsTxId`/`topicId`.** The field used to hold an HCS message id
> and the topic it was on. It is an Arc transaction hash now, and the companion
> field is a URL rather than a topic id.

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
`settlementTxId`, a ready-made `explorerUrl` for it, the price paid at the
time, the game, and the `key` (`tokenId`, `serial`) it minted.

**Two changes here, 2026-10-03.**

**`explorerUrl` is new, and you should use it instead of building a link.**
Which explorer resolves a transaction is a fact about the network the server
is pointed at, and the server is the only side that knows it. `lib/hashscan.ts`
in the client builds `hashscan.io` URLs, which is a Hedera explorer: it cannot
resolve an Arc transaction at all, and the id-mangling it does on the way
(`@`→`-`) is for a Hedera id format Arc does not use. **That file should go**,
and every place it is used should read a server-sent URL. The same field now
exists on the agent's decisions (`inferenceUrl`), on a claim (`explorerUrl`,
`vaultUrl`), and on a price-history row (`explorerUrl`, already there).

**Trial chunks no longer appear in this list, and they used to.** A chunk is
also a `sales` row against the same buyer and nothing filtered them out, so
the receipts screen listed one row per chunk as though each were a purchase of
a game the buyer may not own. Measured on a real test buyer: four rows, one
actual purchase. Stage 7 made it worse — a Gateway-settled chunk's
`settlementTxId` is a transfer id that resolves nowhere on an explorer, so
three of those four rows would have rendered a receipt with a dead link.
**What a buyer spent trialling is reported by `GET /api/games/:id/trial` as
`spentUnits`/`creditUnits`**, which is the honest place for it: it is credit
toward a purchase, not a purchase. Nothing to change on your side unless you
were counting on chunks being in here.

### Authors on reviews and comments

Both lists now carry `authorProfile` alongside the existing `author` string:

```json
{ "author": "kai", "authorIsEns": false,
  "authorProfile": { "handle": "kai", "displayName": "Kai",
                     "avatarUrl": "…", "address": "0x…", "label": "Kai" } }
```

`author` is unchanged in shape, so nothing breaks — it just says a name now
instead of `0x0000…0000`.

**`POST /api/games/:id/reviews` returns the same shape**, as of 11 Sep. It
used to hand back the bare inserted row, so a review posted from the page
rendered with no author on it until something reloaded the list: the one
review on screen definitely written by somebody was the only anonymous one.
Nothing to change if you already run the response through the same adapter as
the list.

### Who bought it, on a `sale` notification

The `sale` payload now carries the buyer, resolved on this side because only
this side can tell an agent's wallet from a person's:

```json
{ "buyerLabel": "lottie.cgs-sanctuary.eth", "buyerEns": "lottie.cgs-sanctuary.eth",
  "buyerKind": "agent", "buyerAccountId": "0.0.10416169" }
```

`buyerKind` is `"agent"` or `"person"`. `buyerLabel` follows the same order
the rest of the product names people by: ENS name, then display name or
handle. It is `null` in two cases, and both mean **"Someone"** rather than an
error: the account belongs to nobody who ever signed in here, or the buyer has
`libraryPublic: false` and is saying they would rather not have their buying
watched. The payload carried no buyer at all before, so every sale read
"Unknown bought your game" on the one storefront where the buyer is always a
resolvable on-chain identity.

**An agent's purchase names the agent, not its owner.** The GameKey still goes
to the person who funded it — that has not changed — but the account the money
actually left is the agent's, and that is who the studio is told about.

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
  "registry": "0xE7aa837c45caE001bAc6f026B23edf6e96Ee490e",
  "explorerUrl": "https://explorer.testnet.arc.io/address/0xE7aa83…" }
```

Every time the count crosses a milestone (1, 5, 10, 25, 50, 100…) a `Demand`
event is emitted on `GameRegistry`. Wishlist counts are private platform data on
every other storefront — it's one of the things Steam won't give away. Here
anyone can read the number off the chain. That's a real differentiator and it
currently has no UI at all.

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

### An invite link is not a bearer token

`POST /api/invites/:id/accept` compares the signed-in account's email against
the address the invite was sent to. A mismatch is `403 INVITE_EMAIL_MISMATCH`,
with the masked address in both the message and `details.email`. Someone who
already accepted the row is let through regardless, so changing the address on
your account never locks you out of a membership you hold.

Accepting moves real money — it backfills every split naming that row and
releases the payouts held against it — and an emailed link gets forwarded, so
until this check existed whoever opened the link first collected. `GET
/api/invites/:id` now returns `email` for the screen to show, **masked**
(`ka•••@example.com`): the route takes no auth, so the whole address would be
readable by anyone holding the link.

`POST /api/invites/:id/accept` also stops echoing the raw membership row. It
returns `{ id, studioId, handle, role, acceptedAt }` and nothing else; `email`
and `userId` were on it before and belong to the invitee alone.

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
{ "id": "…", "status": "watching", "balanceUnits": "500000", "balanceUsd": 0.5,
  "mode": "autonomous", "expiresAt": "…", "ensLabel": "kai-agent",
  "erc8004AgentId": "896985",
  "identityUrl": "https://explorer.testnet.arc.io/token/0x8004…BD9e/instance/896985",
  "agentAccountId": null, "hcs14Aid": null }
```

`status` moves `draft` → `funded` → `watching` on its own once money lands and
identity anchors — nothing to poll for beyond `GET`. `DELETE /api/me/agent`
retires it and refunds whatever's left, in one step, no separate withdrawal.

> ### ⚠ The agent's identity moved to ERC-8004, and the screen still says Hedera
>
> **New fields, 2026-10-03: `erc8004AgentId` and `identityUrl`.** The agent
> registers itself on Arc's predeployed ERC-8004 `IdentityRegistry` — the
> `agentId` *is* an ERC-721 token id it owns — and `identityUrl` opens that
> token on the explorer. That link is worth putting on the page: it is what
> turns "this agent has an identity" from something we assert into something a
> stranger can check, the same argument the split bar makes about money.
>
> Both are `null` until the agent is **funded**, and that is correct rather
> than missing: registration is a transaction the agent pays for itself, so an
> empty wallet cannot have an identity yet. At `draft`, render "not registered
> yet".
>
> **`agentAccountId` and `hcs14Aid` are dead.** They are the Hedera-era
> identity, they are always `null` on Arc, and they are kept in the response
> only so a client written against the old shape keeps parsing. `Agent.tsx`
> currently branches on `agentAccountId` and prints, to every user, on every
> agent, forever: **"No account on Hedera yet. The first money in makes one."**
> That string is wrong in all three of its claims now. Replace that whole
> branch with `erc8004AgentId` / `identityUrl`.

**Decisions carry `inferenceUrl` too.** `GET /api/me/agent/decisions` rows now
include it alongside `inferenceTxId` — the explorer link for the x402 payment
the agent made for its own reasoning on a contested round. `DecisionFeed.tsx`
builds this link with `hashscanTx()` today, which points at a Hedera explorer
and cannot resolve it. Read the server's field instead.

### Naming an agent after it exists

```
PATCH /api/me/agent
{ "ensLabel": "lottie" }
```

`ensLabel` was create-only, and no screen ever sent it — so in practice no
agent could get a name at all, which matters because an agent's ENS name *is*
its public identity: it is what a sale notification prints (§7 of the testing
round doc), and the reason a stranger can tell one autonomous buyer from
another. It is accepted on `PATCH` now, so it can be claimed from the agent
page once the thing exists rather than being asked for before it does.

**Write-once.** An agent that already has one answers `409
AGENT_ALREADY_NAMED`, and there is no way to clear it: a rename would mint a
second name and leave the first pointing at the same wallet. A retired agent
answers `409 AGENT_ALREADY_RETIRED`. A name that is taken comes back as `422
VALIDATION_FAILED` on `ensLabel`, and a mint that fails on chain as `502
ENS_MINT_FAILED` with nothing written. It is a real Sepolia transaction, so
budget ten seconds or more for the call.

**Check first with `GET /api/studios/ens-availability?name=…`.** Despite the
path it answers for the whole namespace: studio and agent subnames are minted
into one flat subregistry, so a label is either free or it is not, and a
studio and an agent compete for the same one.

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

A want becomes buyable when the game's price is at or below `agentMaxUnits`
and it's still unowned. Working that out never calls a model and never costs
anything beyond the games themselves.

**It does not buy the instant a price drops.** A sale is open until it ends,
so waiting costs nothing and what arrives while it waits can change the
answer. The agent decides at the **wire** — an hour before the soonest sale
it's watching ends — so that one decision sees everything that showed up in
between. A price drop usually just moves that alarm clock and spends nothing.
Where nothing eligible has a deadline at all, there's nothing to wait for and
it decides immediately.

**At that round, when the balance can't cover everything eligible**, a real
model call decides what to buy and what to pass on. This is the only thing
that spends on "thinking" rather than games, and it's small and reported
separately — see `GET /api/me/agent/decisions`'s `inferenceCostUnits`.

For the UI this mostly matters in one way: **a want sitting there unbought
during a live sale is normal, not a stall.** It's waiting for the wire.

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

---

## 19. Paid trials

A developer can let you try a game in paid chunks before buying it. Every cent
spent trialling comes off the price if you go on to buy — it never expires and
it's never a rental, just a running discount.

```
GET /api/games/:id/trial
```

```json
{ "enabled": true,
  "chunkPriceUnits": 30000, "chunkPriceUsd": 0.03, "chunkMinutes": 5,
  "maxChunks": 10, "worstCaseUnits": 300000, "worstCaseUsd": 0.3,
  "chunksConsumed": 2, "chunksLeft": 8,
  "spentUnits": 60000, "spentUsd": 0.06,
  "creditUnits": 60000, "creditUsd": 0.06,
  "owedUnits": 1440000, "owedUsd": 1.44,
  "asset": "0.0.429274", "assetDecimals": 6 }
```

`enabled: false` (with everything else zeroed) means this game doesn't offer a
trial — most games. Public: anyone can read the config. `chunksConsumed`,
`spentUnits`, `creditUnits` and `owedUnits` are yours alone — signed out, or on
a game that doesn't know you, they read zero (and `owedUnits` reads the full
price).

**`owedUnits` / `owedUsd` is what buying costs this caller right now**, credit
already subtracted, from the same function `/download` prices a purchase with.
Added 2026-09-11 because the client had no way to say what a purchase would
cost after a trial and went on showing the undiscounted price. **Show this
number, don't compute `price - credit` yourself** — a second opinion assembled
on your side can only ever disagree with the one that moves money.

**Worst case is knowable before the first chunk**: `chunkPriceUnits ×
maxChunks`, always ≤ the game's price — enforced when a developer sets it, not
left as a mistake to discover mid-trial. Worth surfacing before the first
chunk purchase: "try this for up to $0.30, stop whenever, it all comes off
the price."

### One-time setup: depositing into Gateway (added 2026-10-03, Stage 7)

Chunks settle through Circle Gateway's nanopayment batching now, not the
regular facilitator — a chunk is too small for the gas a normal settlement
costs. Mechanically this only changes one thing for you: **a buyer must
deposit USDC into a Gateway contract once before their first chunk**, the
one part of this whole integration that asks for an extra signature.

`GET /api/games/:id/trial`, when signed in, now also returns:

```json
{ ...,
  "gatewayDeposit": {
    "walletAddress": "0x0077777d7EBA4688BDeF3E311b846F25870A19B9",
    "usdcAddress": "0x3600000000000000000000000000000000000000",
    "chainId": 5042002,
    "availableUnits": "0",
    "availableUsd": 0,
    "needsDeposit": true
  }
}
```

`null` when signed out, when the trial isn't enabled, or when there are no
chunks left — don't prompt for a deposit in any of those cases.

When `needsDeposit` is true, before offering the first chunk, have the
buyer's wallet submit two ordinary transactions (it pays its own gas, same as
any other write on Arc — nothing to prepare/sign/complete through us, this
isn't an x402 payment):

```solidity
// 1. approve USDC spending
IERC20(usdcAddress).approve(walletAddress, amount);
// 2. deposit into Gateway
IGatewayWallet(walletAddress).deposit(usdcAddress, amount);
```

**A sensible `amount` is the trial's own `worstCaseUnits` minus whatever
`availableUnits` already holds, capped at the wallet's balance** — what
`CGS-client` actually shipped, and better than the flat-dollar suggestion this
section originally made. Depositing the worst case means the meter can never
run dry mid-game, which is the failure worth designing out since it would
interrupt someone while playing; capping at the wallet avoids asking for money
that isn't there, since the funding rung before this one only guarantees one
chunk's price. **Leave the buyer's wallet holding a small margin** — don't
deposit down to zero — or they have nothing left for the deposit's own gas
(measured: ~0.0036 USDC for both transactions together).

One deposit lasts across every game with a trial, not just this one: it's a
single Gateway balance, not per-game. After it lands, `availableUnits`
catches up within a few seconds (measured: ~2.3s) — poll `GET /trial` rather
than assuming it's instant, the same as any other read-right-after-write on
this backend.

**`GET /api/me/gateway`, added 2026-10-05 — what's already set aside, outside
any one game's trial.** The unplayed rest of a deposit is real money and
isn't in the wallet balance anywhere else; worth a line on `/money` or
wherever else a balance is shown, not just inside a trial.

```
GET /api/me/gateway        requireAuth
```
```json
{ "availableUnits": "240000", "availableUsd": 0.24,
  "asset": "0x3600000000000000000000000000000000000000", "assetDecimals": 6 }
```

Its own route rather than a field on `/api/me`, deliberately: reading it is a
call to Circle, and `/api/me` runs on every page load and decides whether
sign-in worked — coupling that to a third party's latency would make every
page as slow as Circle's slowest answer, for a number only one page needs.
Bounded at 8 seconds server-side; a failure answers `503
GATEWAY_UNAVAILABLE` rather than a zero, because "we could not ask" and
"there is nothing there" are different claims about somebody's money, and the
second one would be false.

**No withdrawal exists yet.** A Gateway balance can currently only be spent on
a trial chunk — there is no route to take it back out to the wallet. Circle's
own same-chain `withdraw` (one signature, one transaction, ~$0.0035 fee)
would add that; not built, flagged here so it isn't assumed to exist.

### Buying a chunk

Same two-step shape as buying the game — prepare, sign in the browser,
complete — pointed at a different resource:

```
POST /api/games/:id/trial/chunks/prepare
POST /api/games/:id/trial/chunks/complete   { "intentId": "…", "signature": "0x…" }
```

Identical request/response shapes to `/pay/prepare` and `/pay/complete` (§4)
for `prepare` — if that flow is already wired up, sign whatever `typedData`
comes back exactly as before; it's addressed to a different contract now
(Gateway's, not USDC's), but nothing about *how* you sign changed. **One
thing is different in `complete`'s response:** a chunk no longer returns
`settlementTxId` — it returns `gatewayTransferId`, a Gateway transfer id, not
an on-chain transaction hash. Don't link it to the explorer; the money hasn't
reached the vault yet when this responds (Circle's documented design: the
chunk is yours immediately, the on-chain batch that moves the USDC runs
later, on Circle's own schedule). `409 TRIAL_CHUNKS_EXHAUSTED` means
`chunksLeft` was already 0; check `GET /trial` before offering the button.

**Buy the next chunk while the current one is still running**, not after it
ends, so a signature prompt never interrupts play. That's on you — the backend
has no opinion about when a chunk is bought, only that each one is real money
that lands the moment it settles. The client does this on a timer now
(`TrialSession`), topping up ~30s out and stopping when the session closes.

**Budget two counted requests per chunk, not four.** Settling calls our own
gated route over loopback, and those self-calls are exempt from the rate limit
as of 2026-09-11 — before that a metering trial spent four units a minute and
walked into the ceiling mid-session. A `429` now answers in the normal error
shape with `code: "RATE_LIMITED"`; it means slow down, not stop, so back off
and retry rather than tearing the session down.

No GameKey, no ownership, on a trial chunk — just five (or however many)
minutes and a credit that's now a little bigger.

### Buying the game after trying it

Nothing new to call. `GET /api/games/:id/download` already checks who's
asking and reduces what it charges by whatever credit you've earned on that
game — pay the difference, or nothing at all if chunks already covered it.
The response shape is identical either way; there's no "trial purchase"
variant to branch on.

---

## 19b. Frontend punch list after the Arc port — 2026-10-03

A full route-by-route and shape-by-shape audit of both repos, done at the end
of Stage 8. **The client builds clean and most of it is already correct** —
purchases, claims, earnings, notifications, wishlist and the whole agent API
module are wired to the right endpoints with the right shapes. What follows is
everything that is not, worst first.

Two things to know about how this list was produced, so you know what to
trust. Everything marked **measured** was confirmed against live testnet data
or a real database row. Everything marked **read** was found by reading the
code: there was no browser-automation tool available in that session, so
nothing below was confirmed by clicking it. Where a claim is about what a user
sees on screen, treat it as a strong inference and worth 60 seconds in a
browser before you start.

> **All six items done, 2026-10-05.** Suparno shipped and browser-tested all
> six (`c4a6c0f`..`3804cea` on `CGS-client`, plus the three backend
> prerequisites in `0a51ac0`..`7a68dd4` on `CGS-server`) — not just fixed, but
> clicked through with a real Privy wallet, which found four more bugs no
> check script could have (they sign with a raw key, never through Privy): the
> embedded wallet sending on Ethereum mainnet instead of Arc, Privy's own
> "sign this" modal popping over a running game every minute, the trial gate
> losing the deposit requirement across a sign-in, and an upload timeout
> shorter than a real publish takes. Kept below as the record of what the
> first pass found; the live details and what's still unclicked (publish,
> withdraw, agent funding, a signed-out trial) are in `PROGRESS-LOG.md`'s
> 2026-10-05 entries.

### 1. Trials are completely broken, and need new UI — **read**

`GET /api/games/:id/trial` → `gatewayDeposit` is the new field; §19 has the
full flow. A chunk now settles through Circle Gateway, which needs a one-time
on-chain deposit before the *first* chunk will ever succeed, and nothing in the
client does that deposit — grepping the repo for `gateway` or `deposit` finds
nothing related. So every chunk fails immediately with an
insufficient-funds-shaped error, no matter the wallet balance, because the
balance Gateway checks is a different balance.

Not a regression: this UI never existed, because the backend half of it only
landed in Stage 7. It is the one part of the entire Arc port that *adds* a step
for a user, so it is worth designing rather than bolting on — §19 and §20's
"Paid trials" both cover the shape.

Also: a chunk's `complete` response returns **`gatewayTransferId`, not
`settlementTxId`**. `api/trials.ts` still declares `settlementTxId: string` in
`ChunkBought`, so that field is now always `undefined`. Don't link it to an
explorer; the money has not reached the vault when the call returns.

### 2. `lib/hashscan.ts` should be deleted — **measured**

It builds `hashscan.io` URLs. That is a Hedera explorer and cannot resolve an
Arc transaction. Used in one place today (`DecisionFeed.tsx`, for the agent's
inference payment). The server now sends a ready-made URL everywhere it
returns a transaction id: `explorerUrl` on a receipt, on a claim, on a
price-history row; `inferenceUrl` on a decision; `identityUrl` on an agent;
`vaultUrl` on a claim. Read those instead — which explorer is correct is a
fact about the network the server is pointed at, and only it knows.

Verified live: `https://explorer.testnet.arc.io/token/…/instance/896985`
resolves, and the inference transaction it names reports `success` through
Blockscout's own API.

### 3. `Agent.tsx` tells every user the wrong thing — **read**

```
'No account on Hedera yet. The first money in makes one.'
```

Printed on every agent, forever, because it branches on `agentAccountId`,
which is always `null` on Arc. Use `erc8004AgentId` and `identityUrl` — see the
boxed note in §18. Both are `null` until the agent is *funded*, which is
correct and is the state to render as "not registered yet".

### 4. `PAYMENT_PENDING` is now `409`, not `202` — **measured**

See the boxed warning in §4. The short version: the old `202` meant
`response.ok` was true, so `lib/api.ts` returned the error envelope as an
`AccessGrant` and checkout proceeded with undefined fields on a payment that
may have taken the buyer's money. It is a `409` now, so your existing error
path catches it. **Retry by calling `/pay/complete` again with the same
`intentId`** — never `prepare` again.

### 5. `ApiErrorCode` is missing codes the client branches on — **measured**

The type is `ApiErrorCode | string`, so unknown codes work at runtime; this is
about autocomplete and about knowing the code exists. Emitted by the server and
absent from the union, the ones that actually matter:

`ALREADY_OWNED` · `PAYMENT_PENDING` · `NOTHING_TO_CLAIM` · `NO_AGENT` ·
`CHAIN_NOT_CONFIGURED` (503, a server misconfiguration rather than anything a
user did) · `ABOVE_MANDATE` · `ALREADY_RESOLVED` · `AGENT_EXISTS` ·
`AGENT_ALREADY_RETIRED` · `NOT_ASKABLE` · `HANDLE_TAKEN` · `STUDIO_EXISTS` ·
`WITHDRAW_FAILED` · `UPLOAD_REJECTED` · `MODEL_UNAVAILABLE`

And declared in the union but never emitted any more:
`PAYMENT_SIGNATURE_INVALID`, plus the `payout_held` / `payout_settled` family
of notification types — there is no held-payout state on Arc, so nothing can
raise them.

### 6. Dead Hedera code that degrades safely — **measured**, low priority

None of this breaks anything today. Worth a cleanup pass, not an urgent one.

- `WithdrawPanel.tsx` has an HBAR asset toggle. `/api/me` no longer returns
  `hbar`, so `session.hbar` is always `0` and the toggle is gated on
  `> 0` — permanently invisible. The whole `HBAR` / `HBAR_DECIMALS` /
  `isHbar` branch is dead. Its address hint still reads "A Hedera account id,
  or a wallet address."
- `api/me.ts` declares `hederaAccountId`, `hbarUnits`, `hbar`; the server
  sends none of them. They land as `undefined` and fall back cleanly.
- `ProfileMenu.tsx` passes `session.hederaAccountId` (always null) and has a
  comment about "the first top-up also creates the Hedera account".
- `SalePanel.tsx` prints `Announced as {promotion.hcsStartTxId}`. The field
  name is stale but **the data is correct** — it holds an Arc transaction hash
  now. Worth renaming eventually; worth linking with an explorer URL sooner.
- `api/profiles.ts` has `hcsSaleTxId` (always null now) and a comment telling
  you to look `settlementTxId` up on the Mirror Node.

### 7. Not a bug, but the notes that sent me looking — **measured**

`CGS-client/CLAUDE.md`'s "Next up" section still says `POST /api/agents` and
`GET /api/agents/:id` are 404 and that `src/mocks/agent.ts` and `AgentPanel`
point at nothing. **All three statements are out of date**: the agent module
is correctly on `/api/me/agent`, `src/mocks/agent.ts` is deleted, and there is
a real `components/agent/` directory. I repeated that stale claim to Kai
before checking the code, so it is worth fixing in that file before it
misleads anyone else.

---

## 20. What's left on the frontend, and how I'd build it

Three surfaces are fully live on the backend and have no screen yet: the
agent, sales, and trials. Concrete suggestions below — flow, the calls
involved, and the shape I'd give each one. Treat these as a starting point,
not a spec; you know the design system, I don't.

### The agent

**It's a page, not a modal.** An agent is an ongoing thing that runs for
weeks, not a one-off action — closer to "your wallet" than to "confirm this
purchase." `/agent` (or under `/me/agent`), reachable from the profile menu
and from a shortcut on any wishlisted game ("watch with your agent →").

**First visit, no agent yet:** one screen explaining the pitch (pick games
and a max price, the agent buys when the math works, you never watch a sale
timer) and one button. Creating it asks for mode (autonomous / ask-first)
and, optionally, a name (`ensLabel`) — skip the ENS field entirely in v1 if
it's not worth the UI real estate; it's genuinely optional server-side.

**After creation:** it needs money before it can do anything, and that's
just `POST /api/me/withdraw/prepare` pointed at `fundAddress` instead of an
external address — the exact withdraw flow you've already built, so this
should be close to free.

**The agent's page, once funded**, three things stacked:
1. Balance, prominently — people will worry it's underfunded before they
   worry about anything else.
2. The want list — each row a game, its max price, its note, a remove
   button. This *is* the wishlist with a filter applied
   (`GET /api/me/wishlist`, rows where `agentMaxUnits` isn't null) — don't
   build a second list, add a column to the one you have.
3. Recent decisions (`GET /api/me/agent/decisions`) — bought/held/declined/
   asked, with `reasoning` shown when it's not null. This is the "what is my
   agent actually thinking" screen and it's worth making legible; it's also
   the strongest demo moment.

**Setting a want** lives on the game itself, not on the agent page — next to
the existing wishlist heart, "watch with agent" opens a tiny inline form
(max price, optional note) that calls `PATCH /api/games/:id/wishlist`.
`422 NO_AGENT` means send them to create one first.

**Ask-first is the one that needs its own attention.** When a question comes
in (`agent_asked` notification), it needs to be impossible to miss and easy
to act on without leaving what you're doing:
- The bell gets a normal notification, same as any other type.
- The agent page shows it as a highlighted card at the top while it's
  unresolved, reasoning and deadline visible.
- Both places offer the same four buttons: Buy / Skip / Remove / Keep, each
  a single call to `POST /api/me/agent/decisions/:id/respond`. `409
  ALREADY_RESOLVED` means someone (or the deadline) beat them to it — show
  what actually happened, don't treat it as an error.

No polling loop needed beyond what you already have for notifications; the
agent page can just refetch on mount and after any action.

### Sales

Additive to what already exists — no new page.

**On the game card and detail page**, wherever price shows: if
`GET /api/games/:idOrSlug`'s `promotion` is non-null, show the discount and a
countdown to `endsAt` instead of (or beside) the plain price. This is the
one piece worth real visual weight — a sale with a visible deadline is what
makes the demand curve pitch land, and the deadline is provably real (it's
on HCS), which is worth saying somewhere on the page.

**On the manage screen**, a "Start a sale" action next to the price field —
a small form (sale price, optional scheduled start, end date), same modal
pattern you'd use for anything else there. `409 PROMOTION_ACTIVE` on a
direct price edit means send them to the sale instead of failing silently.
An active sale shows extend/end-early controls in place of the edit form.

### Paid trials

**The buy button gets a sibling, not a new page.** On a priced game with
`GET /api/games/:id/trial`'s `enabled: true`, a secondary "Try for up to
$X" button next to Buy — `worstCaseUnits` is the number to show, up front,
so there's no meter anxiety to design around.

**If `gatewayDeposit.needsDeposit` is true, that button opens a small "add
funds to play by the minute" step first**, not the game. One amount field
(a sensible default, a few chunks' worth), two wallet confirmations (approve,
deposit — see §19), then straight into the trial. This is deliberately a
one-time thing across every game with a trial, so after the first time most
people see "Try for up to $X" go straight to the game with no interruption.

**Starting a trial boots the game exactly like a real play session** — same
isolated origin, same iframe — because that's the whole pitch (no separate
demo build). The first chunk is bought transparently before it boots
(`POST /trial/chunks/prepare` → sign → `POST /trial/chunks/complete`, the
identical shape your `/pay/prepare`+`/pay/complete` component already
knows, just pointed at a different pair of URLs).

**A small persistent HUD during the session** — corner-positioned, not a
modal, the same reasoning as checkout being an overlay rather than a route:
the game is running underneath it and shouldn't be interrupted. Shows
chunks left and credit earned so far, and buys the next chunk while the
current one still has time left (not after it ends), so a signature prompt
never freezes play. A "Buy now — $X" button is always visible in it; that
call is just `GET /download`, unchanged, since credit is applied
automatically.

**On the manage screen**, trial config sits next to price: chunk price,
minutes, max chunks, with a live "worst case: $X" computed client-side as
they type (server enforces `chunkPrice × maxChunks ≤ price` regardless —
this is just so they see the number before submitting, not a substitute for
the real check). Both fields null clears the trial.

### Suggested order

1. **Sales** — purely additive, smallest surface, no new screen.
2. **Trials** — one button, one HUD component, reuses the payment-signing
   code you already have.
3. **The agent** — the largest piece, and the one with a genuinely new page.

None of the three depend on each other. Pick whichever's most useful for the
demo first.
