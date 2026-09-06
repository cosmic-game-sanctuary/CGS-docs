# Progress log

Shared status across the three repos. Two people, two halves, one file.

**Before you start:** read Status and Blockers. That is the whole handover.

**When you stop:** add one entry at the bottom of the Log, update the half of Status you moved, and add or clear blockers.

Keep this file to **what the other person needs**: anything crossing the repo boundary, any decision that changes the contract, anything that unblocks them. Work only one repo cares about belongs in that repo's `CLAUDE.md`, not here. Append; never rewrite someone else's entry.

Entries are tagged by **side**, not by person. Suparno is mostly on the frontend and Priyanshu is mostly on the backend, but either of us can work on either, and the log should show what actually happened rather than who usually does what. Sign your name so the other one knows who to ask.

Log entry template, copy this:

```markdown
### YYYY-MM-DD · Frontend · <your name>      <- or `Backend · <your name>`

**Shipped:** what now exists and works.
**Changes the contract:** anything the other side has to code against differently. Skip the line if nothing does.
**Needs from you:** what you're waiting on. Put it in Blockers too if it's actually stopping you.
**Next:** what you're picking up.
```

---

## Status

_Whoever moves a half updates it, regardless of whose it usually is._

### Frontend · CGS-client

**Stage:** integration under way, one workflow at a time. Six done: browse signed out, sign in, library and notifications, make a studio, publish a game, and **buy it and play it**. Everything else still runs on mocks until its turn.
**Real, not mocked, now:** the whole buyer path and the whole dev path. A dev makes a studio with a real ENS subname, publishes a build that is genuinely pinned and tokenised, and can put a collaborator on the splits by email alone. A buyer signs in with Privy, pays through x402 on Hedera with their own wallet, and the game boots in the same tab. Ownership, balance and library are all live Mirror Node answers rather than anything the browser remembers.
**Still on mocks:** reviews, reports, the invite screen, the agent.
**Deployed:** no. **New requirement:** the server keeps uploaded builds on disk under `storage/`, so it needs a persistent volume rather than an ephemeral filesystem. Why, in the 2026-09-06 (2) entry.
**Next:** invites and the held payouts they settle, then reviews and reports, then the agent.
**Note for Priyanshu:** there is a `frontend-integration` branch on CGS-server with the server-side half of all of this. See the log entries below before you branch off `main`.

### Backend · CGS-server

**Stage:** All 8 numbered stages done, plus Stage 9 (profile, library, likes, comments, playtime) on top. Built and verified on live Neon, Hedera testnet, Blocky402, Sepolia, and Pinata. No route returns `501`.
**Working end to end:** a real buyer pays through x402 and the GameKey lands in their account. An agent with its own wallet and its own on-chain identity watches the public listings topic and buys with no human present. A subregistry we own on Sepolia, `cgs-sanctuary.eth` registered under it, studio subnames minted for real on studio creation. A moderation report immediately delists, and a human resolution can restore it, confirm it, or genuinely unpin it from IPFS. `GET /api/me` and `GET /api/me/library` answer "who am I" and "what do I own" for real against the Mirror Node; likes, comments, and timed play sessions are all real, checked against a live-minted GameKey and live testnet transactions, not mocked.
**Deployed:** no.
**Blocked on:** no CSAM-scanning provider chosen — every upload fails closed with `MODERATION_BLOCKED` until one is. Deliberate, not a bug.
**Next:** deployment, so integration doesn't need a local backend. Frontend integration can start now — see [INTEGRATION.md](INTEGRATION.md).

---

## Blockers

Only things stopping work right now.

| Who | Blocked on | Since | Needs |
|---|---|---|---|
| Priyanshu | No CSAM-scanning provider chosen | 2026-09-05 | A vendor decision — Cloudflare's CSAM Scanning Tool, PhotoDNA Cloud, Thorn Safer, or Hive Moderation. See `docs/stage-2.md` §2. |
| Both | The operator holds ~$10 of testnet USDC | 2026-09-06 | Top-ups from faucet.circle.com to `0.0.10375438`. It funds every test wallet **and** pays every split, so checkout testing drains it from both ends. |

_Cleared: server-side signing on a user's Privy wallet. It was never the right question — the browser signs now, and nothing is delegated. See the 2026-09-06 (2) frontend entry._

---

## Decisions

Cross-repo only. Decisions internal to one repo live in that repo's `CLAUDE.md`.

### Backend and chain

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
| Database | Neon (managed Postgres) | One shared cloud DB, nothing to install locally, same place for dev and deploy. |
| IPFS upload | Official `pinata` SDK | Real, typed, matches the v3 Files API. Directory upload (`fileArray`) keeps `index.html` at the CID root. |
| CSAM check timing | At upload, not at publish | Nothing reaches IPFS — public, essentially permanent — without passing first. See `docs/stage-2.md` §4. |
| CSAM provider | None chosen yet, fails closed | No vendor decision exists and no free instant option does either. `runProvider()` in `services/moderation/csam.ts` is the single seam a real one drops into. |
| x402 wiring | `x402ResourceServer` called directly, not `paymentMiddleware` | The route needs two bypass branches (free game, already owns) that depend on the authenticated caller, which the middleware's hook can't see. Same official library either way. |
| Payment signing | ~~Backend, via Privy's `secp256k1_sign`~~ **Superseded 2026-09-06** — the agent still signs here, a person signs in their browser | The premise was that the browser SDK doesn't expose raw-hash signing. It does. Keeping it server-side for a person would have required them to delegate their wallet to us first. See the Frontend table below and the 2026-09-06 (2) entry. |
| Frontend integration | Frontend owns it; backend provides the x402 helper | See [INTEGRATION.md](INTEGRATION.md). Privy in the browser is frontend too. |
| Sales audit trail | A `sales` table, one row per settled purchase | Nowhere recorded a failed split distribution before this, and nothing could retry one. `npm run splits:retry` retries every `failed` row. |
| Agent identity (HCS-14) | Hand-implemented, not `@hashgraphonline/standards-sdk` | The install never finished in 4+ minutes. The AID variant's own spec allows offline derivation from public inputs — no SDK needed, just SHA-384 + Base58. See `docs/stage-5.md` §1. |
| Agent + buyer payment signing | One shared function, `payForGame()` | Both consume our own x402 route as a real client would; the agent uses its own wallet, a logged-in buyer's `/pay` call uses theirs. Same code path on purpose. |
| Privy wallet public key | Derived from a real signature, not read from the API | `Wallet.public_key` is empty in practice on both `create()` and `get()`, despite the type marking it optional. A `secp256k1_sign` response carries a recovery byte, which makes the key recoverable from one real signature. Verified across four trials. |
| ENS subregistry ownership | Platform deploys and owns one subregistry; studios get a scoped role bitmap on their own subname, not their own registry | Studios can point their own name and renew it; they can't unregister or transfer it away from platform control. Same shape as GameKey treasury staying with the operator. |
| ENSv2 Sepolia contract addresses | Sourced from docs.ens.domains, cross-checked against a pinned repo commit's interfaces | The two disagreed on addresses (a stale redeploy the repo commit never caught up to); confirmed which was live with a real `isAvailable`/`getRegisterPrice` call rather than trusting either source blind. See `docs/stage-7.md` §1. |
| Moderation review, who triggers `removed_from_storage` | A CLI script (`scripts/resolve-report.ts`), not an admin route | No admin auth model exists anywhere in this codebase and neither doc describes one. A two-person team reviewing a handful of reports is the same shape as `splits:retry` — an operator runs a script after looking at something, no new infrastructure invented for it. |
| Likes, comments, playtime | Built after all, reversing the original "deliberately not building" cut | That cut was made under time uncertainty; time stopped being the constraint, so the cut no longer applies. `plays`/`likeCount` are computed live from real rows, not a stored counter — see `docs/stage-9.md`. |
| `plays`/`likeCount` storage | Computed at read time (batched, like `rating`), not a stored column | A stored counter can drift from what actually happened; a live count over the real rows can't. Cheap at this catalog's scale. |
| Likes uniqueness | A real DB unique index on `(game_id, user_id)`, not just an app-level check | A race (double-click, two tabs) gets rejected by Postgres itself instead of silently producing two "liked" rows. |
| Session end reliability | No heartbeat, no guaranteed end call | A session with no end call still counts once toward `plays`; it just earns no duration. Simpler and more honest than guessing a duration for a tab that just disappeared. |

### Frontend, where it reaches the contract

| Decision | Choice | Why |
|---|---|---|
| Running builds get their own origin | `VITE_PREVIEW_ORIGIN`, a subdomain | The iframe needs `allow-same-origin` or Godot, Unity and Construct all die on boot. On the app's origin that hands a stranger's game the session and the DOM. **This applies to real published builds too, not just local previews.** Same shape as itch.io's `html-classic.itch.zone`. |
| Splits are public on every listing | handle, role and percent | It's the pitch, so the API has to return handles and roles rather than addresses and numbers. |
| The agent lives on the game listing | Not a page of its own | Buying and setting a price trigger are the same decision made two ways. Agents are per game, so several can watch at once. |
| Checkout is an overlay, not a route | | The claim is that the game boots in the same tab. A navigation unmounts the page and breaks exactly what we're claiming, which also means the payment call's latency is visible and budgeted. |
| Whole UI on mock data before any integration | | Lets the design and the flows be validated without waiting on the backend or Privy. Integration is a later, deliberate phase. |
| Money in the UI | `priceUsd` for display only | All arithmetic will use `priceUnits`. Nothing does money math on a float. |
| Integration order | One workflow end to end, then the next | A layer at a time leaves every screen half-wired and nothing testable. A workflow at a time means each pass ends with something that either works in a browser or doesn't. |
| Who derives display amounts | The server | The float needs the asset's decimals, and the same contract says clients must never hardcode anything about the asset. Sending both is the only way both rules hold. |
| Who writes notification copy | The client | The server sends `type` plus facts. Wording is a design decision that belongs beside the design system, and a sentence stored in a database row cannot be reworded without a migration. |
| A user's public key | ~~Derived lazily, on first signing use~~ **Recovered from the payment signature**, and checked against the address being charged | Deriving it at all needs signing authority over the user's wallet, which we deliberately don't have. And it can't be read from the chain either: a wallet that has only received value has a hollow account with no key published, which is every first-time buyer. Recovery costs nothing, needs no permission, and doubles as proof the signer holds the wallet. |
| Where a purchase is signed | The server builds and freezes the transfer; the **browser** signs the hashes; the server settles | The split falls where the authority does. The alternative was asking every buyer for standing permission to move their money, on the checkout screen, before their first purchase. The agent is unaffected — its wallet is ours, so it still signs in one step, from the same transaction builder. |
| Where a build is served from | The API, not an IPFS gateway | No gateway will serve one: Pinata refuses HTML on `*.mypinata.cloud` and the public gateways can't find freshly pinned CIDs. IPFS keeps the job it is actually good at — proving what a build is, via the CID on the listing and on-chain. A custom gateway domain would reverse this in one function. |
| How a build reaches the player | One ownership-checked zip, unpacked in the browser onto the build origin | Per-file auth would mean a Mirror Node call per asset, and a Godot build makes dozens. Unpacking locally also reuses the pipeline the publish preview already had, so purchased builds and previews run the same way on the same isolated origin. |
| Funding a test wallet | A dev-only server route, off by default | A Privy wallet has no Hedera account until it receives value and no faucet gives testnet USDC to an address, so the operator is the only thing that can start one. Refused at boot when `NODE_ENV=production`. |
| Seed data | No build, no HTS token, on purpose | Pinning and minting spend real testnet resources, and the first purchasable game should come from the publish flow rather than a script that fakes its way past it. A seeded game browses and refuses to sell, which is the right answer for a row with no build behind it. |

---

## The contract

### For the frontend

**Full integration guide, including who does what and the suggested order: [INTEGRATION.md](INTEGRATION.md).** Read that one before starting; this is the summary.

Every endpoint below is live and tested against real Neon + Hedera testnet. Nothing returns `501` any more. Reviews now carry `author` (truncated address) and `authorIsEns` (always `false` until ENS lands).

```
GET   /api/games                 catalog: search, tag, sort, freeOnly, cursor
GET   /api/games/:idOrSlug
POST  /api/studios
GET   /api/studios/ens-availability   ?name= — DB-only check for now, not chain-verified
GET   /api/studios/:idOrSlug
POST  /api/studios/:id/members   invite by email
POST  /api/games                 upload (multipart) — fails MODERATION_BLOCKED until a CSAM provider is picked
POST  /api/games/:id/publish     locks splits, mints the token, writes the HCS listing
GET   /api/games/:id/download    x402-gated — the one non-REST call
POST  /api/games/:id/pay/prepare   builds the transfer, returns the hashes to sign
POST  /api/games/:id/pay/complete  attaches the browser's signatures and settles
GET   /api/games/:id/build.zip   the build itself, ownership-checked once
GET   /api/games/:id/owned
GET   /api/games/:id/reviews
POST  /api/games/:id/reviews     ownership-gated
PATCH /api/reviews/:id
POST  /api/games/:id/like        toggle, no ownership gate
GET   /api/games/:id/comments
POST  /api/games/:id/comments    no ownership gate, unlike reviews
PATCH /api/comments/:id
POST  /api/games/:id/sessions    call when play actually starts
PATCH /api/games/:id/sessions/:id  call when play ends
GET   /api/me                    identity, wallet balance, your studio
GET   /api/me/library            every game you actually hold a key for
POST  /api/agents                returns a wallet address to fund
GET   /api/agents/:id
POST  /api/reports
GET   /api/invites/:id           public, for the emailed link
POST  /api/invites/:id/accept
GET   /api/notifications
POST  /api/notifications/:id/read
```

Five things that affect client code:

- Auth is `Authorization: Bearer <privy access token>`. No cookie, so no CSRF handling.
- Catalog endpoints work signed out. They add an `owned` flag when a token is present. Don't gate browsing behind login.
- Prices come as `priceUsd` for display and `priceUnits` (integer) for math. Don't do money math on the float.
- Never hardcode `payTo`, `feePayer`, or `asset`. All three come back in the 402 response.
- **`keyStatus: "pending"` means boot the game now.** Payment has settled; the GameKey mints in the background. Don't block the player on it — that would put ~6s of chain calls in front of the instant-play moment.

`Game`, `Studio`, `Review` and `SplitMember` match your `src/mocks/types.ts` — the backend model was extended to fit those rather than the other way round. `coverSeed` stays alongside a real `coverCid` so the placeholder art keeps working.

Errors are always `{ error: { code, message, details? } }`. Codes worth handling: `UNAUTHENTICATED`, `WALLET_NOT_FUNDED`, `NOT_OWNER`, `GAME_NOT_PUBLISHED`, `MODERATION_BLOCKED`, `VALIDATION_FAILED`, `RATE_LIMITED`.

### For the backend

What the client already assumes, so the API doesn't have to guess:

- **`CGS-client/src/mocks/types.ts` is the client's read of the contract.** Everything on screen is typed against it. If a shape changes, that file is the diff.
- **Browsing never touches auth.** Catalog, listing, studio pages, reviews and the invite screen all work signed out. Sign-in appears at buy and at publish, nowhere else.
- **Free games still mint a key.** `priceUsd: 0` is a real purchase with a real GameKey, not a bypass.
- **Splits are shown to buyers with handles and roles**, not just percentages, and there is no edit affordance anywhere in the app. Anyone invited by email is on the splits from the first sale whether or not they've accepted. The invite screen says so explicitly, so it has to be true.
- ~~**The session mock is Privy-shaped**… swapping it is one file.~~ Done. `src/mocks/session.ts` is deleted and Privy owns sign-in.
- ~~**Nothing persists.** Publishing pushes into an in-memory catalog.~~ Done. Publishing writes to your database, and what a wallet owns is a live Mirror Node answer.
- Integration seams are still marked `TODO(integration)` in the client. Grep finds what's left: reviews, reports, the invite screen, the agent.

**Everything you flagged as missing an endpoint is answered now:**

- **`/invite/:id`** — `GET /api/invites/:id` (public) + `POST /api/invites/:id/accept` (auth). Accept sets `user_id` and `accepted_at`. No decline endpoint — not accepting *is* the decline, reversible by opening the link again.
- **Notifications** — `GET /api/notifications` + `POST /api/notifications/:id/read`. Plain polling, not SSE, not derived from HCS. Written by whichever handler causes the event — invite *accept* notifies the studio owner (the invitee has no account row to notify until they accept).
- **Publish media** — `POST /api/games` takes `build`, `media` (up to 8 files), `coverMediaIndex` marking the star. No index falls back to the generated cover.
- **Studio creation** — returns the created studio with its `id`. ENS availability is real today but DB-only, not chain-verified — see the endpoint list above.
- ~~**Download path** — `playUrl` is a direct `ipfs.io/ipfs/<cid>/index.html` URL… your self-hosted second origin is only needed for the pre-publish local preview.~~ **Both halves turned out wrong**, and the replacement is `buildPath` + `GET /:id/build.zip`. No gateway will serve a build: Pinata refuses HTML on its shared subdomains and the public ones can't find freshly pinned CIDs. So the second origin is needed for purchased builds after all, and it is what runs them. Evidence in the 2026-09-06 (2) frontend entry.
- ~~**The x402 helper** — the signing bridge lives on the backend, because it needs Privy's server-side raw-signing primitive that the browser SDK doesn't expose.~~ **The browser SDK does expose it** (`secp256k1_sign`), and using it is what removed the need for the buyer to delegate their wallet. The bridge is still yours for the agent; a person's purchase signs in the browser and settles through `/pay/prepare` + `/pay/complete`. Still no Hedera transactions built client-side — the server freezes it, the browser only signs hashes.

---

## Validation

**This is not testing.** It's the record of people outside the two of us touching the thing, which is scored separately from whether it works. Judges want evidence that the product was checked against reality rather than assumed, and the only way to have that evidence in November is to write it down as it happens.

A row belongs here when **someone who isn't us did something real**:

- An indie dev uploaded an actual build, or told us why they wouldn't.
- Someone bought a game without us walking them through it, and we watched where they got stuck.
- A real game got listed by the person who made it.
- A mentor or another team pointed at something and we changed it.
- We posted it somewhere devs actually are and got a reaction, positive or not.

Not a row: our own testing, a green CI run, a screenshot in the group chat, a friend saying it looks nice.

**Write down what actually happened, including the bad ones.** "Three devs said no because they already have an itch audience" is worth more than five polite yeses, and it's the kind of thing that changes what gets built next.

| Date | What we did | Who with | What came back | Changed because of it |
|---|---|---|---|---|
| | | | | |

The weakest axis right now is that the catalog is twelve games we wrote ourselves. Real listings from real devs is the single highest-value row this table could get.

---

## Log

Oldest first, so a new entry goes on the end and the file reads in the order things happened. One entry per side per day, tagged so both of us can append without landing on the same line. If you worked on the other side that day, that's a second entry under the other tag, not a note inside your usual one.

### 2026-09-05 · Frontend · Suparno

**Shipped:** the design language, locked and written up as `CGS-client/DESIGN.md` (Paper Arcade: ink on warm paper, colour rationed to money, ownership, the agent and warnings, deliberately nothing that reads as a crypto app). Then every screen in the brief, on mock data: catalog with filters and search, game listing, checkout overlay, in-browser player, dev upload with a splits editor, studio page and studio creation, library, agent setup, teammate invite, a notifications inbox, and a real 404.

Two flows work end to end. **Buy:** browse, buy, sign in, fund, pay, the key mints and the game boots in the same tab. **Publish:** drop a zip, watch it run, set details and price, split by email to exactly 100%, publish, and the game is actually in the catalog with a working listing.

**Real, not faked:** a dropped zip is genuinely unpacked in the browser with `fflate`, written into the Cache API, and served by a service worker from real URLs, so an actual HTML5 or WASM build plays in the page with no backend at all. Relative paths, `fetch`, workers and WASM all resolve normally, which URL rewriting could never do.

**Changes the contract:** builds run on a **second origin**, not the app's. The iframe needs `allow-same-origin` or most engines die on boot, and granting that on our own origin lets an uploaded game read the session and rewrite the page. This is not a preview-only concern: it applies to the real build coming down from IPFS, so it needs a decision on the server side.

Also worth knowing before the API lands: a few screens exist with no endpoint behind them yet. Listed under **For the backend** above.

**Needs from you:** the x402 helper walkthrough before the download path gets wired, and what `/download` actually returns.

**Next:** a per-screen audit pass, then integration as a deliberate phase in this order: Privy, the API, then x402.

### 2026-09-05 · Backend · Priyanshu

**Shipped:** repo split into three, `.gitignore` fixed on CGS-server. Then Stage 1 for real: Express app, auth (Privy access tokens, no sessions), all 11 tables on Neon, and every route that doesn't need Stage 2+ chain work — catalog, studio create/detail/invite, invites, notifications, reviews, reports, agent creation. Hedera client + Mirror Node helper are real. Tested against the live database and a running server, not mocked — inserted and read back real rows, hit the running app with `curl` for the catalog, filters, 404s, and auth rejection.

**Changes the contract:** invites and notifications added — both already existed on your side. Answered everything in your needs table; see the resolved list above. Nothing returns `501` any more, so integration is unblocked.

Fixed the operator key (it was for a different account than the one configured — caught by deriving the public key and diffing it against the Mirror Node) and hit Stage 1's real done condition: a topic message round-tripped through the Mirror Node.

Then Stage 2: upload really unzips the build, pins it to Pinata as a directory, and publish creates a real HTS token — verified on the Mirror Node that it carries no wipe, freeze, pause or admin key, so the "we can't take your game back" claim is checkable rather than asserted.

Then Stage 3, the purchase path. A real buyer paid through the full x402 flow: 402 challenge with Blocky402's live fee payer, a signed Hedera transfer, verify and settle through the facilitator, `200` with a playable URL. Confirmed on the Mirror Node that the money moved, the GameKey NFT reached the buyer, and the sale is on the HCS topic.

Two things that cost time and are worth knowing: `payTo` was still the `0.0.xxxxx` placeholder from `.env.example` (the env schema now rejects that at boot), and a test buyer created with `AccountCreateTransaction` couldn't receive the NFT at all — those accounts get 0 auto-associations. Buyers have to be alias-created, which is what a real Privy wallet is.

**Needs from you:** nothing blocking.

**Next:** Stage 5, the wishlist agent.

### 2026-09-06 · Backend · Priyanshu

**Shipped:** Stage 4 (a real `sales` table + `npm run splits:retry` — a failed split distribution used to just log an error and vanish; now it's recorded and retryable). Then Stage 5, the wishlist agent, fully working: its own Privy wallet, an HCS-14 identity anchor naming the buyer as funding principal, and a watcher that polls the public listings topic through the Mirror Node and fires the same purchase path a person's checkout does.

**Changes the contract:** `POST /api/games/:id/pay` — the helper I told you to ask for. Signs and settles a purchase with the logged-in buyer's own wallet server-side, since the browser can't hold a signing key. Same shape as the `200` from `/download`.

Three real bugs, all caught by testing the actual path rather than trusting a type or a doc: Privy's `Wallet.public_key` comes back empty in practice (fixed by deriving it from a real signature — its `v` byte makes the key mathematically recoverable, verified across four trials against real wallet addresses); the Mirror Node flat-out rejects `sequencenumber=gt:0` (`gte:1` works, `gt:0` is a `400`) — meaning every new agent silently stalled on its very first tick before this was found; and `@hashgraphonline/standards-sdk`'s install never finished in 4+ minutes, so HCS-14's AID format is hand-implemented directly from its own published spec instead (SHA-384 + Base58 over six canonical fields — the format explicitly allows offline derivation, no SDK needed).

**Tested for real, verified on the Mirror Node, not from our own status field:** created a game as a draft, created an agent watching it, funded the agent with real HBAR, watched it anchor its identity and move to `watching` on its own — then published the game for the first time *after* the agent was already watching. Within one tick it moved `watching → fired` with no script telling it to buy. Confirmed after: its balance dropped by exactly the game's price, its account holds the GameKey NFT, and the split distributed for real.

**Needs from you:** nothing blocking.

**Next:** deployment, so integration doesn't need a local backend on your machine. Otherwise Stages 6–8 (reviews' live ownership check, ENS subnames, moderation actions) — independent of each other, pick any order.

### 2026-09-06 · Backend · Priyanshu (2)

**Shipped:** Stage 6, reviews. The ownership gate and edit flow were already correct from Stage 1 — what was missing was `author`/`authorIsEns` on each review in the list, which the frontend type needs and the API never sent. Fixed with a truncated-address fallback; real ENS names wait for Stage 7.

**Tested for real:** minted a GameKey to a real account, confirmed `ownsGame()` says yes for the actual holder and no for an address that was never funded — this exact function gates every review post and hadn't been called directly in a test before, only indirectly through routes that reject bad tokens. Posted, listed, and edited a review against that real ownership state.

**Needs from you:** nothing blocking. Before Stage 7 (ENS) starts: a Sepolia RPC endpoint (a free Alchemy/Infura key works) and a funded Sepolia wallet — I'll confirm exactly what's needed when I get there.

**Next:** Stage 7 or 8, independent of each other.

### 2026-09-06 · Backend · Priyanshu (3)

**Shipped:** Stage 7, ENS on Sepolia. A subregistry we own outright (deployed via `VerifiableFactory`), the parent name `cgs-sanctuary.eth` registered through the real commit-reveal flow (paid in self-minted test USDC, not ETH), and studio subname minting with a limited role set — a studio can point its own name and renew it, not unregister or transfer it away from platform control.

Two Sepolia sources for contract addresses (docs.ens.domains vs. a pinned repo commit) flatly disagreed — both had real bytecode. Resolved by calling `isAvailable`/`getRegisterPrice` directly and trusting whichever address actually returned a sane answer, not whichever source looked more official. Details in the private stage doc.

**Tested for real:** every step ran as a real Sepolia transaction — subregistry deployed, name registered (confirmed `isAvailable` flipped to `false` afterward), a studio subname minted to a real address with a checked `status: 0x1` receipt. Whole flow cost under 0.0008 ETH.

**Needs from you:** nothing blocking.

**Next:** Stage 8, moderation — the last of the numbered stages.

### 2026-09-06 · Backend · Priyanshu (4)

**Shipped:** Stage 8, moderation — the last numbered stage. `POST /api/reports` already did the immediate delist; what was missing was the human-review half: a report can resolve as a false alarm (restores the game to `published`), a confirmed delist (no change), or `removed_from_storage` (unpins the game's build/cover/media from IPFS for real, sets `status = removed` — the one case that actually ends an owner's access). No admin route exists for this and none was needed — a person on the team resolves a report via `scripts/resolve-report.ts`, same shape as the existing `splits:retry` script.

Also closed a real gap from Stage 7: the deployed subregistry's address only existed in test output, so nothing running could actually mint a studio subname. `POST /api/studios` now mints a real ENS subname before inserting the studio row, and the availability check is a live simulated call against the subregistry, not the old DB-only stand-in.

**Tested for real:** a synthetic game went through all three report resolutions with real Pinata pins — confirmed the unpinned CIDs are genuinely gone by querying Pinata directly afterward, not just trusting the function returned. Also caught and fixed a real bug along the way: `db/client.ts` depended on import order to have `dotenv` loaded already, which broke the moment a standalone script imported it first (`SASL: client password must be a string` — a confusing error for a missing env var). Fixed at the source.

**Needs from you:** nothing blocking. All 8 stages are done — next is deployment, so integration doesn't need a local backend on your machine.

**Next:** deployment.

### 2026-09-06 · Backend · Priyanshu (5)

**Shipped:** Stage 9, not one of the original 8 but a real gap and a deliberate reversal. The gap: login, state, and studio creation were already fully built, but nothing ever told the frontend who it was or what it owned in bulk. `GET /api/me` (identity, live wallet balance, which studio you own or belong to) and `GET /api/me/library` (every game you actually hold a key for, checked live against the Mirror Node in one call rather than one per game) close that. `resolveHederaAccount` — documented since Stage 1, never once called — is now real and caches `hedera_account_id` on first resolution. `plays` and `likeCount` have been in the API shape and in your own `types.ts` since Stage 1; nothing ever computed them until now, batched the same way `rating` already is.

The reversal: likes, comments, and timed play sessions, all cut in the original brief for time reasons that don't apply any more. Likes are a toggle with a real DB unique constraint, not just an app-level check. Comments are unrestricted (no ownership gate, no rating) unlike reviews, on purpose — that's the whole reason both exist. Play sessions start when a boot actually happens (re-checked server-side against ownership, the same branch logic `/download` already uses) and end on an explicit call; a session that never gets one still counts once toward `plays`, it just earns no duration. `GET /api/me/library` sums a user's own sessions per game into `myPlayCount`/`myPlaytimeSeconds` — the concrete answer to "how much have I played this."

**Changes the contract:** six new endpoints, listed above. `Game` gains real `plays`/`likeCount` and (signed in) `liked`. `INTEGRATION.md` §6 has the full shapes and the one thing that matters for your code: call `POST .../sessions` right when the player actually boots, not before, and don't worry about the end call always firing.

**Tested for real:** a genuine fresh EVM address, funded via a real Hedera transfer to trigger auto-account-creation, holding a real GameKey minted through the actual production `fulfilPurchase` path — then `resolveHederaAccount` and the new bulk NFT lookup were both confirmed against that real account and a real Mirror Node query, not a fixture. Play session duration, the plays/likes/comments CRUD, and the DB-level unique constraint on likes were all exercised against live Neon with real inserts, edits, and a genuine duplicate-key rejection. Every new `requireAuth` route correctly 401s with no or a garbage token, same as every prior stage's own test log.

**Needs from you:** nothing blocking.

**Next:** deployment.

### 2026-09-06 · Frontend · Suparno

**Shipped:** integration started, run as one workflow at a time — build it, test it end to end, then the next. Three are done: **browse signed out** (catalog, filters, sort, search, listings, reviews, studio pages), **sign in** (Privy, `GET /api/me`, header balance, sign out), and **library plus notifications**. `src/mocks/session.ts` is gone; `mocks/games.ts` still backs the screens whose turn hasn't come.

Server-side changes for those three live on a **`frontend-integration` branch on CGS-server**. Everything on it is additive — no existing response changed shape, and the one migration only relaxes a constraint — so it should merge without a conversation. Branch off it rather than `main` if you touch these files.

**Changes the contract:**

- **`priceUsd` and `priceAssetDecimals` on every serialized game**, and `balanceUsd`/`balanceAssetDecimals` on `/api/me`. INTEGRATION.md §7 has always promised prices arrive twice; `serializeGame` only ever sent units, and the decimals needed to derive the float were server-only config that §7 also says never to hardcode. Arithmetic still runs on the integer everywhere.
- **`studio.ens` is now the full name** (`tinroof.cgs-sanctuary.eth`), not the bare label. Clients would otherwise have to be told the parent name separately. `coverUrl` and each media item's `url` come as gateway URLs beside the CIDs for the same reason.
- **The embedded studio gains `memberCount` and `ownerAddress`**, both shown on every listing and neither on the studios row. Batched: two queries per page, not two per game.
- **`GET /api/studios/:idOrSlug`** gains `ens`, `ownerAddress`, `memberCount`, and member `id`s. Member **`email` only when the caller owns the studio** — that page is public and was about to leak them.
- **`GET /api/games` takes `studioId`.** The studio page shows full cards and the studio route returns only enough of each game to identify one.
- **`sort=rating` actually sorts by rating.** It silently fell back to newest, which on screen is a filter chip that does nothing. Unrated games sort last rather than tying at zero. `search` now covers tagline, tags and studio name; title-only missed a genre typed into the box, which is the obvious case.
- **`POST /api/studios` inserts the owner into `studio_members`** with a handle (optional `handle` in the body, else the part before the @), and returns it. Without that row a new studio reported zero people on every listing, and the owner had no handle to put on their own game's splits — the one person on the team the credits couldn't name. `/api/me`'s `studio` carries `handle` now.
- **Notification payloads carry facts, not prose.** Every one gains `slug` so a row can link; invites carry `studioName`/`studioSlug`; `agent_fired` carries `title`, `priceUnits` and `triggerPriceUnits`, since it names a game the buyer never opened. Money in a payload gets a `*Usd` beside the units, added on read so existing rows get it too. **A `sale` now tells each person what they earned** (`sharePct`/`shareUnits`, null if they're not on the splits) instead of sending the full sale price to everyone — the row's "your share is already in your wallet" was wrong for every collaborator on a split.
- **`POST /api/notifications/read-all`.** One gesture was thirty POSTs against a 200-per-15-minutes limiter.
- **`POST /api/dev/faucet`**, dev only. Not mounted unless `DEV_FAUCET=on`, and the env schema refuses to boot with it on under `NODE_ENV=production`. It exists because a Privy embedded wallet has no Hedera account until it first receives value and nothing hands out testnet USDC to an address, so a new buyer could never buy anything. Sends HBAR first (that is what creates the account), then the asset, from the operator.
- **Migration 0004: `users.public_key_hex` is nullable.** See below.

**Four things that were broken, found by wiring a browser to them:**

1. **`GET /api/games/:idOrSlug` and `GET /api/studios/:idOrSlug` returned 500 for every slug.** `or(eq(games.id, idOrSlug), …)` makes Postgres cast the parameter to uuid and throw `invalid input syntax for type uuid`. Since every listing URL in the client is a slug, the entire listing route was unreachable. Fixed with an `isUuid` guard so the id branch is only included when the value could be one.

2. **Deriving a user's public key at sign-in cannot work for a real buyer.** `requireAuth` → `upsertUser` → `derivePublicKeyHex` asks Privy to **sign with the user's wallet**, during login, purely to cache a key. Privy refuses with `No valid authorization keys or user signing keys available`: an app can sign freely with wallets **it** created (`privy.wallets().create()` — the agent path, which is why every test passed) but a user's embedded wallet is made in the browser and owned by the user. So every authenticated request failed for anyone who signed in through the UI. It is now derived lazily on first use and cached, which is right regardless: logging in does not need a signature, and making it need one meant a user who hadn't delegated couldn't even browse signed in, let alone reach a screen that offers delegation. `POST /:id/pay` calls `ensureUserPublicKey` and returns `WALLET_NOT_DELEGATED` rather than surfacing a raw Privy error on the screen where money is about to move.

3. **`PRIVY_VERIFICATION_KEY` had to be pasted in exactly one shape or nothing authenticated worked.** `jose` needs real PEM, and a key in a `.env` almost never survives as one. It now accepts full PEM, PEM flattened with literal `\n` (dotenv only expands those inside double quotes), or the bare base64 the dashboard shows, and **parses it at boot with node's own crypto** — so a bad key is one clear line at startup instead of `"spki" must be SPKI formatted string` buried in a 401 on every request.

4. **`requireAuth` threw away every reason.** Missing header, unverifiable token, and a Privy account with no embedded wallet all became the same "Sign in required", and those are three different fixes. It now says which.

Also: `scripts/seed-dev.ts` (`npm run seed:dev` / `seed:wipe`) — the shared database was completely empty, so there was nothing to build browse screens against. Six studios and twelve games, deliberately **with no build and no HTS token**: pinning and minting cost real testnet resources, and the first genuinely purchasable game should come out of the publish flow rather than a script that fakes its way past it. Everything it writes belongs to one placeholder user so the wipe removes exactly that.

**Next:** studio creation and ENS, then publish (which is where splits-without-a-wallet has to be solved), then checkout, then the agent.

### 2026-09-06 · Frontend · Suparno (2)

**Shipped:** three more workflows, same one-at-a-time rule. **Make a studio** (real ENS subname minted on Sepolia, skipping the name is a first-class path), **publish a game** (build pinned to IPFS, token minted, splits locked, and a collaborator can be named by email alone), and **buy it and play it** — the critical path, end to end, with real money moving through x402 on Hedera. Six workflows done. Reviews, reports, the agent and the invite screen are still on mocks.

Still on the **`frontend-integration` branch on CGS-server**. Everything below is on it.

**Changes the contract:**

- **`POST /api/games/:id/pay` is gone.** It could never have worked and is replaced by two calls: `POST /api/games/:id/pay/prepare` builds and freezes the Hedera transfer and returns `{ status: "prepared", intentId, hashes, expiresAt, amountUnits, asset }`, and `POST /api/games/:id/pay/complete` takes `{ intentId, signatures: [{ hash, signature }] }`, attaches them and settles. Prepare answers `{ status: "granted", …grant }` instead when the game is free or already owned. Why it had to split is the first finding below.
- **`GET /api/games/:id/download` no longer returns `playUrl`.** It returns `buildPath` (a relative, ownership-checked URL on this API) and `buildCid` (what the build is on IPFS). `keyStatus` is unchanged. The second finding below is why.
- **`GET /api/games/:id/build.zip`** — new. One ownership-checked request for the whole build. Per-file auth was never affordable: a Godot build makes dozens of requests and each one would have been a Mirror Node call.
- **A split can name someone who has no wallet.** `POST /api/games` accepts each split as `wallet`, `studioMemberId` **or** `email`. An emailed person's `studio_members` row is created by the upload and *that row is the invite* — there is no separate call — and the response carries `invited: [{ id, email, handle }]` so the publish screen can show the link. Migration **0005**: `pending_payouts`, `splits.wallet` nullable, `splits.studio_member_id`, `split_status` gains `partial`.
- **One unpayable recipient no longer fails everybody's payout.** Amounts are worked out across every share first, so the maths doesn't depend on who happens to have an account; everyone payable is paid in one transaction and the rest is written to `pending_payouts`, with the sale reading `partial`. Accepting an invite backfills the wallet onto every split naming that member and settles what was held.
- **Entitlement is the chain *or* a purchase record.** `services/games/entitlement.ts`. Between settlement and the GameKey landing, the buyer has paid real money and holds no key; gating play on the Mirror Node alone locked them out of the one moment this product is about. `ownsGame` is unchanged and still backs `/owned` and the review gate, because a verified-purchase badge is a claim made to other people.
- **`fulfilPurchase` now awaits the sale and key rows** before returning, and backgrounds only the chain work. It was fired with `void`, so `/download` could respond before the purchase existed — and the client asks for the build within milliseconds of that response.
- **`POST /api/studios`** answers `STUDIO_EXISTS` (409) rather than a constraint error, and `ENS_MINT_FAILED` (502) with `ensTxHash` when the name is taken on-chain after passing the DB check. `GET /api/studios/ens-availability` returns `fullName` alongside the label.
- **`PINATA_GATEWAY`** in env, and **`storage/`** on disk. Deployment note: builds are now kept as files, so **the server needs a persistent volume**, not an ephemeral filesystem. `deleteBuild` runs on `removed_from_storage` alongside the unpin, or unpinning would be theatre.

**Three findings, and the first two changed the design:**

1. **The server can never sign with a buyer's own wallet, and it does not need to.** This was the open blocker from the last entry. Both documented answers — the buyer delegates their wallet, or the app holds an authorization key over it — grant *standing* permission to move someone's money, which is a much larger thing to ask than one purchase and would have put a wallet-delegation modal in front of every first buy. The browser could always sign: `secp256k1_sign` is a supported method on Privy's own embedded-wallet provider (it is the call Privy makes internally to sign an EIP-7702 authorization) and it signs a raw hash with no Ethereum prefix, which is exactly Hedera's format. **This contradicts the "Payment signing → backend" decision and the INTEGRATION.md line saying the browser SDK doesn't expose raw-hash signing.** So the purchase splits where the authority actually does: the server builds and freezes the transfer, because that needs the 402 terms and a Hedera client; the browser signs the body hashes; the server settles. Nothing is delegated and nothing is held. **The agent is untouched** — its wallet is one we created, so `payForGame()` still signs in one step, and both paths build the identical transaction from one function.

   Two details worth having. A frozen transaction carries one body per node it may be submitted to, each needing its own signature; testnet picked seven, so `setMaxNodesPerTransaction(3)` cuts the browser round trips by more than half while keeping failover. And signatures are matched to bodies **by hash**, not by position, so the order they come back in cannot corrupt a payment.

2. **Every new buyer's account has no public key at all.** First attempt read the key off the Mirror Node and refused every real wallet. The account holding $10 of test USDC reports `key: null`, and its `alias` decodes to 20 bytes — the EVM address, not a 33-byte key. It is a **hollow account (HIP-583)**: an account created by *receiving* value at an address that has never signed anything holds the money and publishes no key until the first time it signs. That is exactly the account a first-time buyer has, at exactly the moment they try to buy, and the faucet is what creates it that way. So nothing asks for the key up front. It is recovered from the payment signature and verified against the address being charged — which also proves the signer holds that wallet — and signing the payment is what completes the account. Both recovery ids are tried and the one deriving the right address wins, because Privy documents neither the 0/1 nor the 27/28 encoding of the v byte. Checked over 60 generated keys with a wrong-key signature correctly refused. `ensureUserPublicKey` is deleted; `users.public_key_hex` stays for the agent's wallets, which we create and can sign with.

3. **IPFS cannot serve a build to a browser at all.** Not slowly — at all. Pinata answers **403 `ERR_ID:00023`** for HTML through any `*.mypinata.cloud` gateway: the whole directory CID, not just `index.html`, and for authenticated reads as well as anonymous ones. `?download=true`, `?format=raw`, `?format=car` and the SDK with the JWT were all refused; its own advice is to add a custom domain to the gateway. And no public gateway substitutes — `ipfs.io` and `dweb.link` both time out on this account's CIDs, because Pinata does not announce freshly pinned content to the DHT quickly. All of that was checked against the real pinned build, not assumed. **This retires the "`playUrl` is a direct `ipfs.io` URL, so the second origin isn't needed for purchased builds" line under The contract** — both halves of it were wrong. Delivery and provenance are separate now: IPFS still proves what a build *is* and the CID still goes on-chain, but the server serves it, and the client unpacks the zip onto its own isolated build origin using the same pipeline the publish preview already used. Cover art was never affected; images are not blocked. The way back is a custom domain on the Pinata gateway, which is one function's worth of change and nothing on the client would move.

**Needs from you:** nothing blocking. Two things worth knowing: the operator is still near $10 of testnet USDC and now pays the faucet *and* every split, and if you have a domain to point at the Pinata gateway, that turns finding 3 back into a one-line config change.

**Next:** the invite screen and the held-payout settlement it triggers, then reviews, likes and reports, then the agent.
