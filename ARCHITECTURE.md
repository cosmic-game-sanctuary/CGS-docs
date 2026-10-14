# How it works

CGS is a storefront for browser-playable indie games that runs on
[Arc](https://www.arc.io/), Circle's EVM L1 where USDC is the gas token.

Three things here are unusual: **revenue splits enforced by a contract** that pays
a whole team without the money ever touching us, an **autonomous agent** with its
own wallet that chooses which games to buy, and **trials metered by the minute**
over x402 and Circle Gateway.

Everything they depend on, the money, the ownership token and the price history,
lives on a public chain, so none of it needs us online or honest.

🕹️ **[Try it](https://cgs.blackslate.me)** · 📖 [The pitch](README.md) · 🔌 [API contract](INTEGRATION.md) · 📜 [Contracts](https://github.com/cosmic-game-sanctuary/CGS-contracts)

**Jump to:** [Buying](#buying-a-game) · [Publishing](#publishing) ·
[The agent](#the-agent) · [Paid trials](#paid-trials) ·
[On-chain reference](#whats-on-chain) · **Sponsors:** [Circle and Arc](#circle-and-arc) ·
[ENS](#ens) · [Privy](#privy) · [Gaps](#whats-not-done)

---

## Stack

| | |
|---|---|
| Backend | Node, TypeScript, Express, Postgres via Drizzle |
| Frontend | Vite, React 19, Tailwind v4 |
| Chain | Arc mainnet (chain 5042), with a full testnet twin (5042002) |
| Contracts | `GameRegistry`, `GameKey`, `SplitVault`, `VaultFactory`, written for this project |
| Payments | [x402](https://x402.org) over HTTP, settled by Circle's Facilitator; trials through Circle Gateway nanopayments |
| Agent identity | [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) IdentityRegistry, predeployed on Arc |
| Wallets + login | Privy |
| Names | ENSv2 on Sepolia |
| Builds | IPFS via Pinata |
| Agent reasoning | Groq, `openai/gpt-oss-20b`, strict JSON schema |

---

## Buying a game

You sign in with an email. Privy makes you a wallet. You press buy, and seconds
later the game is running in the same tab. You never need to hold gas.

The request to download the build comes back `402 Payment Required` with terms
attached, and `payTo` is **the game's own vault**:

```
browser                  server              Circle Facilitator        Arc
   │ GET /games/:id/download │                        │                  │
   │ ───────────────────────>│                        │                  │
   │ 402 + terms (payTo = vault)                      │                  │
   │ <───────────────────────│                        │                  │
   │ Privy eth_signTypedData_v4                       │                  │
   │ (EIP-3009 transferWithAuthorization, offchain)   │                  │
   │ retry + signed authorization                     │                  │
   │ ───────────────────────>│ ─────── settle ──────> │ ───────────────> │
   │                         │                        │  USDC → vault    │
   │                         │ <───── tx hash ─────── │                  │
   │                         │ mint GameKey ────────────────────────────> │
   │ build + playUrl         │                        │                  │
   │ <───────────────────────│                        │                  │
```

The buyer signs once, off chain, with their own key. Circle's facilitator
broadcasts the transfer, **pays the settlement gas**, and screens both parties.
The USDC goes from the buyer straight into the game's vault. The only gas we pay
for a sale is minting the buyer's `GameKey`, about $0.004.

Arc has deterministic finality, so settlement is final on inclusion and there is
no reorg path to handle.

---

## Publishing

Make a studio, optionally claim a name. Drop a zip, which unpacks and runs in
your browser before it uploads anywhere, so you see the real build. Set a
price. Split the revenue.

**You split with people who have only an email address.** Type
`sam@gmail.com`, 40%, done. Sam is on the credits and earning from the first
sale, having never touched a wallet. When Sam signs in, their share is already
in the vault under their new wallet.

On publish, one transaction path does three things: `VaultFactory` deploys a
**`SplitVault`** for the game, with every payee's share and the platform's 5%
fixed in its constructor, `GameRegistry` emits `Listed`, and the build is pinned
to IPFS twice (directory for provenance, zip for delivery).

There is no endpoint to edit a split afterwards, and the vault has no setter. The
vault accounts by arithmetic, not by hooks: `owed = received × share`,
`claimable = owed − claimed`. It needs no call when money arrives, so a sale
priced differently, discounted, or paid in trial credit is split correctly with
no extra code.

**Claiming.** A payee calls `claim()`. Because gas on Arc is USDC, a developer
whose first earnings are still inside the vault holds nothing to pay with, so the
vault also has `claimFor(address)`: anyone may pay the gas, and the money can only
go to that payee. We use it to release shares for people who ask, at about
$0.003 each.

**Why one vault per game.** An EIP-3009 transfer credits a contract without
calling it, so a payment carries no game id. `payTo` is therefore the game
identifier, and a shared vault could not tell games apart.

---

## The agent

Here is the scenario it exists for, which we have actually run:

> Two games on your list, both $1.00, both on sale. Your agent holds $1.20. It
> can have one.

A dumb bot buys whichever went on sale first. Ours waits.

It has its own app-owned Privy wallet and its own **ERC-8004 identity**, and
optionally its own ENS name. You give it games and a ceiling for each, then fund
it. The balance is the total budget, and the per-purchase ceiling is enforced
where the authorization is signed.

**It reads the public chain.** It polls `GameRegistry` events with `eth_getLogs`
(`Listed`, `PriceChanged` with the sale's deadline, `BuildUpdated`, `Delisted`).
Not our database, not a webhook from us. Someone else could write a competing
agent against the same events tomorrow and we could not stop them, which is the
point.

**It decides at the wire.** A sale is open until it ends, so buying early gains
nothing and gives up the chance to compare. The agent schedules itself shortly
before the soonest deadline it is watching, and decides then, having seen
everything that arrived in the meantime. A price drop usually just moves the
alarm clock. If a buy fails, it retries within the same window.

**When the choice is genuinely contested, it pays to think.** Most rounds are
unambiguous and no model is involved. When the money is actually spoken for, it
calls a second x402-gated endpoint and is charged per verdict, from its own
wallet:

```
GET /api/agent/verdict  →  402 + terms  →  agent signs  →  settle  →  answer
```

That endpoint takes no request body. What it reasons about comes from *who
paid*: the settled payment names an account, and the wants and balance are
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

The buyer deposits a little USDC into Circle's `GatewayWallet` once. After that
the meter buys the next minute shortly before the current one runs out, each a
real x402 payment signed quietly by your own wallet and settled through Gateway's
batching, which costs the buyer no gas. Close the tab and it stops. The loop
lives in the page, so there is no server timer to cancel.

Everything you spent trying it comes off the price if you buy, summed live from
the payment rows rather than tracked in a counter that could drift.

When the time is up the game frame is **removed from the page**, not covered. A
cross-origin build has no pause button to press, so unmounting it is the only
thing that genuinely stops it.

**Where trial money goes, stated plainly.** Gateway credits a payee a balance
*inside Gateway*, and a vault has no key to withdraw it. So trial chunks are paid
to the platform operator, and a relay forwards each game's completed chunks into
its vault as a normal same-chain transfer once Circle's batch lands (minutes,
batched about every 16). The relay is a public transaction naming the vault as
recipient, and the developer's earnings page shows the money as *on its way*
meanwhile. For those minutes trial money is held by us. **Purchases never are.**

---

## What's on chain

All public, none of it needs our permission to read.

| | Arc mainnet (5042) | Arc testnet (5042002) |
|---|---|---|
| GameRegistry | [`0x864a…4C6c`](https://explorer.arc.io/address/0x864a562db243C46c5825cBd4224069361b734C6c) | [`0xE7aa…490e`](https://explorer.testnet.arc.io/address/0xE7aa837c45caE001bAc6f026B23edf6e96Ee490e) |
| GameKey | [`0x3938…11B6`](https://explorer.arc.io/address/0x393889a68b6e78F0138b1bb240e9b3fD151111B6) | [`0xaDC0…9c41`](https://explorer.testnet.arc.io/address/0xaDC0757f81680c7014FbCE73d664653c5c629c41) |
| VaultFactory (operator-only) | [`0xd2ca…95d0`](https://explorer.arc.io/address/0xd2ca652AbAbbA2E1D6Bab2FFA950BC66e73F95d0) | [`0xCAE4…3BfB`](https://explorer.testnet.arc.io/address/0xCAE4b6037b0AbD416459bD88ae90903a64293BfB) |
| SplitVault | one per game, deployed at publish | [example](https://explorer.testnet.arc.io/address/0x0F4cbd4cd4C80868459865D996A62F6AfA0ac760) |
| Settlement asset | native USDC, `0x3600…0000` (6 dp ERC-20 view) | same |
| Agent identity | ERC-8004 `0x8004A169…a432` | `0x8004A818…BD9e` |

Source is verified on [explorer.arc.io](https://explorer.arc.io), so the claims
below are readable, not asserted:

- `GameKey` has no owner, pause, burn or freeze function. Its only restricted
  call is `mint`, limited to a minter fixed at deploy with no setter.
- `SplitVault` has no owner and no way to change a split.
- `GameRegistry` can announce but cannot touch a key or a vault.
- `VaultFactory` deploys a vault only for the operator, so nobody can pre-deploy
  a vault with their own payees for a game that has not published yet.

ENS lives on Sepolia and is separate from the Arc story:

| | |
|---|---|
| ENS subregistry | [`0xf721…2AF1`](https://sepolia.etherscan.io/address/0xf721c1d6883FC03466d7fCac8Eb9fdfd98432AF1), under `cgs-sanctuary.eth` |
| ENS resolver | [`0xfFC6…49e7`](https://sepolia.etherscan.io/address/0xfFC642Bea5522Bb07db8123b5a1C0f038fe649e7), our own PermissionedResolver proxy |

**The chain is the only thing we believe.** Receipts from our own database never
gate access: ownership is read from `GameKey`, balances from the vault.

---

# Sponsors

## Circle and Arc

**The short version:** the whole product is on Arc, and three different things
are paid for with x402 USDC, one of them an AI agent paying for its own reasoning.

**1. Buying a game**: `GET /api/games/:id/download`, settled by Circle's
Facilitator straight into a vault.

**2. The agent's own inference**: `GET /api/agent/verdict`. When the agent has to
choose between games it cannot both afford, it pays per verdict out of its own
wallet. Metered pay-per-call, not a subscription. The settlement transaction id
is stored on the decision row, so you can click from "it chose this" to the
transfer that paid for the thinking.

**3. A trial that meters itself**: `GET /api/games/:id/trial/chunks/settle`.
A human's wallet is charged for one minute of play at a time through Gateway
nanopayments, and stops when they close the tab.

**Arc-specific design, forced by the chain:**
- Gas is USDC, so the first earnings of a developer who holds none are
  unreachable by `claim()` alone. `claimFor` solves it without giving the payer
  any say over the money.
- The runtime blocklist reverts transfers to blocked addresses, so a push-payment
  split would let one blocked payee stop everyone being paid. Claims are pull, so
  each payee is isolated from the others.
- Arc's public RPC caps `eth_getLogs` below 10,000 blocks, so ownership checks use
  `GameKey`'s enumerable state rather than scanning logs.
- Native USDC accounting is 18 decimals and the ERC-20 view is 6, so the vault
  does its arithmetic on the native balance to avoid stranding fractions.

**What a sale costs us**: Circle pays settlement, so about $0.007 in gas
however many people are on the split. We take 5%, written into the vault.

## ENS

**What a name does here.** Everywhere a person or an agent appears, a studio on a
listing, a line on a revenue split, the buyer named in a sale notification, we
show an ENS name instead of `0x71C7…3e4F`. One resolution order, ENS first,
everywhere.

**For an agent it matters more than that.** An autonomous buyer with a name is a
participant in the market. The same wallet with a raw address is a script.

**An agent's name carries what it may spend.** Its per-purchase ceiling is not a
row in our database that you have to take our word for. It is a text record on
its own name, and the agent reads it back before it buys:

```
best-agent.cgs-sanctuary.eth
  addr           0x…                     the agent's Arc wallet
  cgs:role       agent
  cgs:maxSpend   1
  cgs:mode       autonomous
```

It is enforced rather than advertised: whatever the plan says and whatever the
model returned, nothing above the number published on chain gets bought. The
records are republished whenever the mandate changes, so a ceiling raised in the
app is a ceiling raised on chain, and a failed republish stops the buy rather
than spending past a stale number.

**How it is built.** ENS's own `PermissionedRegistry` and `PermissionedResolver`,
our own instances of both, deployed through ENS's `VerifiableFactory` as UUPS
proxies we own, called directly with viem. The parent `cgs-sanctuary.eth` went
through a real commit–reveal registration including the 60-second
`MIN_COMMITMENT_AGE` wait. Permissions are ENS's role bitmaps, copied verbatim
from `RegistryRolesLib.sol`.

A subname owner gets `ROLE_SET_RESOLVER | ROLE_RENEW`: enough to point their name
where they like and keep it renewed, not enough to unregister it. An agent's
subname gets `ROLE_RENEW` only and deliberately not `ROLE_SET_RESOLVER`, so the
account being limited cannot repoint its name at a resolver stating a different
number. Both facts are on each name's roles page.

**Verify it without us.** The names live on the ENSv2 deployment ENS runs for
ETHOnline, so ENS's own
[explorer](https://hackathon-deployment-portal-app.ens-cf.workers.dev/best-agent.cgs-sanctuary.eth)
reads them directly. Our API is not in that path.

**Honest about the edges:** ENS is on Sepolia, not Arc, and testnet and mainnet
share one namespace, so a name taken on one is taken on the other. We do not use
wildcard resolution or record aliasing.

## Privy

**What a player sees:** an email field. They type an address, click the link, and
they have a wallet. No seed phrase, no extension, no "connect wallet" screen
before they are even sure they want the game.

**What a collaborator sees:** an email that says *"Tin Roof added you. You are on
40% of Deadzone. Your share is already set."* They sign in and claim. They never
installed anything. That is the flow that makes splitting revenue with a jam
team work at all.

**The design decision worth the integration.** A purchase is an EIP-712 typed-data
signature over an EIP-3009 authorization, produced in the **browser** by the
buyer's own embedded wallet with `eth_signTypedData_v4`. The server builds the
terms and never holds a buyer's key. We did not ask buyers to grant a shop
standing permission to move their money before their first purchase.

**Two kinds of wallet, one clear line.** User-owned embedded wallets for buyers,
which we cannot sign for. App-owned wallets from
`privy.wallets().create({ chain_type: 'ethereum' })` for autonomous agents,
which we can sign for via `privy.wallets().rpc(...)` because we created them.
That is how an agent buys a game and pays for its own reasoning at 3am with
nobody watching.

Auth is `verifyAccessToken` on every request, verified locally with no network
hop. Withdrawals go from the user's own wallet to any address they choose.

**Honest gap:** we do not integrate Privy's funding or onramp on mainnet. A buyer
funds by sending USDC to their wallet address, and it is the one place
onboarding still feels like crypto. Testnet has a faucet button.

---

## Where to look in the code

`src/agent/watcher.ts`: the agent reading `GameRegistry` events and deciding.
`src/services/arc/`: the chain layer, vault, registry, keys, identity.
`src/services/arc/x402/`: the payment rails all three gated routes share, and
`trialRelay.ts`, which forwards trial money into vaults.
`CGS-contracts/src/`: the four contracts, short enough to read in one sitting.

## What's not built

No resale or secondary market, no refunds, no editing splits after publish, no
identity verification on upload, no adult content. Each cut deliberately.

## What's not done

Privy's onramp is not integrated, so mainnet users fund by sending USDC. No CSAM
scanning provider is wired; uploads follow the server's configured mode. The
contracts have not been independently audited, so mainnet prices are kept low.
Trial revenue is briefly held by the operator before its relay, as described
above. ENS stays on Sepolia.
