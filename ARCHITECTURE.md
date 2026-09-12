# How Cosmic Game Sanctuary works

A storefront for browser-playable indie games where the payment rail, the
ownership record, the revenue split and the catalog are all public
infrastructure rather than rows in our database.

This is the technical companion to [README.md](README.md). It covers the four
flows that matter, what lands on chain, and why each piece of the stack is
load-bearing rather than decorative. [Four-minute demo](https://youtu.be/WyJehf7Vgb4).

---

## The shape

Three repos, not a monorepo. [CGS-server](https://github.com/cosmic-game-sanctuary/CGS-server)
is Node + TypeScript + Express over Postgres, and owns everything chain-facing.
[CGS-client](https://github.com/cosmic-game-sanctuary/CGS-client) is Vite +
React 19, and is the only thing that ever touches a buyer's key. This repo is
the shared contract and status.

Hedera testnet carries the money, the ownership token and the public record.
Sepolia carries the names. Payments go over [x402](https://x402.org), settled
through the Blocky402 facilitator. Privy provides login and both kinds of
wallet. Builds are pinned to IPFS via Pinata.

Nothing here is a custom contract. There is no Solidity in the tree at all —
ENSv2's own contracts do the name enforcement, Hedera's native services do the
rest, and that was a deliberate choice rather than a shortcut.

---

## Four flows

### Publishing

A developer makes a studio, optionally claiming a name on our ENS subregistry.
They drop a zip, which is unpacked and *run in the browser* before it is ever
uploaded — the preview is the real build on a real isolated origin, not a
screenshot. Then details, price, and the split.

Splits name collaborators by email, not by wallet address. That is the whole
reason a jam team can use this: someone who has never touched crypto is on the
credits and earning from the first sale, and they claim the wallet later. On
publish, the splits lock, an HTS NFT collection is created for the game, the
build is pinned to IPFS twice (directory for provenance, zip for delivery),
and a `listed` message goes onto the public listings topic.

There is no endpoint to edit a split after publish. Not a missing feature —
the absence is the promise.

### Buying

The interesting part is who signs.

```
browser                     server                    Blocky402         Hedera
   │  GET /games/:id/download  │                          │                │
   │ ─────────────────────────>│                          │                │
   │   402 + payment terms     │                          │                │
   │ <─────────────────────────│                          │                │
   │                           │ build + freeze transfer  │                │
   │   hashes to sign          │                          │                │
   │ <─────────────────────────│                          │                │
   │ secp256k1_sign (Privy)    │                          │                │
   │   signatures              │                          │                │
   │ ─────────────────────────>│ ───────────────────────> │ ──────────────>│
   │                           │                       verify + settle     │
   │   build + playUrl         │                          │                │
   │ <─────────────────────────│                          │                │
```

The server builds and freezes the transfer, because that needs a Hedera
client. The browser signs the hashes, because the key belongs to the person.
The server settles. At no point does the server hold a buyer's key, and we
never asked anyone to delegate their wallet to a shop.

Settlement is the moment of entitlement. The GameKey mint and the split
distribution both run *after* the response, so the game boots in the same tab
in seconds rather than waiting on six chain round trips.

Revenue reaches the whole team in **one** `TransferTransaction` — one debit,
N credits, executed once. Either everyone is paid or nobody is. A collaborator
who has not claimed their wallet yet has their share held, and it goes out the
moment they accept their invite, paid directly to their EVM address, which
under HIP-542 creates their Hedera account as a side effect of the payment.

### The agent

One agent per person, not one per game. It has its own Privy wallet, its own
Hedera account, its own HCS-14 identity, optionally its own ENS name, and a
list of *wants* — a wishlisted game plus a price ceiling.

**It reads the public listings topic through the Mirror Node.** Not a database
flag, not a webhook from us. That is the difference between an app with a bot
in it and a public action anyone could independently rebuild, and it is the
one architectural decision in this project we would not trade away.

It does not buy the instant a price drops. A sale is open until it ends, so
buying early gains nothing and gives up the chance to compare. The agent
decides at the **wire** — an hour before the soonest sale it is watching ends
— so that one decision sees everything that arrived while it waited. A price
drop usually just moves the alarm clock.

When the greedy plan is unambiguous, no model is involved and nothing extra is
spent. When it genuinely cannot have everything it wants, it pays for a
verdict:

```
GET /api/agent/verdict   →  402 + terms
                         →  agent signs with its own wallet
                         →  settle, then reason, then answer
```

That endpoint takes no request body. What to decide about is derived entirely
from *who paid* — the settled payment names an account, and if that account is
a known agent, its wants and balance are recomputed fresh server-side rather
than trusted from whatever the caller claimed. Same rule as everywhere else
here: the chain is the identity, not the request body.

Its spending cap is its balance. There is no cap field, because there does not
need to be one.

### Paid trials

Try a game by the minute. The same build, the same isolated origin, the same
boot sequence a purchase gets. The difference is what was bought: time.

The meter buys the next chunk shortly before the current one runs out, each a
real x402 payment signed silently by the player's own embedded wallet. Playing
is paying. Leaving stops it — there is no server-side timer to cancel, because
the loop lives in the tab and dies with it.

Every cent spent trialling comes off the price if they buy, derived live from
the sales rows rather than held as a counter that could drift.

When the time runs out the frame is **unmounted**, not covered. A cross-origin
build has no pause to call, so removing it from the DOM is the only thing that
actually stops it. The clock is honoured by the page, not enforced by the
server — the build is unpacked in the browser, so pretending otherwise would
be theatre.

---

## What lands on chain, and where to check it

Everything below is testnet and public. Nothing needs our permission to read.

| | |
|---|---|
| Listings topic | [`0.0.10380868`](https://hashscan.io/testnet/topic/0.0.10380868) — listed, price changed, build updated, delisted, relisted, demand |
| Sales topic | [`0.0.10380869`](https://hashscan.io/testnet/topic/0.0.10380869) — every settled purchase and trial chunk |
| Agent identity topic | [`0.0.10380872`](https://hashscan.io/testnet/topic/0.0.10380872) — HCS-14 identities, each naming its funding human |
| Settlement asset | USDC `0.0.429274`, an HTS token |
| GameKey | one HTS NFT collection per game |
| ENS parent | `cgs-sanctuary.eth` on Sepolia |
| Subregistry | [`0xbD7E9E226a6Dd9641Adb9E00d86A0E2EDbcd6c2D`](https://sepolia.etherscan.io/address/0xbD7E9E226a6Dd9641Adb9E00d86A0E2EDbcd6c2D) — our own instance |

**The Mirror Node is the only ground truth.** SDK receipts and
`ScheduleInfoQuery.executedAt` can both be stale or wrong; we learned that the
expensive way and now nothing in this codebase believes a write happened until
the mirror says so.

As of writing, on testnet: 36 settled purchases, 50 metered trial chunks, 24
GameKeys minted, and 19 recorded agent decisions — 13 that ended in a buy, 5
that passed on a game, and 1 it deliberately held until the wire.

---

## Why each sponsor's tech is load-bearing

### Hedera

The whole payment and ownership layer, and the part of the product that could
not be built the same way anywhere else.

**Two x402-gated resources, not one.** The game download is the obvious one.
The second is `GET /api/agent/verdict`: the agent pays for its own reasoning,
per verdict, out of its own wallet. That is metered pay-per-call inference
rather than a flat subscription, and it is real — the settlement transaction id
is stored on the decision row and links straight to HashScan.

**A third that meters itself.** A paid trial charges the player's own wallet
for one chunk of play at a time, on a loop, and stops the instant they close
the tab. Same rails, no agent involved, and it is end-to-end metered
consumption by a human rather than a machine.

**HTS in the settlement path, both ways.** The asset moved is an HTS token.
The thing received is an HTS NFT. The GameKey is created with a supply key and
**nothing else** — no wipe key, no freeze key, no pause key, no admin key. We
are structurally incapable of taking back what we sold, and that is checkable
on HashScan rather than promised in a README.

**HCS as the audit trail and the agent's actual input.** Three topics, read
back through the Mirror Node. The listings topic is not a log we write for
tidiness — it is what the agent reads to decide. Delete our database and the
price history survives.

**HCS-14 for agent identity**, derived offline and anchored on a topic. The
reference SDK never finished installing in our environment, and the AID spec
is explicit that anyone must be able to derive the identifier from canonical
public inputs without permission — so it is implemented directly from the
spec: SHA-384 over canonical JSON, Base58, six fields, keys alphabetical.
Ten agents carry one today.

Honest about what we do not have: no A2A or ACP negotiation, no UCP manifest,
and the recurring trial payment is a client loop over x402 rather than
`ScheduleCreateTransaction`. We do not claim those.

### ENS

Names for studios and for agents, out of a registry we deployed and own.

We use ENS's own `PermissionedRegistry`, our own instance of it, deployed
through their `VerifiableFactory` as a UUPS proxy we control, called directly
with viem. The parent name went through a real commit–reveal registration with
a genuine 60-second `MIN_COMMITMENT_AGE` wait. Permission scoping is their
Enhanced Access Control role bitmaps, taken verbatim from `RegistryRolesLib.sol`
rather than guessed. No custom registrar, no custom Solidity anywhere.

A subname owner gets `ROLE_SET_RESOLVER | ROLE_RENEW` — enough to point their
own name somewhere and keep it alive, not enough to unregister it or transfer
it out from under the platform.

**The part that makes this central rather than cosmetic: agents have names on
the same registry.** An autonomous buyer with a name, a wallet, an HCS-14
identity and a public spending record is a namespace with its own identity and
permissions, which is ENS's own stated bonus almost word for word. Two exist on
Sepolia today — `suved.cgs-sanctuary.eth` and `best-agent.cgs-sanctuary.eth` —
each pointing at the agent's own account, not its owner's.

Studio and agent names share one flat namespace, so they compete for the same
label and one availability check answers for both. A name is write-once: there
is no rename endpoint, because renaming would mint a second name and leave the
first pointing at the same wallet.

We deliberately do not claim wildcard resolution or a per-subname Permissioned
Resolver. We use ENS's standard resolver, and saying otherwise would be false.

### Privy

Login, both wallets, and one design decision worth the whole integration.

A buyer signs in with an email and gets an embedded wallet. No seed phrase, no
extension, no wallet-connect-first landing page. That is table stakes and it
works.

**The part that matters is the signing split.** The server builds and freezes
the transfer, the browser signs the hashes with
`provider.request({ method: 'secp256k1_sign' })`, and the server settles. That
raw-hash primitive with no Ethereum message prefix is exactly what Hedera
needs, and it is the reason the server never holds a buyer's key.

The alternative was delegated signing — asking every buyer to grant the shop
standing permission to move their money before their first purchase. That is a
much larger thing to ask than "approve this purchase", and a storefront should
not be asking it. So we did not.

The server *does* sign, but only ever with wallets it created itself:
`privy.wallets().create({ chain_type: 'ethereum' })` makes the agent's wallet,
and `privy.wallets().rpc(id, { method: 'secp256k1_sign' })` is how the agent
pays for games and for its own inference. App-owned and user-owned wallets in
one system, with a clear line between them.

Auth is `verifyAccessToken`, verified locally with no network hop on every
request.

Honest gap: **we do not integrate Privy's funding or onramp.** A buyer funds by
receiving USDC from elsewhere, and that is the one place onboarding still feels
like crypto.

---

## Things that are true in the code

A short list, each checkable rather than asserted.

The agent reads HCS through the Mirror Node, never a database flag —
`src/agent/watcher.ts`.

Splits lock at publish. There is no edit endpoint to find.

The GameKey has no wipe, freeze, pause or admin key —
`src/services/hedera/hts.ts`, and HashScan agrees.

Revenue splits go out in one atomic transaction —
`src/services/games/fulfil.ts`, one debit and N credits.

Trial credit is a live sum over sales rows, never a stored counter that could
drift from what was actually paid.

An invite link is not a bearer token: accepting one checks the signed-in
address against the address it was sent to.

---

## What we did not build, and why

No resale or secondary market. No refunds. No editing splits after publish. No
identity verification on upload, because identity checks harm exactly the
creators this exists to protect. No adult content, which also removes any
age-gate obligation. Each was cut for a reason rather than left undone.

## What is not done

Not deployed — everything above runs on testnet against live infrastructure,
but there is no public URL yet. Email delivery needs an SPF record and a real
`APP_URL`. Privy's onramp is not integrated. No CSAM-scanning provider is
wired, so uploads fail closed by design rather than silently passing.
