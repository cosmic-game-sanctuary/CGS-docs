# Cosmic Game Sanctuary (CGS)

> **Steam-like convenience. Unrevocable on-chain game ownership. Powered by native USDC on Arc Mainnet.**

[**🚀 Live on Arc Mainnet**](https://cgs.blackslate.me) · [**🧪 Testnet Twin (With Faucet)**](https://testnet.cgs.blackslate.me) · [**🎥 Watch Demo Video**](#)

---

## The Problem We're Solving

In July 2025, Visa and Mastercard pressured Steam and itch.io into pulling thousands of indie games in a single night. Even on PlayStation or Xbox, paying full price only buys you a revocable license—and switching payment processors doesn't help when every processor answers to the same two card networks.

We started building **Cosmic Game Sanctuary (CGS)** as a two-person team at ETHOnline 2026 on Hedera to give players true, one-time game ownership on-chain, where nobody—including us—can take back what a player bought.

The catch? Traditional Web3 stores have way too much friction for normal gamers. Asking someone to buy a separate volatile token just to pay gas kills a $2 indie game purchase before it starts. When **Arc launched its Mainnet**, everything clicked. We ported CGS in full to Arc because it uses **one currency—USDC, 1:1 pegged to the US dollar—for both buying games and paying sub-cent gas fees.**

---

## Try It Yourself in 2 Minutes

1. **Sign in with Email:** Open **[cgs.blackslate.me](https://cgs.blackslate.me)** and log in with any email via Privy—no browser extensions or seed phrases required.
2. **Buy a Game with USDC:** Pay for the game and the tiny gas fee (~$0.007) from the same stable USDC balance. Funds settle directly into that game's dedicated on-chain vault.
3. **Verify on the Explorer:** Open the vault on **[explorer.arc.io](https://explorer.arc.io)**. Collaborator revenue splits are permanently fixed in the contract's constructor—not even we have a function to change them.

*Want to test the entire flow without spending real money? **[testnet.cgs.blackslate.me](https://testnet.cgs.blackslate.me)** is an exact twin with a built-in faucet button.*

---

## Core Features

* **Unrevocable GameKeys (ERC-721):** Every purchase mints a `GameKey` with zero admin, pause, burn, or freeze functions. Delisting a game from the storefront never affects anyone's existing copy.
* **Agents That Buy While You Sleep (ERC-8004 + x402):** Pick your wishlist games and target prices, fund your agent with USDC, and walk away. Backed by an **ERC-8004** identity and hard spending caps enforced at signing, the agent monitors on-chain listings for sales. If two wishlisted games go on sale and compete for your remaining budget, the agent pays an AI model over **x402** to choose the better deal.
* **Pay-Per-Minute Game Trials (x402 + Circle Gateway):** Test games by the minute using Circle Gateway nanopayments. Every cent you spend during a trial is automatically deducted from the full purchase price if you buy the game.
* **Immutable Developer Splits (95% to Devs):** Publishing a game deploys a dedicated vault with fixed collaborator shares. Devs can add teammates by email even before they have a wallet—their share waits safely in the vault.

---

## Why Arc Mainnet Makes This Possible

* **Single-Token UX (USDC for Games + Gas):** Gamers price games in US dollars and pay gas in the exact same USDC, eliminating volatile gas tokens.
* **5% Flat Fee vs. Steam’s 30%:** Because settling a sale on Arc costs roughly **$0.007 in gas** regardless of how many collaborators split the revenue, a hard-coded 5% platform fee is enough to run sustainably—giving indie studios a real reason to publish on Arc.
* **Built Specifically for Arc’s Rules:**
  * **Pull-based payouts:** Prevents a single blocklisted address from freezing payouts for other collaborators on the same game.
  * **Gas-sponsored first claims:** New developers who hold $0 USDC don't get stuck—we sponsor the gas to release their very first earnings.
  * **18-decimal native vault math:** Vault accounting runs on Arc's 18-decimal native balance so fractional cents never get stranded as dust.

---

## Live on Arc Mainnet

| Contract | Verified Arc Mainnet Address | Description |
| :--- | :--- | :--- |
| **`GameRegistry`** | `0x864a…4C6c` | Stores game metadata, pricing, and active sale states |
| **`GameKey`** | `0x3938…11B6` | Unrevocable, admin-free ERC-721 ownership license |
| **`VaultFactory`** | `0xd2ca…95d0` | Deploys per-game USDC revenue-split vaults |

---

## Repository Structure

* **[`CGS-contracts`](https://github.com/cosmic-game-sanctuary/CGS-contracts)** — Solidity smart contracts & Foundry deployment/verification scripts for Arc Mainnet.
* **[`CGS-server`](https://github.com/cosmic-game-sanctuary/CGS-server)** — TypeScript backend handling x402 trial streams, Circle Gateway settlement, and ERC-8004 agent execution.
* **[`CGS-client`](https://github.com/cosmic-game-sanctuary/CGS-client)** — Storefront web app with Privy integration and developer publishing portal.
* **[`CGS-docs`](https://github.com/cosmic-game-sanctuary/CGS-docs)** — Detailed system architecture (`ARCHITECTURE.md`) and protocol specifications.