# Progress log

Shared status across the three repos. Two people, two halves, one file.

**Before you start:** read Status and Blockers. That is the whole handover.

**When you stop:** add one entry at the top of the Log, update the half of Status you moved, and add or clear blockers.

Keep this file to **what the other person needs**: anything crossing the repo boundary, any decision that changes the contract, anything that unblocks them. Work only one repo cares about belongs in that repo's `CLAUDE.md`, not here. Append; never rewrite someone else's entry.

Entries are tagged by **side**, not by person. Suparno is mostly on the frontend and Kai is mostly on the backend, but either of us can work on either, and the log should show what actually happened rather than who usually does what. Sign your name so the other one knows who to ask.

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

**Stage:** scaffolded, no running code yet.
**Deployed:** no.
**Next:** Express + Drizzle + Postgres up, Hedera client, Mirror Node helper, create the three HCS topics.

---

## Blockers

Only things stopping work right now.

| Who | Blocked on | Since | Needs |
|---|---|---|---|
| — | — | — | — |

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

### For the backend

What the client already assumes, so the API doesn't have to guess:

- **`CGS-client/src/mocks/types.ts` is the client's read of the contract.** Everything on screen is typed against it. If a shape changes, that file is the diff.
- **Browsing never touches auth.** Catalog, listing, studio pages, reviews and the invite screen all work signed out. Sign-in appears at buy and at publish, nowhere else.
- **Free games still mint a key.** `priceUsd: 0` is a real purchase with a real GameKey, not a bypass.
- **Splits are shown to buyers with handles and roles**, not just percentages, and there is no edit affordance anywhere in the app. Anyone invited by email is on the splits from the first sale whether or not they've accepted. The invite screen says so explicitly, so it has to be true.
- **The session mock is Privy-shaped**: an email identity plus an embedded wallet with a balance. Swapping it is one file, `src/mocks/session.ts`.
- **Nothing persists.** Publishing pushes into an in-memory catalog that resets on reload. That's deliberate for now.
- Integration seams are marked `TODO(integration)` in the client. Grep finds all of them.

**Screens that exist with no endpoint yet.** Not urgent, but they'll need one before integration:

| Screen | Needs |
|---|---|
| `/invite/:id` | Accept and decline. What's in the token, and whether the handle is set at accept time. |
| Notifications inbox | Sales, invites, agent buys, publishes. Poll, SSE, or derived client-side from the HCS topic. |
| Publish, media step | Several screenshots and clips plus a starred cover, not one image. Field names and limits on `POST /api/games`. |
| Studio creation | Live ENS subname availability while the dev types. An endpoint, or a direct chain read from the client. |
| Studio creation | `POST /api/studios` returning the created studio with its id, so the client can route to `/studio/:id`. |

Two things still open on the download path: whether `GET /api/games/:id/download` returns a zip stream or an IPFS CID the client fetches itself (a CID needs CORS on the gateway), and where a running build gets served from, given it can't be the app's origin (see Decisions).

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

### 2026-09-05 · Backend · Kai

Split into three repos. Settled the open stack decisions (see table above) by checking the actual packages rather than the docs — the SDK one turned out to be forced by an x402 dependency, not a preference. Wrote the API contract and setup notes. Added a `.gitignore` to CGS-server
