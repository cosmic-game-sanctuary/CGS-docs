# Cosmic Game Sanctuary

**An indie game store where an agent buys games for you while you sleep.** Revenue
splits are enforced by a contract no one can edit, games are rentable by the
minute, and you hold what you buy in your own wallet.

It runs on [Arc](https://www.arc.io/), Circle's EVM L1 where USDC is the gas
token. No card network decides what's sellable, and no company decides what a
buyer keeps owning.

🟢 **Live on Arc mainnet: [cgs.blackslate.me](https://cgs.blackslate.me)** ·
🧪 **Try it free on testnet: [testnet.cgs.blackslate.me](https://testnet.cgs.blackslate.me)**
(sign in with any email, then use **Add funds** for free test USDC)

🕹️ **[Try it](https://cgs.blackslate.me)** · 📖 **[How it works](ARCHITECTURE.md)** · 📜 **[Contracts](https://github.com/cosmic-game-sanctuary/CGS-contracts)**

Built by a team of two for the [Arc Microgrants](https://dorahacks.io/hackathon/arc-microgrants/detail)
program. It started at ETHOnline 2026 on Hedera
([that demo](https://youtu.be/WyJehf7Vgb4)) and was ported to Arc in full.

---

## Why

In July 2025 itch.io and Steam deindexed adult-tagged games overnight, after
Visa and Mastercard threatened to cut payment processing. LGBTQ-themed and
merely suggestive titles were swept up with the rest. Changing payment
processor does not help, because every processor routes back to the same two
card networks.

So we took the processor out of the loop.

This is not a web3 game and not GameFi. There is no token, nothing to
speculate on, no play-to-earn. Ordinary indie games where USDC on Arc is a
payment rail the buyer never thinks about.

## What works

**Buy and play.** Sign in with an email, get a wallet, buy a game, play it in
the same tab seconds later. The buyer signs once and pays no gas: Circle's
facilitator settles the payment and pays for it.

**Try before you buy.** Play by the minute, paid from your own wallet as you go
through Circle Gateway nanopayments, stopped the moment you close the tab. Every
cent comes off the price if you buy it.

**Ship with a team.** Split revenue with people who have only an email address.
The split is written into the game's own vault contract at publish, and nobody
can change it afterwards, including us. Each person claims their share straight
from the vault. We pay the gas for that, so a first-time developer holding no
USDC can still be paid.

**Let an agent buy for you.** Fund a small wallet, name your price, walk away.
Two games on sale and money for one, and it waits, compares, and chooses,
paying for its own reasoning out of its own wallet. Its spending ceiling is
published on chain against its ERC-8004 identity and enforced when it signs.

Plus the ordinary storefront: reviews from verified buyers, developer replies,
public profiles, cloud saves, sales with a deadline, earnings and withdrawals,
moderation, and studio rosters with real permissions.

## What's true in the code, not just the pitch

**The money never passes through us.** A purchase pays the game's `SplitVault`
directly. The vault splits whatever arrives by fixed shares, and only a payee
can be paid their share.

**Splits cannot change after publish.** The vault has no owner and no setter.
itch.io's issue for multi-dev payout splits has been open since 2016.

**Delisting does not revoke anyone's copy.** `GameKey` is an ERC-721 with no
admin, pause, burn or freeze function. We are structurally unable to take a
purchase back, and the verified source on the explorer shows it.

**The agent reads the public chain, not our database.** It learns about
listings and price changes from `GameRegistry` events via `eth_getLogs`. Anyone
could write a competing agent against the same events and we could not stop
them.

**Our cut is 5%, and it's in the contract.** It's fixed per game at publish and
visible before the developer agrees, so it can never be raised later. Steam
takes 30%, itch.io 10%.

## Repos

| | |
|---|---|
| [CGS-server](https://github.com/cosmic-game-sanctuary/CGS-server) | API, chain integration, the agent |
| [CGS-client](https://github.com/cosmic-game-sanctuary/CGS-client) | storefront, player, upload UI |
| [CGS-contracts](https://github.com/cosmic-game-sanctuary/CGS-contracts) | `GameRegistry`, `GameKey`, `SplitVault`, `VaultFactory` |
| CGS-docs | [ARCHITECTURE.md](ARCHITECTURE.md) · [INTEGRATION.md](INTEGRATION.md) · [PROGRESS-LOG.md](PROGRESS-LOG.md) |

## Not building

Resale and secondary markets, refunds, editing splits after publish, identity
verification on upload, adult content, achievements, native builds. Each was
cut for a reason rather than left undone.
