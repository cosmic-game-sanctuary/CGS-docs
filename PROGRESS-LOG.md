# Progress log

Shared status across the three repos. Two people, two halves, one file.

**Before you start:** read Status and Blockers. That is the whole handover.

**When you stop:** add one entry at the top of the Log, update the half of Status you moved, and add or clear blockers.

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

**Stage:** the whole UI is built and working on mock data. No integration yet, by design.
**Working end to end:** buy (browse, listing, checkout, key, instant play), publish (drop a zip, see it running, details, splits, live in the catalog), get invited (invite by email at publish, accept, land in the studio).
**Real, not faked:** a dropped zip is genuinely unpacked in the browser and plays in the page.
**Deployed:** no.
**Next:** per-screen audit, then Privy, then the API, then the x402 download.

### Backend · CGS-server

**Stage:** Stages 1–3 done. Foundations, the publish pipeline, and the x402 purchase path are all built and tested against live Neon, Hedera testnet and the Blocky402 facilitator. No route returns `501`.
**Working end to end:** a real buyer pays through x402, the payment settles through Blocky402, the GameKey NFT lands in their account, the split goes out atomically, and the sale is logged to HCS. Verified against the Mirror Node at every step, not from SDK responses.
**Deployed:** no.
**Blocked on:** no CSAM-scanning provider chosen — every upload fails closed with `MODERATION_BLOCKED` until one is. Deliberate, not a bug.
**Next:** Stage 5, the wishlist agent (identity anchor + the Mirror Node watcher). Frontend integration can start now — see [INTEGRATION.md](INTEGRATION.md).

---

## Blockers

Only things stopping work right now.

| Who | Blocked on | Since | Needs |
|---|---|---|---|
| Priyanshu | No CSAM-scanning provider chosen | 2026-09-05 | A vendor decision — Cloudflare's CSAM Scanning Tool, PhotoDNA Cloud, Thorn Safer, or Hive Moderation. See `docs/stage-2.md` §2. |

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
| Payment signing | Backend, via Privy's `secp256k1_sign` | Privy's browser SDK doesn't expose raw-hash signing. Same bridge serves the buyer and the agent, so the agent fires the identical path a person does. |
| Frontend integration | Frontend owns it; backend provides the x402 helper | See [INTEGRATION.md](INTEGRATION.md). Privy in the browser is frontend too. |

### Frontend, where it reaches the contract

| Decision | Choice | Why |
|---|---|---|
| Running builds get their own origin | `VITE_PREVIEW_ORIGIN`, a subdomain | The iframe needs `allow-same-origin` or Godot, Unity and Construct all die on boot. On the app's origin that hands a stranger's game the session and the DOM. **This applies to real published builds too, not just local previews.** Same shape as itch.io's `html-classic.itch.zone`. |
| Splits are public on every listing | handle, role and percent | It's the pitch, so the API has to return handles and roles rather than addresses and numbers. |
| The agent lives on the game listing | Not a page of its own | Buying and setting a price trigger are the same decision made two ways. Agents are per game, so several can watch at once. |
| Checkout is an overlay, not a route | | The claim is that the game boots in the same tab. A navigation unmounts the page and breaks exactly what we're claiming, which also means the payment call's latency is visible and budgeted. |
| Whole UI on mock data before any integration | | Lets the design and the flows be validated without waiting on the backend or Privy. Integration is a later, deliberate phase. |
| Money in the UI | `priceUsd` for display only | All arithmetic will use `priceUnits`. Nothing does money math on a float. |

---

## The contract

### For the frontend

**Full integration guide, including who does what and the suggested order: [INTEGRATION.md](INTEGRATION.md).** Read that one before starting; this is the summary.

Every endpoint below is live and tested against real Neon + Hedera testnet. Nothing returns `501` any more.

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
GET   /api/games/:id/owned
GET   /api/games/:id/reviews
POST  /api/games/:id/reviews     ownership-gated
PATCH /api/reviews/:id
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
- **The session mock is Privy-shaped**: an email identity plus an embedded wallet with a balance. Swapping it is one file, `src/mocks/session.ts`.
- **Nothing persists.** Publishing pushes into an in-memory catalog that resets on reload. That's deliberate for now.
- Integration seams are marked `TODO(integration)` in the client. Grep finds all of them.

**Everything you flagged as missing an endpoint is answered now:**

- **`/invite/:id`** — `GET /api/invites/:id` (public) + `POST /api/invites/:id/accept` (auth). Accept sets `user_id` and `accepted_at`. No decline endpoint — not accepting *is* the decline, reversible by opening the link again.
- **Notifications** — `GET /api/notifications` + `POST /api/notifications/:id/read`. Plain polling, not SSE, not derived from HCS. Written by whichever handler causes the event — invite *accept* notifies the studio owner (the invitee has no account row to notify until they accept).
- **Publish media** — `POST /api/games` takes `build`, `media` (up to 8 files), `coverMediaIndex` marking the star. No index falls back to the generated cover.
- **Studio creation** — returns the created studio with its `id`. ENS availability is real today but DB-only, not chain-verified — see the endpoint list above.
- **Download path** — built and tested. `playUrl` is a direct `ipfs.io/ipfs/<cid>/index.html` URL, not a zip stream. That answers the origin question: `ipfs.io` is already a different origin from the app, so `allow-same-origin` on that iframe is safe by construction — **your self-hosted second origin is only needed for the pre-publish local preview**, not for purchased builds.
- **The x402 helper** — the signing bridge lives on the backend, because it needs Privy's server-side raw-signing primitive that the browser SDK doesn't expose. Ask for it when you get to checkout; don't build Hedera transactions in the browser.

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

Newest first. One entry per side per day, tagged so both of us can append without landing on the same line. If you worked on the other side that day, that's a second entry under the other tag, not a note inside your usual one.

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
