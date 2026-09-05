# Cosmic Game Sanctuary

**ETHOnline 2026 · async · Sept 4–16, 2026**
*A decentralized indie game storefront. itch.io, minus the payment processor that can delete you.*

---

## 1. The pitch

> In July 2025, itch.io deindexed every NSFW-tagged game on the platform overnight. No warning — developers found out when their games vanished from search. The reason wasn't a policy change: Visa and Mastercard, pressured by an activist campaign, threatened to cut off itch.io's ability to process *any* payment unless it complied. Steam did the same thing days earlier. Two of the biggest platforms for independent developers, one shared point of failure — a card network's opinion about what games are allowed to exist. Cosmic Game Sanctuary is a storefront where that veto is architecturally impossible: payment, ownership, payouts, and reviews all settle on-chain, so there's no processor to pressure and no account for anyone to delete.

**Shorthand:** CGS. Sanctuary as in *a place that can't be raided*, not as in cozy.

### Why this holds up as a hackathon pitch

- **It already happened.** Real event, real dates, real coverage. Not a speculative "imagine if agents needed X" premise.
- **The affected users asked for exactly this.** itch.io's own forums have a thread titled *"I guess it's time for a gaming platform that only accepts crypto payments."* Demand you can screenshot.
- **The incumbent won't build it.** itch.io's founder publicly declined a crypto payment processor's pitch. The gap stays open.
- **The mechanism was requested by devs years ago.** A 2021 itch.io forum post proposed, unprompted: *"if you sell access to a game as an NFT it's simple to write a smart contract to automatically route revenue share to contributors."* That's our team-payout feature, validated by the target user before we existed.
- **The demo shows product mechanics, not politics.** The NSFW story is framing at the open and close only — ten seconds each. Everything on screen is splits, ownership, autonomy, and review integrity.

---

## 2. Prior art — the honest version

**No ETHGlobal prior art.** Searching the showcase turns up plenty of single blockchain *games*, essentially no storefront/marketplace *infrastructure*. Real gap in hackathon-project space.

**Real commercial competitors exist. Do not claim we invented this.**

| | itch.io | Isotopic.io | altgames.io | Ultra.io |
|---|---|---|---|---|
| what | the incumbent | live store, 8% fee, crypto payout *optional* | built in response to the July 2025 crisis | older blockchain-native store |
| the gap | fully processor-dependent | crypto is an option alongside fiat → same processor risk likely still applies to most volume | small, early, payments-focused | leans launcher/client ecosystem |

*(Based on public/marketing info, not hands-on audit. Re-verify before submission — these move fast.)*

**Our honest positioning:** not "first," but *"crypto is the rail, not a checkbox — plus trustless team payouts nobody has built, and publicly verifiable ownership and reviews that outlive the company, delivered at itch.io's friction level."*

---

## 3. The edge — what CGS does that none of them do

| axis | them | CGS |
|---|---|---|
| **payment independence** | crypto as an *option* next to a card processor — the processor relationship still exists, so the pressure vector still exists | x402/stablecoin **is** the rail. no processor relationship to lean on. this is the difference between "supports crypto" and "can't be deplatformed" |
| **team payouts** | manual, off-platform, trust-based — one person holds the account and is supposed to pay everyone later | on-chain split contract, pre-committed at upload, executes at time of sale. **nobody else has this** |
| **review authenticity** | platform-attested ("trust us, this person owns it") | cryptographically verifiable by any third party without trusting our database — reviewer must hold the GameKey |
| **survivability** | company dies → library and reviews die | GameKey is a token in *your* wallet; catalog is a public index. store dies, ownership doesn't |
| **friction** | itch.io: zero, browser-first. web3 stores: reintroduce wallet setup, gas, approvals — and lose the whole advantage | Privy embedded wallet, email/social login, buy-and-play in one flow. **this is a first-class design constraint, not a polish item** |

The friction row is the one that kills web3 game platforms. Over-indexing on the blockchain part and under-indexing on "just let me click play" is the standard failure mode. Treat wallet UX as core scope.

---

## 4. What we build

### 4.1 Buy & own — Hedera (x402 + HTS + HCS)
- Listings support pay-what-you-want (incl. $0) or fixed price in USDC/HBAR.
- Purchase flows through an **x402-gated download/play endpoint on Hedera**. This is Hedera's required "real x402-gated service"; the storefront is the platform consuming it.
- Success mints a **GameKey via Hedera Token Service** into the buyer's wallet. It unlocks the build. It's a real asset — tradeable, giftable, yours — not a revocable license tied to an account we control.
- Purchases, splits, and reviews log to an **HCS topic**: public, timestamped, tamper-evident sales ledger. (Hedera calls this out as an extra-points item.)

### 4.2 Trustless team payouts
- At upload, a team pre-commits a split: coder 50 / artist 30 / composer 20.
- Every sale divides automatically at settlement. No one holds the money. No one chases anyone on Discord three weeks later.
- This is the single most differentiated feature in the product and the strongest demo beat.

### 4.3 Verified-purchase reviews
- Only a wallet holding that game's GameKey can review it.
- Kills review-bombing by people who never bought the game, and does it in a way anyone can verify independently — not "our moderation team checked."

### 4.4 Studio & game identity — ENSv2 (see §5.3 for why this shape)
- Each studio deploys its **own subname registry** under the CGS parent.
- Games are subnames: `hollowgrave.studioname.eth`, with build CID and metadata in the resolver.
- **Enhanced Access Control** gives real per-role permissions inside a jam team.
- Expiring/revocable subnames for demo builds, playtest access, jam-period listings.

### 4.5 Wishlist sniper bot
A player sets a **trigger**, not a bookmark: *buy automatically if it drops under $2 / goes pay-what-you-want / a jam bundle drops.*

How it works:
1. Player pre-approves a capped allowance (Privy session key / spending limit — "up to $20 total, max $5 per purchase").
2. Bot watches the subgraph's listing + price-change feed.
3. On match it constructs and pays the x402 request itself — **same endpoint, same call path a human's Buy button uses**. Not a backdoor.
4. GameKey lands in the *player's* wallet, not the bot's. Event logs to HCS.
5. Player gets: "Auto-purchased Hollowgrave for $1.80 — your trigger fired."

Why it matters beyond being a nice feature: it's what makes Hedera's *agentic* payments brief literally true instead of loosely true, it's the ENSv2 bonus criterion (agent with its own namespace + delegated permissions), and it's the best beat in the demo — money moves with nobody touching a mouse.

Effort: ~half a day. Reuses the buy call and the subgraph you already built. A polling loop is fine; don't reach for an agent framework.

### 4.6 Frictionless money — Privy (see §5.2)
Email/social login → embedded wallet → fund → buy → play. No seed phrase, no extension install, no "first, set up MetaMask." Also powers the sniper bot's capped spending allowance and the payout leg.

### 4.7 The subgraph — built for product reasons, not prize reasons

*(Recap, since this got fuzzy: a subgraph indexes your contract events into a fast GraphQL API.)*

Three reasons it's non-optional regardless of sponsors:
1. **The storefront can't function without it.** "All games under $2, puzzle tag, newest first, with review counts" is not answerable from raw chain state.
2. **Thesis consistency.** If the catalog only lives in our Postgres, *we* are the single point of failure the pitch claims to eliminate. A public index means anyone can run an alternate frontend if we go down or get pressured.
3. **The sniper bot's price feed.**

It no longer wins a Graph prize (§5.4). Build it anyway.

---

## 5. Tracks

**Test applied:** is the integration *necessary to the product* (we'd want it with zero prize money) **and** does it give the sponsor a genuinely good showcase? Both, or it's out.

### 5.1 Hedera — confirmed, the core rail
The whole buy/own/split/log loop runs on it. Remove it and there's no product.

**Their upside:** proof that real x402 commerce runs on Hedera — recurring small-dollar transactions from an actual consumer product, plus an agent that budgets and pays autonomously. Closer to their agentic-payments narrative than most things in the track.

**Covered:** x402-gated endpoint ✓ · consuming platform ✓ · HTS tokens ✓ · HCS audit trail (extra points) ✓ · autonomous agent (extra points) ✓

### 5.2 Privy — "Best financial flow," $2,500
Their brief names *payments* and *payouts* explicitly. We are a payments-and-payouts product.

Requirements → what we have:
| requirement | us |
|---|---|
| Privy as a core part of the product | login + wallet + funding is the entire onboarding path |
| create/use ≥1 Privy wallet | every buyer and every dev |
| ≥1 functional financial flow | **three**: onramp/funding, purchase transfer, 3-way payout split |
| hide unnecessary onchain complexity | our whole UX thesis, in their words |
| explain how Privy improves UX | itch.io's magic is click-and-play; every web3 store loses it at wallet setup. we don't |
| working demo + source | yes |

**Lead the submission with the payout flow.** "Gamer pays $3 with an email-login wallet, three devs get paid automatically, nobody in the flow ever saw a seed phrase" is a far better Privy story than "we added social login."

Note their carve-out: Privy Cards can be mocked but doesn't count as the required integration — so keep the live flow on transfers/funding, which we have anyway.

> ### ⚠️ DAY-ONE RISK — verify before anything else
> Privy is primarily EVM + Solana. Hedera has an EVM-compatible layer with a JSON-RPC relay and Privy supports custom EVM chains, so this *should* work — **but it is not confirmed**, and our entire payment rail is Hedera. If Privy can't sign cleanly against Hedera's EVM layer, the Privy+Hedera combination we're building around breaks.
> **Spend the first hour of day 1 on this.** Not day 10.

### 5.3 ENS — "Best Use of ENSv2," $4,500 (four payout slots)
ENSv2 beta is live on Sepolia. Both ENS tracks require it, and the qualification line is explicit: *"ENSv2 features should be central to the product, not a cosmetic add-on."*

**So `studioname.eth` + text records is dead** — that's precisely the cosmetic version they're rejecting. The version that qualifies:

- **Own subname registry per studio.** Studio deploys its own registry; each game is a subname with build CID + metadata in its resolver.
- **Enhanced Access Control = real jam-team permissions.** Artist can edit cover art records, coder can push a new build hash, composer can edit neither. Right now three people share one itch.io login and hope for the best. This pairs directly with the revenue-split feature — same team, same trust problem, solved at both the money layer and the permission layer.
- **Expiring / revocable / non-transferable subnames** for demo builds, timed playtest access, jam-period listings.
- **Their bonus criterion, satisfied for free:** *"agents as namespaces, each with their own identity and permissions"* — the sniper bot gets its own subname with a delegated, capped spending permission. We were building the bot anyway.

**Why the odds are good:** ENSv2 is brand-new beta, so far fewer teams will have touched it; four payout slots (1500/1500/1000/500); sponsors reward early adopters of new stuff.

**Risk:** beta tooling and thin docs could eat a day or two. Budget for it.

### 5.4 The Graph — OUT (but still build the subgraph)
Re-read the live prize page. There is **no plain "best subgraph" track anymore.** Three tracks, all $5k:

1. **Composable/Standardized Products** — explicitly: *"simply querying one Subgraph with no composition or standardization does not qualify."* Requires composing 2+ Graph products or building on a standardized schema. That's work with zero product benefit for us.
2. **AI Tooling/Use Case (From Scratch)** — the only door we fit. The bot querying the subgraph and auto-paying via x402 qualifies (their brief even says *"let your agent pay per query autonomously with x402"*).
3. AI (Continuity) — for existing projects, not us.

**Verdict: don't build around it.** The AI track will be the most crowded at the event, and a price-trigger bot competes poorly against research assistants and portfolio copilots. Forcing it also promotes the bot from "nice half-day addition" to "load-bearing or we get nothing."

**But:** if by day 10 the bot does more than poll, throwing a submission at the AI track costs nearly nothing. Lottery ticket, not a track.

### 5.5 Explicitly skipped
- **World** — only tracks are Continuity (old projects, not us) and Selfie Check. A biometric selfie gate is a bad fit for a store whose whole pitch is *fewer* gatekeepers, and it's not something we'd build without the prize. Out.
- **1inch** — this cycle wants a SwapVM/Aqua position. Doesn't map onto a marketplace. Forcing it reads worse than not entering.
- **0G / Uniswap / Chainlink** — decentralized storage is architecturally necessary, but 0G *specifically* isn't; IPFS/Arweave are more mature for a 12-day build. Uniswap resale-royalty hooks are a real but secondary-market edge case. Chainlink unpublished.

**Final: Hedera + Privy + ENS.** Three deep, mutually-beneficial integrations beats seven shallow ones.

---

## 6. Trust & safety — non-negotiable

The obvious attack on "censorship-resistant": *what if someone uploads CSAM?* Every judge and every real user should ask this, and having a real answer is what separates this from a naive "decentralized = uncensorable" pitch.

**The distinction the whole product rests on:** censorship-resistant means resistant to *a payment processor's opinion about legal content* — the actual itch.io failure mode. It never means illegal content can't be removed. Those are different things and the README must say so plainly.

- **Two layers, moderated differently.** Storage (IPFS/Arweave — the bytes) is separate from the storefront index (what's findable and playable *in our product*). We operate the index. We can and must delist reported content from it instantly. Decentralizing storage is not abdicating moderation.
- **Upload-time hash matching, before indexing.** Every upload checked against known-CSAM hash databases (PhotoDNA / NCMEC hash lists / Thorn's Safer). Industry standard, not exotic.
- **Mandatory reporting.** In the US, 18 U.S.C. § 2258A requires service providers to report to NCMEC's CyberTipline on discovery. Legal obligation, not a roadmap item.
- **Upload accountability.** ENSv2 identity + on-chain uploader record means uploads aren't anonymous. Anonymity is the enabling factor for this abuse; removing it is a real deterrent and gives any legal process something to act on.
- **Published policy drawing the line itch.io blurred under pressure:** legal content a processor merely dislikes → protected. Illegal content → removed, reported, zero exceptions.

For a 12-day build the full pipeline doesn't need to ship, but this section must be in the README. "Censorship-resistant hosting" with no answer here isn't an oversight a judge lets slide — and more importantly it's the difference between a defensible product and an indefensible one.

---

## 7. The demo

**The trap:** "publisher uploads, gamer downloads" looks like a CRUD app with a wallet bolted on, and it accidentally makes the NSFW backstory look like the product. Fix: every beat proves a *different* claim, and none of it shows adult content.

**Beat 0 — framing (10s).** One line: two of gaming's biggest platforms just showed indie devs their livelihood depends on a card network's opinion.

**Beat 1 — upload with a pre-committed split (10s).** Three-person jam team uploads a real playable HTML5 game, sets 50/30/20 on screen. Not "upload a game" — "upload a game with a contract-enforced payout structure."

**Beat 2 — one purchase, three wallets move (20s).** ⭐ *The money shot.* All three teammates' balances visible side by side. Buyer pays $3 with an **email-login wallet, no seed phrase, no extension**. On confirmation all three balances update — $1.50 / $0.90 / $0.60 — with zero manual action.
> *"On itch.io this goes to one person's account and the other two just have to trust them. Here it isn't trust. It's the transaction."*

**Beat 3 — instant play (15s).** GameKey lands in the buyer's wallet. Game runs in-browser. No client, no account, no install.

**Beat 4 — the bot fires, nobody touches anything (30–40s).** Second buyer has a wishlist trigger. Dev drops the price live on screen. Seconds later, no click: *"Auto-purchased for $1.80."* Makes agentic payments concrete instead of asserted.

**Beat 5 — fake review rejected on camera (15–20s).** A wallet that never bought the game tries to review-bomb the listing → rejected, no GameKey. Real buyer's review posts fine. Fast payoff on the ownership mechanic from beats 2–3.

**Beat 6 — close (10s).** The real headlines. *"This happened. Here's where it can't."*

Under 3 minutes — comfortably inside Hedera's 5-min cap and ENS/Graph's 2–4 min asks. Reuse one master video across submissions.

---

## 8. Build plan (~12 days)

### Day 1 — de-risk first, build second
- ⚠️ **Hour 1: Privy ↔ Hedera EVM relay.** If it doesn't work, we need to know now — it changes the architecture, not the polish.
- Then: x402-gated endpoint live on Hedera (`scaffold-hbar` + Blocky402 facilitator).
- Spin up ENSv2 on Sepolia early too — beta tooling surprises are better found on day 1 than day 7.

### Days 2–3 — core loop
- HTS GameKey minting on successful payment.
- Privy login → embedded wallet → fund → buy.
- Storefront skeleton: list, buy, get key, play. One real HTML5 game in the catalog — **use an open-source game, do not build one.**

### Days 4–6 — the differentiators
- On-chain revenue splitter (pre-commit at upload, execute at sale).
- Payment-gated review contract.
- HCS logging for purchases/splits/reviews.
- Decentralized storage for builds (IPFS/Arweave).

### Days 7–9 — ENSv2 + agent
- Per-studio subname registry, games as subnames, build CID in resolver.
- Enhanced Access Control roles for jam teams (the piece that makes ENSv2 *central*, not cosmetic).
- Subgraph deployed, storefront browse/search wired to it.
- Wishlist sniper bot + its own ENS subname with delegated capped spending.

### Days 10–11 — demo assets
- 2–3 real playable HTML5 games in the catalog so it doesn't look empty on camera.
- Record the master demo video.
- Per-track READMEs: Hedera (architecture + payment flow), Privy (**how Privy improves UX** — explicitly required), ENS (functional, no hardcoded values — explicitly required).

### Day 12 — buffer.

### Descope order (cut from the bottom)
1. ~~Graph submission~~ ← first to go
2. ~~Sniper bot~~ ← but this costs the Hedera agentic angle + ENS bonus; cut reluctantly
3. ~~Expiring subnames / advanced EAC roles~~ ← keep at least basic EAC or ENS becomes cosmetic and disqualifies
4. ~~Decentralized storage~~ ← weakens the thesis but doesn't break the demo

**Never cut:** buy → GameKey → play → payment-gated review, the 3-way split, or Privy login. That's the story.

---

## 9. Honest risks

- **Privy ↔ Hedera compatibility is assumed, not verified.** Single biggest technical unknown. Day 1, hour 1.
- **ENSv2 is beta.** Thin docs, possible breakage. Start it early, and don't let it slide into a cosmetic integration — cosmetic *disqualifies* on that track.
- **Isotopic and altgames are real.** Don't pitch "nobody's built this." Pitch depth: crypto as the rail, trustless payouts, verifiable reviews, itch-level friction.
- **Scope creep via native builds.** Browser-playable HTML5/WASM only. itch.io's magic is click-and-play, not a 40GB installer, and decentralized storage can't carry native builds in 12 days.
- **The friction trap.** The moment a judge has to install a wallet extension, our core differentiator evaporates on camera. Privy isn't a track integration, it's the defense of the pitch.
- **Politically charged framing.** The event is real and documented — that's the strength. Stay factual (dates, what happened). Don't editorialize about the activist group or the processors. Let the facts carry it.

---

## 10. 60-second pitch

> "In July 2025, itch.io deleted every adult game from search overnight. No warning — developers found out when their games disappeared. Why? Visa and Mastercard threatened to cut off itch.io's ability to process *any* payment unless they complied with an activist campaign. Steam did the same thing the same week. Two of the biggest platforms for independent developers, one shared point of failure: a card network's opinion.
>
> Cosmic Game Sanctuary is a storefront where that veto is architecturally impossible. Payment runs on x402 — there's no processor to pressure. Every purchase mints a real ownership key into your wallet, not a license we can revoke. And when a jam team ships together, the split happens on-chain the moment someone buys — no more one person holding the money and everyone else hoping.
>
> Watch: three devs, one $3 purchase, three wallets update live. The buyer logged in with an email — no seed phrase, no extension. Game plays instantly in the browser. A wishlist bot buys a second game on its own when the price drops. And someone who never bought the game tries to review-bomb it — rejected, because reviews require ownership.
>
> This isn't hypothetical. It already happened to real developers. This is where it can't."
