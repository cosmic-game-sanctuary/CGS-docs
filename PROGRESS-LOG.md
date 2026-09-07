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

**Stage:** six workflows plus a catch-up pass that closed most of the gap Stages 10–16 opened. Tested end to end against the live API on 2026-09-07; everything below was exercised in a browser, not assumed.
**Real, not mocked, now:** the whole buyer path and the whole dev path, plus **reviews, reports, wishlists, profiles, receipts, managing a published game, and studio roster management**. A dev can now edit a live listing, reprice it, ship a patch every owner receives, reorder screenshots and unlist or relist. A buyer can save games, see a price drop land, review, and read a receipt that names its settlement transaction.
**Still on mocks:** the invite screen and the agent. Comments have no UI, deliberately.
**Untested:** cloud saves. The plumbing is built on both sides, but the only build we have does not write to storage, so nothing has been proved either way.
**Deployed:** no. The persistent-volume requirement is gone — builds are pinned as a zip and refetched when missing, which we round-tripped for real (see below).
**Next:** the invite screen and the payouts it settles, then the agent.
**Note for Priyanshu:** the `frontend-integration` branch is merged; everything is on `main` now. One small change of mine is in **your** file — see the entry below.

### Backend · CGS-server

**Stage:** All 8 numbered stages done, Stage 9 (profile plumbing, library, likes, comments, playtime) on top of those, and Stages 10–16 close out almost everything the product-gap review found. Built and verified on live Neon, Hedera testnet, Blocky402, Sepolia, and Pinata. No route returns `501`.
**Working end to end:** a real buyer pays through x402 and the GameKey lands in their account. An agent with its own wallet and its own on-chain identity watches the public listings topic and buys with no human present. A subregistry we own on Sepolia, `cgs-sanctuary.eth` registered under it, studio subnames minted for real on studio creation. A moderation report immediately delists, and a human resolution can restore it, confirm it, or genuinely unpin it from IPFS. `GET /api/me` and `GET /api/me/library` answer "who am I" and "what do I own" for real against the Mirror Node.
**New since Stage 9 (10–16), all tested against real infra and documented in INTEGRATION.md:**
- **A game can be edited after publishing** — price, description, cover, tags — and **shipped a new build**, a real version history rather than a second listing. Price changes go on the public HCS topic.
- **Everyone has a handle and a public profile** — reviews, comments and game credits link to a real person, not a truncated address.
- **A real wishlist**, not just an agent: price-drop notifications, and a public demand count on HCS at milestones — nobody else exposes that.
- **Cloud saves** for browser games — 3 slots, checksums, version-conflict detection.
- **Studio management** — remove/leave/promote/transfer a member, kept strictly separate from the permanent credit ledger so nobody's payout or credit is ever touched by an org-chart change.
- **Developer replies on reviews**, delete for reviews/comments, and **reports on reviews/comments** — deliberately no auto-hide (unlike a game report), and reporters now learn the outcome either way.
**Also real:** wallet balances including HBAR, withdrawals back out to any Hedera account or EVM address, earnings for a studio and for an individual across every studio they're on, invite emails, held payouts that settle themselves, timed play sessions.
**Deployed:** no.
**Blocked on:** no CSAM-scanning provider chosen — every upload fails closed with `MODERATION_BLOCKED` until one is. Deliberate, not a bug. Email can only reach one address until a domain is verified (see Blockers).
**Next:** not engineering. Deploy is the biggest gap and nothing blocks it. The agent is an open design question rather than a build task — four designs were considered and rejected, so it is deliberately unscheduled. Devlogs/following and curated browsing stay open on purpose.

---

## Blockers

Only things stopping work right now.

| Who | Blocked on | Since | Needs |
|---|---|---|---|
| Priyanshu | No CSAM-scanning provider chosen | 2026-09-05 | A vendor decision — Cloudflare's CSAM Scanning Tool, PhotoDNA Cloud, Thorn Safer, or Hive Moderation. See `docs/stage-2.md` §2. |
| Both | The operator holds ~$10 of testnet USDC | 2026-09-06 | Top-ups from faucet.circle.com to `0.0.10375438`. It funds every test wallet **and** pays every split, so checkout testing drains it from both ends. |
| Both | Email only reaches one address | 2026-09-07 | A verified domain. Without one Resend sends from `onboarding@resend.dev` and delivers **only to the address the Resend account was registered with** — so an invite to a teammate is refused and logged, not delivered. A domain is being bought; once its DNS records are in, `RESEND_FROM` changes and nothing else does. |

_Cleared: server-side signing on a user's Privy wallet. It was never the right question — the browser signs now, and nothing is delegated. See the 2026-09-06 (2) frontend entry._

_Cleared 2026-09-07: the three builds with no retrievable copy. `builds:backfill` has run and all three are pinned. Nothing needs delisting — see the frontend entry below for why one that looked unrecoverable was not._

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
| Unknown notification types | Skipped, never rendered | The enum grows on the backend and the client learns about it later. A `switch` with no default returned nothing and put a hole in the list the panel iterates, so one unrecognised row killed the bell. Skipping is the only behaviour that stays correct while the other side keeps adding. |
| The cloud-save bridge | The host frame reads and writes the build's storage; no script is injected into the game | INTEGRATION.md §13 suggests injecting a script into the build's frame. It turned out not to be needed: `preview-host.html` is already same-origin with the build, so it is the one page living on both sides of the isolation boundary. The whole origin is snapshotted rather than a namespaced subset, because the keys belong to the game and the payload is meant to be opaque. |
| Where managing a game lives | `/game/:slug/manage`, reached from the listing | Not behind the profile menu. Looking after a game is something you do to a specific game, not a place you go, and the IA rule is two pages, one action, wallet inline. |
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

### 2026-09-07 · Both · Priyanshu

**Shipped:** the wallet address is on screen, so funding no longer depends on the faucet. Someone with their own testnet funds can send straight to their Privy wallet from HashPack or anything else. Two bugs found by running the stack from a second machine, and one architectural consequence that has to be settled before deployment.

**Changes the contract:** nothing. `/api/me` already returned `evmAddress`; the menu just never showed it.

**Three fixes:**

1. **A first-ever sign-in could land in a permanently broken session.** Privy creates the embedded wallet as part of logging in, and for a moment afterwards its own API still reports the account without one. The server reads that and correctly answers "no embedded wallet" — and the client cached that answer, because it only re-read `/api/me` on funding, a purchase, or a studio creation, none of which are reachable from that state. Only a reload fixed it. `/api/me` is now retried four times over about four seconds. Confirmed it was a race and not a real absence, by resolving the same account server-side afterwards.

2. **`gatewayUrl()` fell back to `ipfs.io`, which does not work for our content at all.** A cover image and a build directory both time out there, because Pinata doesn't announce freshly pinned content to the DHT quickly. `gateway.pinata.cloud` serves it, verified `200 image/png` on the exact cover CID that was failing. That is the default now; `PINATA_GATEWAY` overrides it. **This narrows the earlier finding rather than contradicting it:** Pinata's public gateway refuses **HTML specifically**, not everything. Checked file by file inside a real pinned build — `index.wasm` returns `200`, `index.html` returns `403 ERR_ID:00023`. A directory CID resolves to index.html, so it 403s too. Images were never the problem.

3. **The missing-build 404 read as a mangled sentence.** `Errors.notFound()` appends `" not found."`, and it was being handed a whole explanation. Message and reason are separated now.

**Needs from you:** a decision, and it blocks deployment.

**`storage/builds/<gameId>.zip` lives on the filesystem of whichever server pinned it, and the database is shared while the filesystem is not.** A game published on one laptop 404s on the other — which is exactly what happened, and why one of us could play `deadzone` and the other could not. The same thing breaks a deploy on an ephemeral filesystem: every build vanishes on restart. It cannot be recovered from IPFS either, because the entry file is HTML and the public gateway refuses it. So it needs a persistent volume or shared object storage, or a dedicated Pinata gateway on a custom domain, and that is a call for both of us rather than a code fix.

**Also worth knowing:** a buyer does **not** need HBAR to pay. Blocky402's fee payer covers the network fee, so USDC alone is enough for the purchase. What HBAR is needed for is the very first transfer into a brand new wallet, because receiving value is what creates the Hedera account. A buyer who has nothing at all still cannot start unaided — Privy's onramp is the answer to that, and it is one of their qualification requirements.

**Next:** the profile page's server side, withdrawing funds back out of a Privy wallet, and wiring notifications through to the bell.

### 2026-09-07 · Backend · Priyanshu (2)

**Shipped:** the server side of a wallet page, withdrawals, and the emails that were never being sent. Also fixed the build problem from the previous entry, which turned out to be solvable for free.

**Changes the contract:**

- **`GET /api/me` gains `hbarUnits` and `hbar`.** Reported separately from the settlement asset because HBAR is not spending money here: the facilitator covers the fee on a purchase and the operator covers it on a withdrawal. A wallet with 0 USDC and some HBAR is funded with nothing to spend, and that read identically to an empty wallet before.
- **`POST /api/me/withdraw/prepare` + `/complete`.** Same two-step shape as a purchase, since the server still cannot sign for a user's wallet. `to` takes either a `0.0.x` or an EVM address. Omit `amountUnits` to send everything. **The operator pays the network fee**, so a wallet holding only USDC is not a wallet you cannot empty. Tested on testnet: full balance out, sender's HBAR untouched.
- **Invites are actually delivered.** `POST /api/studios/:id/members`, and naming someone by `email` on a split, both already created the membership row that *is* the invite, and nothing ever told that person. They have no account, so no notification row could reach them and mail was the only channel. Request and response shapes are unchanged.
- **`games.build_zip_cid`** (migration 0006). Not client-facing.

**The build problem is fixed, and cheaply.** The earlier conclusion was that IPFS could not serve a build. That is narrower than it looked: Pinata refuses **HTML content**, not the build. Checked file by file inside a real pinned build — `index.wasm` is `200`, `index.html` is `403`, and a directory CID resolves to index.html so it 403s too. **A zip is `application/zip` and serves normally.** So every build is now pinned twice: `build_cid` (the directory) stays the provenance answer, and `build_zip_cid` (the original zip) is the delivery answer. `findBuild()` treats disk as a cache and refetches a missing build. Round-tripped for real: pinned, deleted the local copy, got a byte-identical file back.

**Needs from you:** `npm run builds:backfill` **on your machine**. The three existing `deadzone` games predate this, so their zips only exist where they were published. The script pins what it can reach and names what it cannot rather than skipping quietly. Republishing also works.

**Worth knowing about email:** with no verified domain Resend only delivers to the address the Resend account itself was registered with. Sends to anyone else are refused and logged with the recipient. That is configuration, not a bug, and nothing on the client changes once a domain is verified. A send never fails the request that caused it.

**Also:** `INTEGRATION.md` §3 and §4 were still describing `POST /:id/pay` and a `playUrl` from `/download`, both of which you replaced. Corrected to prepare/complete and `buildPath`, including the reversal about the second origin now being needed for purchased builds too.

**Next:** the wishlist agent — the price-drop endpoint it needs to fire at all, and the notifications around it.

### 2026-09-07 · Backend · Priyanshu (3)

**Shipped:** earnings, and the payout hole behind them.

**Changes the contract:**

- **`GET /api/me/earnings`** — what you've earned, across **every** studio you're on. Cross-studio because someone added by email is often credited on games from several teams, and "your studio's earnings" would hide money from exactly the people splits exist for. Totals, per-game breakdown with your percentage and share, what's held and why.
- **`GET /api/studios/:id/earnings`** — owner and accepted members. Per game and per person, plus who's unpaid and how much is waiting on them.
- **`/api/me` gains `studios`**, every studio you own or joined. `studio` stays as the primary, so nothing moves.
- **Studio members now see their team's drafts.** Ownership was the wrong line — a collaborator credited on a game couldn't see the game they helped make. Member emails stay owner-only.
- **`payout_held` and `payout_settled`** notification types.

**The real fix underneath:** held payouts only moved at invite-accept, and only if a Hedera account already existed — which a new user's doesn't. So money owed sat until someone ran `splits:retry` by hand, and nothing told anyone it was there. It now settles the moment `/api/me` first resolves an account, on a request we were already serving. Opening the site is what releases it, so there is no "claim" button to build.

Both sides are told now, too: a notification and an email when a share is held, and when it lands. The person owed usually has no account, so mail was the only channel that could reach them at all.

**Tested end to end on testnet:** a 60/40 split with an unclaimed collaborator held 8000 units across two sales. Both reports agreed on every figure, the artist's own report showed what was owed before they could receive it, and after their account appeared the money **actually landed on chain** and both sales flipped `partial` to `distributed`.

**Next:** the wishlist agent.

### 2026-09-07 · Backend · Priyanshu (4)

**Shipped:** the studio and payout side, reviewed end to end and then filled in. Details of each piece are in the three entries above; this is what changed as a result of reading the whole thing at once.

**Changes the contract:** nothing beyond what entries (2) and (3) already listed. `INTEGRATION.md` is current — §3, §4, §6.1 and §6.2 all match the code now, and the stale `POST /:id/pay` and `playUrl` descriptions are gone.

**Two corrections worth flagging, both mine:**

- **Studio members can see their team's drafts.** I had gated drafts on ownership when I stopped them leaking publicly, which put a collaborator credited on a game on the wrong side of the line. Membership is the test now; member email addresses are still owner-only.
- **`/api/me` returns every studio you're on.** It was `findFirst` on both the owned studio and the membership, so someone on two teams saw one of them arbitrarily. Inviting collaborators by email is exactly what produces that situation.

**New scripts:** `npm run game:delist -- <slug>` takes a listing out of the catalog **without deleting it** — a published game can have real sales and real minted keys behind it, and delisting never revokes anyone's copy. `npm run builds:backfill` pins the zip for games that predate `build_zip_cid`.

**Where email stands.** It works and is tested, but with no verified domain Resend only delivers to the address the account itself was registered with. Sends to anyone else are refused and logged with the recipient, and a failed send never fails the request that caused it. A domain is being bought; when its DNS records are verified, `RESEND_FROM` changes and no other code moves. Until then, treat every invite as "the row was created" rather than "the person was told".

**Also corrected the docs:** both `CLAUDE.md` files listed `@hashgraphonline/standards-sdk` in the stack. It was never installed — HCS-14 is hand-implemented, because the install never finished and the spec allows offline derivation. The `ipfs.io` references went with it.

**Next:** the wishlist agent. It's the last unbuilt piece and the one the Hedera track is about.

### 2026-09-07 · Backend · Priyanshu (5)

**Shipped:** a memo on withdrawals, and one finding that changes the funding story.

**A new wallet does not need HBAR.** The assumption baked into the faucet and the funding copy was that HBAR has to arrive first because it is what opens the Hedera account. That is wrong: **HIP-542 charges the account-creation fee to the sender rather than deducting it from what is sent**, so a token transfer creates the account by itself. Verified on testnet by sending only USDC to an untouched address — the account came into existence holding the USDC and **zero HBAR**.

So the funding hint is now just "send USDC here", and nobody has to go and find HBAR before they can start. Confirmed independently from a real wallet as well.

**Worth knowing before pointing anyone at a funding route:** not every wallet can send to an EVM address. HashPack can. **Circle's testnet faucet cannot** — it requires a `0.0.x`, so it is only usable once an account already exists. Exchanges generally reject EVM addresses too. After the first transfer the account has a `0.0.x` and none of that matters any more.

**Changes the contract:** `POST /api/me/withdraw/prepare` takes an optional `memo`. Exchange deposit addresses are pooled accounts that identify the depositor by memo, the same as an XRP tag, so a withdrawal to one without a memo is credited to nobody. Any off-ramp UI has to offer the field.

**Also:** schema for the wishlist agent landed (migration 0008 — new statuses, a shared listener cursor, two indexes) but **no agent code was written and nothing reads those columns**. The existing watcher is untouched and behaves exactly as before. The agent is on hold pending a design conversation.

**One thing that has no answer yet:** there is still no way for a developer to edit, reprice, unpublish or delist their own game. The only writes to a game's status anywhere are moderation and an operator script. That is the gap the agent work would have opened first.

**Next:** agent design discussion.

### 2026-09-07 · Backend · Priyanshu

**Shipped:** a game is no longer frozen once published. `PATCH /api/games/:id`
edits the listing including the price; `POST /api/games/:id/builds` ships a new
version that everyone who already owns the game gets; unlist and relist;
delete a draft; add, reorder and remove screenshots; and `GET
/api/games/:id/manage` returns the whole developer view in one call. Price
changes go on the public HCS topic, which is what finally makes the wishlist
agent able to fire at all. Verified end to end against real Pinata pins, real
HCS writes read back from the Mirror Node, and a byte-for-byte check that the
patched build is the one being served.

**Changes the contract:** INTEGRATION.md §10 is new and covers all of it. Two
things to know without reading it: **the slug never changes when a title does**,
so don't re-route after a rename; and `GET /api/games/:idOrSlug` now also
returns `status`, `buildVersion`, `updatedAt` and `delistedBy`. Nothing was
removed. New error codes: `MODERATION_HOLD`, `GAME_HAS_SALES`, `GAME_IS_WATCHED`.

**Needs from you:** nothing yet. When you get to it, the developer-side screens
this unblocks are: edit a game, upload a patch, put it on sale, take it down.

**Next:** profiles and public identity, then the wishlist.

### 2026-09-07 (2) · Backend · Priyanshu

**Shipped:** everyone has a handle and a public page. `GET /api/users/:handle`
returns their studios, the games they're credited on with their share, their
reviews, their library and their playtime — no email, ever. Profile editing,
avatars and `GET /api/me/purchases` (receipts, each with the settlement
transaction) on the other side. Reviews, comments and game credits now name a
person instead of a truncated address.

**Changes the contract:** INTEGRATION.md §11. Three things worth knowing:
`GET /api/me` gains `handle`, `displayName`, `label`, `avatarUrl` and
`libraryPublic`; reviews/comments gain `authorProfile` and each game split gains
`profile`; and **every `/api/games/:id/…` route now accepts a slug** — several
of them used to return `500` for one, so `/api/games/deadzone/reviews` was a
server error while `/api/games/deadzone` worked. Use `label` for display rather
than reimplementing the name fallback.

**Needs from you:** nothing. The screens this unblocks are a profile page,
avatars, and a receipts list.

**Next:** the wishlist, then cloud saves.

### 2026-09-07 (3) · Backend · Priyanshu

**Shipped:** the wishlist. `likes` is it now — same rows, given a purpose. Each
one remembers the price when it was saved, so the list can say "50% off since
you saved it". Dropping a price notifies and emails everyone waiting, skipping
people who already own the game and people who muted it. Wishlist counts go on
the public HCS topic at milestones, which no other storefront does. Creating an
agent also adds the game to your wishlist, so the agent reads as an upgrade of
the list rather than a separate thing.

**Changes the contract:** INTEGRATION.md §12. **Nothing breaks** — `liked` and
`likeCount` are still on every response and `POST /:id/like` still toggles.
New: `POST`/`DELETE /api/games/:id/wishlist`, `GET /api/me/wishlist`,
`GET /api/games/:id/demand` (public), a `price_drop` notification type, and
`stats.wishlisted` on the manage view.

**Needs from you:** nothing. `GET /api/games/:id/demand` has no UI at all yet
and is the most distinctive thing in this batch — a public, verifiable count of
who is waiting for a game.

**Next:** cloud saves for browser games.

### 2026-09-07 (4) · Backend · Priyanshu

**Shipped:** cloud saves. Three slots per person per game, 512KB each, data
opaque so it works for any engine. Every save carries a checksum and a version;
sending the version you last read turns a blind overwrite into a
`409 SAVE_CONFLICT` that describes both sides.

**Changes the contract:** INTEGRATION.md §13. Also two error responses changed
for the better everywhere, not just here: an oversized body now returns
`413 PAYLOAD_TOO_LARGE` and bad JSON returns `400 MALFORMED_JSON` — **both used
to be `500 INTERNAL`**, including for an oversized build upload. The JSON body
limit is 2mb now (was 1mb).

**Needs from you:** the browser half of this can only be done on your side. The
build runs on an isolated origin so the page can't read its `localStorage`
directly — it needs a small bridge injected into the frame. §13 has the sketch.

**Next:** studio management, review replies, and finishing moderation.

### 2026-09-07 (5) · Backend · Priyanshu

**Shipped:** studio management. Remove a member, change a role, resend a lost
invite, leave a studio, transfer ownership — none of that existed before. The
roster (`studioMembers`) and the credit ledger (`splits`) are now separate on
purpose: removing someone never touches what they earned. If they're credited
on anything they're deactivated, not deleted, and every credit and held payout
survives untouched — checked directly in the test, not assumed.

**Changes the contract:** INTEGRATION.md §14. Five new routes under
`/api/studios`, all documented there. One to flag: **inviting a teammate no
longer requires being the founder** — it's manager-gated now, same as editing a
listing. New error code `IS_FOUNDER` (409) — the founder can't be removed,
demoted, or leave their own studio; they transfer it instead.

**Needs from you:** nothing yet — no frontend screen calls any studio member
route today, so nothing existing changes behavior.

**Next:** review replies (developer voice) and comment deletion, then reports
on reviews/comments and telling a reporter what happened to their report.

### 2026-09-07 (6) · Backend · Priyanshu

**Shipped:** developer replies to reviews (one per review, from the studio,
`POST`/`DELETE /api/reviews/:id/reply`), and delete for both reviews (the
reviewer only) and comments (the author, or a manager of the game's studio —
moderation-lite for a developer's own page).

**Changes the contract:** INTEGRATION.md §15. `developerReply` /
`developerReplyAt` ride along on the existing review shape — no new fetch
needed. New notification type `review_reply`.

**Needs from you:** nothing yet, no frontend screen touches this today.

**Next:** reports on reviews/comments (currently only games can be reported),
and telling a reporter what happened to their report.

### 2026-09-07 (7) · Backend · Priyanshu

**Shipped:** reviews and comments can be reported (`POST
/api/reports/content`), and reporters now learn the outcome — for both this
new path and the existing game-report path. Deliberately **no auto-hide** on a
content report, unlike a game report's immediate delist: hiding a review on
one report would let any developer silence honest criticism of their own game
with a click. It only queues for a human; `npm run reports:content` resolves
one, same shape as the existing `reports:` script.

**Changes the contract:** INTEGRATION.md §16. New notification type
`report_resolved`, fired on **both** report paths now — if your client
already renders `POST /api/reports`'s existing behavior, nothing there
changed; this only adds a notification neither of you had before.

**Needs from you:** nothing yet.

**Next:** this closes out the trust-and-safety and studio-management gaps from
the product review. Remaining open ones are bigger product decisions —
devlogs/following (discovery), collections/curated browsing — worth discussing
before building rather than assuming.

### 2026-09-07 · Frontend · Suparno

**Shipped:** the frontend caught up with Stages 10–16. Wishlists, profiles with avatars and receipts, managing a published game, studio roster management, developer replies, content reports, and the cloud-save bridge. Reviews and reports came off mocks on the way. All of it was then run through in a browser end to end; what that found is at the bottom of this entry.

**Changes the contract:** one line, and it is in your file. `UNIT_FIELDS` in `notification.routes.ts` now also covers `heldUnits`, `amountUnits`, `fromUnits` and `savedAtUnits`, so the payout and price-drop payloads get their `…Usd` companion like every other notification. Without it the client would have had to learn the asset's decimals, which §7 says never to do. Nothing else on the server moved.

**Three things that were broken on our side, worth knowing because two of them are shapes:**

1. **The notification bell crashed on anything new.** `adaptNotification` switched over four types with no default, so the other ten returned `undefined` and the poll hook threw on the first one that arrived. Two `payout_held` rows were already sitting in the database from the deadzone sale, so it was broken in practice, not in theory. All fourteen types have copy now, and — the actual fix — an unknown type is skipped rather than returned as nothing. **Add a fifteenth whenever you like; it cannot take the panel down again.**

2. **A price-history row has no `priceUsd`, and I assumed one.** The rows are `{ fromUnits, toUnits, asset, at, hcsTxId }`. §10 describes the endpoint without listing the row fields, so I typed them from the surrounding prose and `formatPrice(undefined)` took the whole manage screen down — blank on refresh, from the moment a price first changed. Rendering the change rather than a price is better anyway. **Not a bug on your side**, but if §10 ever gains a row shape it would have saved an hour.

3. **Reviews carry a real name now, and something downstream was still truncating it.** `author` used to be an address, so the listing ran `truncateAddress` on it; that quietly mangled any display name over eleven characters into `Priyan…umar`. Anything printing `author` should print it as sent.

**Things your side does that the UI now surfaces, in case the wording matters:** `announced` on a price change is shown as its own sentence, because "I put it on sale" and "the sale is public" are different facts and only the second one an agent can act on. `unsettledSplits` is a yellow panel on the manage screen. `outcome` from removing a member is not shown — deleted and deactivated read the same from the UI's point of view, and the copy says instead that removing somebody never touches what they earned.

**Two asymmetries preserved on purpose,** since they only exist if the UI keeps them: a studio can reply to a review and has no way to delete one, and reporting a review offers no immediate effect the way reporting a game does.

**`builds:backfill` is done, and one conclusion in my last entry was wrong.** I said `deadzone` was unrecoverable because its local zip is gone. It is fine: all three builds are byte-identical, so they content-address to a single CID, and pinning it once made every one of them retrievable. Verified rather than reasoned — fetched the CID (`200 application/zip`, 23,616,945 bytes, 4.2s, sha256 matching both local copies), then called `findBuild` for the game with no local file and watched it pull the build back from IPFS and re-cache it in 4.3s. **That is the publish-here-play-there path running for real**, which is the thing a deploy on an ephemeral filesystem depends on and which nothing had exercised until now. Nothing needs delisting.

**Needs from you:** nothing blocking. Two notes. A first-ever sign-in can still briefly report no embedded wallet — Privy's own API does that for a moment after creating one — and the client now re-reads `/api/me` the instant Privy says the wallet exists rather than guessing with retries, so the message clears itself. And one payment failed once and succeeded on retry with nothing charged; I could not reproduce it and have not guessed at a fix. My suspicion is Mirror Node lag on a wallet funded seconds earlier, where `payerFor` correctly refuses rather than inventing an account.

**Next:** `/invite/:id` against the real endpoints, which is also what settles the held payouts, then the agent.

### 2026-09-08 · Backend · Priyanshu

**Shipped:** an audit pass rather than a feature. Pulled and verified all three
repos against each other, fixed one real inconsistency, and pruned the docs.

**The one bug, and it was mine.** `GET /api/games/:id/price-history` added
`fromUsd`/`toUsd` to every row and `GET /api/games/:id/manage` returned the raw
rows without them — the same data in two shapes depending on which endpoint you
asked. That is what cost Suparno an hour on a blank manage screen. The display
pair is now produced once, in the service, so both endpoints return identical
rows. **INTEGRATION.md §10 now shows the row shape**, which is what he asked
for.

**Verified, not assumed:** every API path the client calls was extracted and
diffed against every route the server serves — 72 routes, all matching. Profile,
demand, builds and price-history payloads were checked field-by-field against
the client's types. Suparno's `UNIT_FIELDS` change is complete: it covers
exactly the money fields the payout and price-drop payloads emit, nothing
missed. Client builds clean, server typechecks clean, every public endpoint
`200`s and every protected one `401`s.

**One thing to fix on the frontend, and it can lose someone's save.** In
`cloudSaves.ts`, `pullSave` returns `{}` on *any* failure, and `pushSave` reads
a missing `baseVersion` as "overwrite, I know". That conflates *no save exists*
(safe to write) with *couldn't read the save* (a blind write destroys a newer
one). Same path if the game closes before the async pull resolves, since
`GameStage` starts with `holder.session = {}`. Distinguishing the two cases
fixes both.

**Also confirmed, not fixed:** the payment that failed once and worked on retry
is Mirror Node lag, exactly as suspected — `payerFor` asks the mirror for an
account that was funded seconds earlier, gets a 404, and returns
`WALLET_NOT_FUNDED`. It fails *safe*, nothing is charged, but the message is
wrong for someone who just funded their wallet. A short retry before concluding
"not funded" is the fix. Deliberately left alone while the payment path is
being tested.

**Docs:** the private folder was pruned — four documents deleted because each
had been overtaken by a real source of truth (endpoints by INTEGRATION.md, the
data model by `schema.ts`, config by `.env.example`, gotchas by the server's
own working rules). Nothing in this repo lost anything; the stage write-ups are
all intact.

**Needs from you:** the cloud-save fix above. Nothing else.

**Next:** deploy is the largest remaining gap and no code blocks it.

### 2026-09-08 (2) · Backend · Priyanshu

**On the `agent` branch of CGS-server, not `main`.** This is the first of four
stages toward reworking the wishlist agent, and none of it is merged yet —
`main` is untouched and still in sync across all three repos.

**Shipped: sales are a real thing.** A promotion is a sale price, a start, an
end, and an **automatic revert** — the developer never has to remember to change
the price back, which is most of why sales barely happened. Scheduled or
immediate, extendable, cancellable, with full history.

Everything routes through the existing `changePrice`, so a sale produces the
same price-history row, the same HCS message and the same `price_drop`
notifications a manual change does. A sale is not a special kind of price, it's
a *scheduled* price.

**Changes the contract:** INTEGRATION.md §17. Two things that will bite
otherwise:

- **`PATCH /api/games/:id` with `priceUnits` now returns `409
  PROMOTION_ACTIVE` while a sale is running.** The sale owns the price until it
  ends; editing underneath it would be silently undone at `endsAt`. The error
  carries `promotionId` and `endsAt` so you can send the person to the sale.
- **`GET /api/games/:idOrSlug` gains `promotion`** — the running sale or `null`.
  Worth a countdown: a discount with a visible deadline is a different thing
  from a cheap game.

New error codes `PROMOTION_EXISTS` and `PROMOTION_ACTIVE`.

**The part that matters beyond sales:** both the start *and* the end of every
sale go on the public HCS topic, and **the message carries `endsAt`**. That's
the piece the whole agent redesign rests on — anything reading the topic can now
tell "the price dropped" from "the price dropped and goes back up on Tuesday",
which is the difference between waiting being a trap and waiting being a bounded
decision.

**Tested:** 30 assertions against the live database and real HCS, all passing —
including two genuinely concurrent activations producing exactly one winner, a
hand-edited price surviving the revert untouched, and the sale-start message
read back off the Mirror Node with its `endsAt` intact.

**Needs from you:** nothing yet — no screen calls these, and nothing existing
changed except the two contract notes above.

**Next:** Stage 18, the agent itself — one agent per person with N wants and a
shared budget, an HCS subscription replacing the per-agent poller, the
deterministic decision path, and the double-buy fix. No model involved yet.
