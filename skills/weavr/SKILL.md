---
name: weavr
description: Create and manage onchain portfolios from a thesis.
version: 0.1.0
author: weavr
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [weavr, portfolio, solana, defi, mcp]
    requires_toolsets: [terminal]
    category: finance
    required_environment_variables: [PAYBOX_CONFIG_DIR, PAYBOX_CREDENTIAL_ID, PAYBOX_CLI, WEAVR_SIGN_TOOL]
---

# weavr

Weavr creates onchain portfolios from a thesis. The weavr MCP server is attached to this agent; this host exposes its tools as `mcp__weavr__<name>` (for example `mcp__weavr__list_assets`), and the names below are the bare ones. The wallet is a separate command-line tool run with the terminal tool (see Wallet). Say asset, portfolio, thesis, rebalance, deposit, shares, agent and onchain.

## When to use
The user wants to create a portfolio, deposit into one, check one, or rebalance one.

## Procedure
Flow: the user types a thesis → list_assets (never before they have typed one) → suggest_mix if they ask for a recommendation → ask for a name, a ticker and any rules → simulate_portfolio → show the result → create_portfolio once they confirm → hand the result to the wallet tool → the wallet tool waits until live.

1. Never invent a name or ticker; use theirs exactly as typed.
2. Never ask the user for a wallet address: the wallet tool provides the creator. Run `node $WEAVR_SIGN_TOOL --address` and pass the address as `creator` to create_portfolio.
3. Keep create_portfolio to 4 assets or fewer on this host (see Limits). Two or three assets are one signature; four are two, which the wallet tool handles in one run.
4. After create_portfolio returns a `deploymentId`, run `node $WEAVR_SIGN_TOOL --deployment <deploymentId>` with the terminal tool. It signs and waits; its output is the portfolio status. When it prints `"status":"live"`, tell the user the portfolio is live and give the `url`. Do not call await_portfolio yourself.
5. Deposits: run `node $WEAVR_SIGN_TOOL --deposit <ticker> --amount <usd>`. It builds, signs and sends; report what it prints. Do not call build_deposit or send_signed yourself.
6. Never print walletPayload, never ask whether they signed, and never ask them to check again. Wallets sign but do not send: the wallet tool passes what it signed to weavr, which sends it and watches the chain.
7. Reads (get_portfolio, list_portfolios, get_asset, the histories, simulate_rebalance) need no wallet.

Fees: 0.40% round trip (0.20% in, 0.20% out), 60% of the fee stream to the creator, no management fee, 5% idle, 2% rebalance band. Amounts are in USD; percents are percents.

## Wallet
The wallet is the command `node $WEAVR_SIGN_TOOL ...`, run with the terminal tool only (never with code execution, which strips the variables it needs). It reads its own keys from files; you never see or handle them.

- `--address` → `{"address": "..."}`: the creator or depositor address to pass to weavr.
- `--deployment <deploymentId>` → signs the create and waits until it is live; prints `{"status":"live","mint":...,"url":...}` or an error.
- `--deposit <ticker> --amount <usd>` → builds, signs and sends a deposit; prints `{"status":"confirmed",...}` or an error.

It refuses transactions it should not sign. Its errors mean: `LEGACY_ONLY` = too many assets for this wallet (use 4 or fewer and try again); `WALLET_DECLINED` = the wallet did not sign, tell the user and stop; `FOREIGN_PROGRAM` or `WRONG_PAYER` = the transaction was not for this user, stop and tell the user; `BUSY` = another signing run is in progress, wait a moment and run the same command once more.

## Pitfalls
- `sign_again` from the wallet tool means the transaction expired before it was signed; run the same `--deployment` command once more.
- `OPERATOR_BUSY` or `AWAIT_IN_PROGRESS` from weavr: wait a moment and run the same command once more.
- `INSUFFICIENT_SOL`: the wallet needs more SOL for the network cost; tell the user the amount and stop.
- Only act in a private chat with the wallet's owner. In a group, explain that portfolio actions are private.

## Limits on this host
- PayBox signs portfolios with up to 4 assets. If the user insists on more, explain that this wallet cannot sign it yet.
- One signature per create up to three assets, two at four; deposits are one signature each.

## Verification
The create is done when the wallet tool prints `"status":"live"` with a `url`. A deposit is done when it prints `"status":"confirmed"`.
