# How it works

Three things here are unusual: an **autonomous agent** with its own wallet that
chooses which games to buy, **revenue splits** that pay a whole team in one
atomic transaction, and **trials metered by the minute** over x402.

Everything they depend on — the money, the ownership token, the price history —
lives on public infrastructure, so none of it needs us online or honest.

📺 **[Four-minute demo](https://youtu.be/WyJehf7Vgb4)** · 📖 [The pitch](README.md) · 🔌 [API contract](INTEGRATION.md)

**Jump to:** [Buying](#buying-a-game) · [Publishing](#publishing) ·
[The agent](#the-agent) · [Paid trials](#paid-trials) ·
[On-chain reference](#whats-on-chain) · **Sponsors:** [Hedera](#hedera) ·
[ENS](#ens) · [Privy](#privy) · [Gaps](#whats-not-done)

---

## Stack

| | |
|---|---|
| Backend | Node, TypeScript, Express, Postgres via Drizzle |
| Frontend | Vite, React 19, Tailwind v4 |
| Money + ownership | Hedera testnet — HTS, HCS, Mirror Node |
| Payments | [x402](https://x402.org) over HTTP, settled by Blocky402 |
| Wallets + login | Privy |
| Names | ENSv2 on Sepolia |
| Builds | IPFS via Pinata |
| Agent reasoning | Groq, `openai/gpt-oss-20b`, strict JSON schema |

No Solidity. ENS's contracts handle names, Hedera's native services handle
everything else.

---

## Buying a game

You sign in with an email. Privy makes you a wallet. You press buy, and about
four seconds later the game is running in the same tab.

Underneath, the request to download the build comes back `402 Payment
Required` with terms attached. What happens next is the part worth looking at:

```
browser                      server                  Blocky402        Hedera
   │  GET /games/:id/download   │                        │               │
   │ ──────────────────────────>│                        │               │
   │  402 + terms               │                        │               │
   │ <──────────────────────────│                        │               │
   │                            │ builds + freezes the transfer          │
   │  hashes to sign            │                        │               │
   │ <──────────────────────────│                        │               │
   │  secp256k1_sign via Privy  │                        │               │
   │ ──────────────────────────>│ ─────────────────────> │ ─────────────>│
   │                            │                 verify + settle        │
   │  build + playUrl           │                        │               │
   │ <──────────────────────────│                        │               │
```

The server freezes the transfer because that needs a Hedera client. The
browser signs it because the key is yours. We never hold it.

Settlement is the moment you own the game. The GameKey mint and the revenue
split both run *after* the response, so you are playing while the chain work
finishes behind you.

---

## Publishing

Make a studio, optionally claim a name. Drop a zip — it unpacks and runs in
your browser immediately, before it uploads anywhere, so you see the real build
rather than a spinner. Set a price. Split the revenue.

**You split with people who have only an email address.** Type
`sam@gmail.com`, 40%, done. Sam is on the credits and earning from the first
sale, having never touched a wallet. When Sam eventually clicks the link in
their inbox, everything held for them lands at once.

On publish the splits lock, an HTS NFT collection is created for the game, the
build is pinned to IPFS twice (directory for provenance, zip for delivery), and
a `listed` message goes on the public topic.

There is no endpoint to edit a split afterwards. We wrote the product so we
could not do it either.

When a sale settles, the whole team is paid in **one** `TransferTransaction`:
one debit, N credits, executed once. Everyone or nobody.

---

## The agent

Here is the scenario it exists for, which we have actually run:

> Two games on your list, both $1.00, both on sale. Your agent holds $1.20. It
> can have one.

A dumb bot buys whichever went on sale first. Ours waits.

It has its own Hedera account, its own Privy wallet, its own
[HCS-14](https://hashgraphonline.com/docs/standards/hcs-14) identity, and
optionally its own ENS name. You give it games and a ceiling for each, then
fund it. The balance *is* the spending cap — there is no cap field because
there does not need to be one.

**It reads the public listings topic through the Mirror Node.** Not our
database, not a webhook from us. Someone else could write a competing agent
against the same feed tomorrow and we could not stop them, which is the point.

**It decides at the wire.** A sale is open until it ends, so buying early gains
nothing and gives up the chance to compare. The agent schedules itself for an
hour before the soonest deadline it is watching, and decides then, having seen
everything that arrived in the meantime. A price drop usually just moves the
alarm clock.

**When the choice is genuinely contested, it pays to think.** Most rounds are
unambiguous and no model is involved. When the money is actually spoken for, it
calls a second x402-gated endpoint and is charged per verdict, from its own
wallet:

```
GET /api/agent/verdict  →  402 + terms  →  agent signs  →  settle  →  answer
```

That endpoint takes no request body. What it reasons about comes from *who
paid* — the settled payment names an account, and the wants and balance are
recomputed server-side from that. A client cannot talk it into considering
something by lying in a payload.

The model's answer is then checked rather than trusted. Every game it names has
to be genuinely eligible and the total has to genuinely fit the balance, or the
whole verdict is thrown away and the deterministic plan runs instead.

Every round is logged with what it saw, what it chose, what it passed on, and
the transaction that paid for the reasoning.

---

## Paid trials

Play by the minute before you commit. Same build, same isolated origin, same
boot as a purchase.

The meter buys the next minute shortly before the current one runs out, each a
real x402 payment signed quietly by your own wallet. Playing is paying. Close
the tab and it stops — the loop lives in the page, so there is no server timer
to cancel and nothing keeps charging you.

Everything you spent trying it comes off the price if you buy, summed live from
the payment rows rather than tracked in a counter that could drift.

When the time is up the game frame is **removed from the page**, not covered. A
cross-origin build has no pause button to press, so unmounting it is the only
thing that genuinely stops it.

---

## What's on chain

Testnet, all public, none of it needs our permission to read.

| | |
|---|---|
| Listings topic | [`0.0.10380868`](https://hashscan.io/testnet/topic/0.0.10380868) — listed, price changed, build updated, delisted, relisted, demand |
| Sales topic | [`0.0.10380869`](https://hashscan.io/testnet/topic/0.0.10380869) — every settled purchase and trial minute |
| Agent identities | [`0.0.10380872`](https://hashscan.io/testnet/topic/0.0.10380872) — HCS-14, each naming the human funding it |
| Settlement asset | USDC [`0.0.429274`](https://hashscan.io/testnet/token/0.0.429274) |
| GameKey | one HTS NFT collection per game |
| ENS subregistry | [`0xbD7E…6c2D`](https://sepolia.etherscan.io/address/0xbD7E9E226a6Dd9641Adb9E00d86A0E2EDbcd6c2D) on Sepolia, under `cgs-sanctuary.eth` |

**The Mirror Node is the only thing we believe.** SDK receipts and
`ScheduleInfoQuery.executedAt` can both be stale or wrong. We learned that
expensively and now nothing counts as done until the mirror agrees.

So far: 36 purchases, 50 metered trial minutes, 24 GameKeys minted, 19 agent
decisions (13 buys, 5 passes, 1 held to the wire), 5 named studios, 2 named
agents.

---

# Sponsors

## Hedera

**The short version:** three different things here are paid for with x402 on
Hedera, and one of them is an AI agent paying for its own reasoning.

**1. Buying a game** — `GET /games/:id/download`, the obvious one.

**2. The agent's own inference** — `GET /api/agent/verdict`. When the agent has
to choose between games it cannot both afford, it pays per verdict out of its
own wallet. Metered pay-per-call, not a subscription. The settlement
transaction id is stored on the decision row, so you can click from "it chose
this" to the transfer that paid for the thinking.

**3. A trial that meters itself** — `GET /games/:id/trial/chunks/settle`. A
human's wallet is charged for one minute of play at a time, on a loop, and
stops when they close the tab. Same rails, no agent involved.

All three settle through **Blocky402**.

**HTS on both sides of the trade.** You pay in USDC, an HTS token. You receive
a GameKey, an HTS NFT created with a supply key and *nothing else* — no wipe
key, no freeze key, no pause key, no admin key. We cannot take back what we
sold, cannot change the token, cannot freeze your account. That is verifiable
on HashScan in about ten seconds, which is why we built it that way instead of
writing a promise in a README.

**HCS is an input, not a log.** The agent's decisions come from reading the
listings topic through the Mirror Node. Price history survives us. And
**HCS-14** gives each agent an identity anchored on a topic that names the human
funding it — implemented straight from the spec (SHA-384 over canonical JSON,
Base58, six fields alphabetical) because the reference SDK never finished
installing in our environment, and the AID spec explicitly allows offline
derivation. Ten agents carry one.

**Not claiming:** no A2A or ACP negotiation, no UCP manifest, and the recurring
trial charge is a client loop over x402 rather than `ScheduleCreateTransaction`.

## ENS

**What a name does here.** Everywhere a person or an agent appears — a studio
on a listing, a line on a revenue split, the buyer named in a sale notification
— we show an ENS name instead of `0x71C7…3e4F`. One resolution order, ENS
first, everywhere.

**For an agent it matters more than that.** An autonomous buyer with a name is
a participant in the market. The same wallet with a raw address is a script.
`best-agent.cgs-sanctuary.eth` outbidding you for a game reads as a rival;
`0x5325…8e1d` reads as a bug. Two exist today, each pointing at the **agent's
own** Hedera account, not its owner's:

| Name | Agent account |
|---|---|
| `suved.cgs-sanctuary.eth` | `0.0.10475095` |
| `best-agent.cgs-sanctuary.eth` | `0.0.10475992` |

And because the name lives on Sepolia rather than in our database, it keeps
resolving whether or not CGS exists. Same argument the GameKey makes about
ownership and the topic makes about price history.

**How it is built.** ENS's own `PermissionedRegistry`, our own instance of it,
deployed through ENS's `VerifiableFactory` as a UUPS proxy we own, called
directly with viem. The parent `cgs-sanctuary.eth` went through a real
commit–reveal registration including the full 60-second `MIN_COMMITMENT_AGE`
wait. Permissions are ENS's Enhanced Access Control role bitmaps, copied
verbatim from `RegistryRolesLib.sol`. No custom registrar and no custom
Solidity anywhere.

A subname owner gets `ROLE_SET_RESOLVER | ROLE_RENEW`: enough to point their
name where they like and keep it renewed, not enough to unregister it or move
it out from under the platform. Studios and agents share one flat namespace, so
they compete for the same label and one availability check answers for both.
Names are write-once — renaming would mint a second name and leave the first
pointing at the same wallet.

**Honest about the edge:** we register and display names, we do not resolve
them in our own code paths. Nothing looks a studio up by name. The registration
and the ownership are real; the resolution is not doing work yet. We also use
ENS's standard resolver rather than a per-subname Permissioned Resolver, and we
do not do wildcard resolution.

## Privy

**What a player sees:** an email field. They type an address, click the link,
and they have a wallet. No seed phrase, no extension, no "connect wallet"
screen before they are even sure they want the game. Thirty seconds later they
are playing something they bought.

**What a collaborator sees:** an email that says *"Tin Roof added you. You are
on 40% of Deadzone. Your share is already set."* They click it, sign in, and
every payment held for them since the game launched arrives at once — paid
straight to their new EVM address, which under HIP-542 creates their Hedera
account as a side effect of the payment itself. They never installed anything.
That is the flow that makes splitting revenue with a jam team work at all.

**The design decision worth the integration.** A Hedera transfer has to be
built and frozen server-side and signed client-side. So the server freezes it,
the **browser** signs the hashes with
`provider.request({ method: 'secp256k1_sign' })` — the raw-hash primitive with
no Ethereum message prefix, which is exactly what Hedera needs — and the server
settles.

The alternative was delegated signing: asking every buyer to grant a shop
standing permission to move their money *before their first purchase*. That is
a much larger thing to ask than "approve this purchase", so we did not ask it.
**The server never holds a buyer's key.**

**Two kinds of wallet, one clear line.** User-owned embedded wallets for
buyers, which we cannot sign for. App-owned wallets from
`privy.wallets().create({ chain_type: 'ethereum' })` for autonomous agents,
which we can — via `privy.wallets().rpc(id, { method: 'secp256k1_sign' })` —
because we created them. That is how an agent buys a game and pays for its own
reasoning at 3am with nobody watching.

Auth is `verifyAccessToken` on every request, verified locally with no network
hop.

**Honest gap:** we do not integrate Privy's funding or onramp. A buyer funds by
receiving USDC from elsewhere, and it is the one place onboarding still feels
like crypto.

---

## Where to look in the code

`src/agent/watcher.ts` — the agent reading HCS and deciding.
`src/services/hedera/hts.ts` — the GameKey, and the keys it deliberately lacks.
`src/services/games/fulfil.ts` — the atomic split.
`src/services/x402/` — the payment rails all three gated routes share.

## What's not built

No resale or secondary market, no refunds, no editing splits after publish, no
identity verification on upload, no adult content. Each cut deliberately.

## What's not done

Not deployed — all of the above runs on testnet against live infrastructure,
but there is no public URL yet. Privy's onramp is not integrated. Email needs
an SPF record and a real `APP_URL`. No CSAM provider is wired, so uploads fail
closed by design.
