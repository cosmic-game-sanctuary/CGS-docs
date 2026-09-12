# Cosmic Game Sanctuary

A storefront for browser-playable indie games where the payment rail, the
ownership record and the revenue split are public infrastructure, so no card
network decides what can be sold and no company decides what a buyer keeps.

📺 **[Four-minute demo](https://youtu.be/WyJehf7Vgb4)** · 📖 **[How it works](ARCHITECTURE.md)**

Built for ETHOnline 2026 on **Hedera**, with **ENS** and **Privy**. Team of two.

---

## Why

In July 2025 itch.io and Steam deindexed adult-tagged games overnight, after
Visa and Mastercard threatened to cut payment processing. LGBTQ-themed and
merely suggestive titles were swept up with the rest. Changing payment
processor does not help, because every processor routes back to the same two
card networks.

So we took the processor out of the loop.

This is not a web3 game and not GameFi. There is no token, nothing to
speculate on, no play-to-earn. Ordinary indie games where the ledger is a
payment rail the buyer never thinks about.

## What works

**Buy and play.** Sign in with an email, get a wallet, buy a game, play it in
the same tab about four seconds later.

**Try before you buy.** Play by the minute, charged from your own wallet as you
go, stopped the moment you close the tab. Every cent comes off the price if you
buy it.

**Ship with a team.** Split revenue with people who have only an email address.
They get a message saying *"you are on 40% of Deadzone"*, click it, and every
payment held for them arrives at once. The whole team is paid in one atomic
transaction: everyone or nobody.

**Let an agent buy for you.** Fund a small wallet, name your price, walk away.
Two games on sale and money for one, and it waits, compares, and chooses —
paying for its own reasoning out of its own wallet.

Plus the ordinary storefront: reviews from verified buyers, developer replies,
public profiles, cloud saves, sales with a verifiable countdown, earnings and
withdrawals, moderation, and studio rosters with real permissions.

## What's true in the code, not just the pitch

**The agent reads a public topic, not our database.** Anyone could write a
competing agent against the same feed and we could not stop them.

**Splits cannot change after publish.** No edit endpoint, no admin override.
itch.io's issue for multi-dev payout splits has been open since 2016.

**Delisting does not revoke anyone's copy.** GameKey tokens are minted with no
wipe key, no freeze key, no pause key and no admin key. We are structurally
unable to take a purchase back, and that takes ten seconds to verify on
HashScan.

**Price history is a public topic.** Every change is timestamped and checkable.
No storefront that owns its own price database can say that credibly.

## Repos

| | |
|---|---|
| [CGS-server](https://github.com/cosmic-game-sanctuary/CGS-server) | API, chain integration, the agent |
| [CGS-client](https://github.com/cosmic-game-sanctuary/CGS-client) | storefront, player, upload UI |
| CGS-docs | [ARCHITECTURE.md](ARCHITECTURE.md) · [INTEGRATION.md](INTEGRATION.md) · [PROGRESS-LOG.md](PROGRESS-LOG.md) |

## Not building

Resale and secondary markets, refunds, editing splits after publish, identity
verification on upload, adult content, achievements, native builds. Each was
cut for a reason rather than left undone.
