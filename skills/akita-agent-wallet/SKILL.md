---
name: akita-agent-wallet
description: Operate live Akita ARC-58 agent wallets through the akita-agent-wallet MCP server—balances, payments, DeFi, NFTs, governance/social, and rewards. Use when the user asks to check a wallet, send ALGO/ASA, stake, vote, post socially, claim rewards, or otherwise act on an already-configured agent wallet via MCP. Prefer this skill for live ops; use sibling skills (akita-wallet-sdk, akita-arc58-plugin-dev) for writing SDK or plugin code. Strong triggers include "get_wallet_balance", "send_payment", "ENV_FILE", "Intermezzo", "agent wallet", "escrow", "MCP wallet".
---

# Akita Agent Wallet (MCP)

Teach Cursor agents how to drive **live** ARC-58 agent wallet operations through the `akita-agent-wallet` MCP plugin. This skill is for **runtime actions**, not for authoring TypeScript SDK or plugin code.

## Prerequisites

Before calling any MCP tool:

1. **`ENV_FILE` is configured** — Plugins → Configure → absolute path to the `.env` produced by `npx @akta/agent-wallet-cli` / `pnpm setup`. It must include Intermezzo/Vault credentials, `WALLET_APP_ID`, and `ESCROW_NAME`.
2. **Intermezzo is running** — the Docker stack behind the agent wallet must be up; MCP calls fail if Vault/Intermezzo is unreachable.
3. **MCP server is loaded** — `akita-agent-wallet` from this plugin (`npx -y @akta/agent-wallet-mcp`, or a local `node …/packages/mcp/dist/index.js` override).

If a tool errors with missing env or connection refused, fix setup first—do not invent credentials or private keys.

## Safety

- **Allowances govern spend.** Escrows enforce flat / window / drip limits. Never assume unlimited send; check balance and allowance before large transfers.
- **No private keys in chat or code.** Signing happens inside Intermezzo/Vault. Never ask the user to paste mnemonics or raw keys into the prompt.
- **Confirm destructive or irreversible actions** (large payments, reclaim, admin changes) with the user before calling the tool.
- **Prefer read tools first** (`get_wallet_info`, `get_wallet_balance`) to establish context, then mutate.


## Sibling skills (when not to use this one)

| Need | Skill |
|------|-------|
| Live MCP wallet ops (this skill) | `akita-agent-wallet` |
| TypeScript `WalletSDK` / plugins client code | `akita-wallet-sdk` |
| Authoring or extending ARC-58 plugins | `akita-arc58-plugin-dev` |
| Social / governance product flows in code | `akita-social-stack` |
| Connect / auth integration | `akita-connect` |

Do **not** generate SDK boilerplate when the user wants a live balance check or payment—call MCP instead.

## Action tables

Tool names below are the MCP surface; pass only documented arguments. If a tool is unavailable, list MCP tools and adapt—do not fake results.

### Wallet info

| Action | Tool | Notes |
|--------|------|-------|
| Wallet / escrow summary | `get_wallet_info` | App id, escrow name, network |
| Balances | `get_wallet_balance` | ALGO and ASA holdings |
| Allowances | `get_allowances` / related read tools | Confirm spend headroom before send |

### Payments

| Action | Tool | Notes |
|--------|------|-------|
| Send ALGO or ASA | `send_payment` | Respect asset id, amount units, receiver |
| Opt-in ASA | `opt_in` / payment-plugin equivalent | Required before receiving ASA |
| Reclaim / return funds | reclaim-related tools | Admin/escrow policy dependent |

### DeFi

| Action | Tool | Notes |
|--------|------|-------|
| Stake / unstake | staking MCP tools | Pool ids from network config |
| Swap | HyperSwap-related tools | Check slippage / allowance |
| Marketplace / auction / raffle | corresponding MCP tools | Read listing state before bid/buy |

### NFT

| Action | Tool | Notes |
|--------|------|-------|
| Mint / transfer ASA NFT | mint / transfer tools | Confirm decimals and clawback settings |
| Marketplace list / buy | marketplace tools | Price in base units |

### Governance / social

| Action | Tool | Notes |
|--------|------|-------|
| DAO propose / vote | DAO / poll tools | Use `akita-social-stack` for product semantics |
| Social post / follow | Social plugin tools | On-chain social actions via escrow |

### Rewards

| Action | Tool | Notes |
|--------|------|-------|
| Claim / view rewards | rewards tools | Check eligibility before claim |
| Subscriptions | subscription tools | Period and asset constraints |

## Typical agent workflow

1. Verify MCP server + `ENV_FILE` / Intermezzo readiness (read `get_wallet_info`).
2. `get_wallet_balance` (and allowances if spending).
3. Explain the planned action and amounts to the user when value moves.
4. Call the mutating tool once; report tx / result ids from the tool response.
5. Re-read balance if the user needs confirmation.

## Common mistakes

- **Missing or relative `ENV_FILE`** — must be an absolute path in plugin config.
- **Intermezzo not running** — symptoms look like MCP timeouts or auth failures; start Docker first.
- **Wrong units** — ALGO microAlgos (1e6); ASA amounts use asset decimals.
- **Sending without opt-in / allowance** — receiver must be opted in; escrow allowance must cover the spend.
- **Writing SDK code instead of calling MCP** — this skill is for live ops; hand off to `akita-wallet-sdk` only when the user wants application code.
- **Requesting private keys** — never; the agent wallet design keeps keys in Vault.
- **Assuming mainnet vs testnet** — trust `get_wallet_info` / `.env` network; do not hardcode app ids.

## References

- Docs: https://docs.akita.community
- Agent wallet: https://github.com/akita-protocol/agent-wallet
- Skills catalog: https://github.com/akita-protocol/skills
- Cursor plugins: https://cursor.com/docs/reference/plugins
