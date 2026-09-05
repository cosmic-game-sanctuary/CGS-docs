# Cosmic Game Sanctuary

A storefront for browser-playable indie games where payment and ownership settle on a public distributed ledger, so no payment network decides what can be sold and no company decides what a buyer keeps owning.

Built for ETHOnline 2026 on Hedera, with ENS and Privy.

## Why

In July 2025 itch.io and Steam deindexed adult-tagged games overnight, after Visa and Mastercard threatened to cut payment processing. LGBTQ-themed and merely suggestive titles were swept up with the rest. Switching payment processors doesn't fix it, because every processor routes back to the same two card networks underneath.

CGS takes the payment processor out of the loop. Payment, ownership, revenue splits and the catalog itself settle on-chain, where anyone can verify them — including software that isn't ours.

It isn't a web3 game and it isn't GameFi. There's no token to buy, nothing to speculate on, no play-to-earn. These are ordinary indie games and the ledger is a payment rail the buyer shouldn't have to think about.

## How it works

A dev uploads a build, sets a price, adds teammates by email and sets revenue splits. On publish the splits lock and the listing is written to a public consensus topic.

A buyer pays with a wallet they got by logging in with an email. Payment runs over [x402](https://x402.org) — HTTP 402, answered by a signed payment on retry — and settles through the Blocky402 facilitator. A GameKey token lands in the buyer's account and the game boots in the same tab.

Revenue reaches the whole dev team in one atomic transaction. Either everyone gets paid or nobody does, so there's no chasing a teammate for your share.

Separately, a buyer can fund a small dedicated wallet and point an agent at a game. The agent watches the public listings topic through the Mirror Node, and when the price hits the trigger it buys, with no human present. Its spending cap is its balance — it can't overspend what isn't there.

## Three things that are true in the code, not just the pitch

**The agent reads the public topic, not our database.** That's the difference between an app with a bot in it and a public action anyone could independently build on.

**Splits can't be changed after publish.** No edit endpoint, no admin override. itch.io's issue for multi-dev payout splits has been open since 2016.

**Delisting doesn't revoke anyone's copy.** GameKey tokens are minted with no wipe key, no freeze key and no pause key. We're structurally unable to take a purchase back, and you can check that on HashScan rather than take our word for it.

## Repos

- [CGS-server](https://github.com/cosmic-game-sanctuary/CGS-server) — API, chain integration, agent
- [CGS-client](https://github.com/cosmic-game-sanctuary/CGS-client) — storefront, player, upload UI
- CGS-docs — this repo, shared status in [PROGRESS-LOG.md](PROGRESS-LOG.md)

## Not building

Resale and secondary markets, refunds, editing splits after publish, identity verification on upload, adult content, achievements, native builds. Each was cut for a reason rather than left undone.
