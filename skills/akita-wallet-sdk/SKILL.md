---
name: akita-wallet-sdk
description: Use the Akita `WalletSDK` from `akita-sdk/wallet` to create ARC-58 abstracted account wallets, install and invoke plugins, manage escrows, configure allowances (flat / window / drip), estimate costs, and pre-authorize transaction batches via execution keys. Use when writing TypeScript against the Akita wallet SDK, integrating a React or Node client, building a new dApp, or scripting plugin interactions. Strong triggers include "newWallet", "WalletFactorySDK", "addPlugin", "usePlugin", "addAllowances", "reclaimFunds", "build.usePlugin", "prepare.usePlugin", "fundsRequest", "executionKey", "isFlatAllowance".
---

# Akita Wallet SDK

High-level client for ARC-58 abstracted account wallets. Use this skill when writing TypeScript that creates wallets, installs plugins, spends through escrows, or estimates costs before sending.

## Package and imports

```typescript
// Core wallet surface
import {
  WalletSDK,
  WalletFactorySDK,
  newWallet,
  // Allowance type guards
  isFlatAllowance,
  isWindowAllowance,
  isDripAllowance,
} from 'akita-sdk/wallet'

// Plugin SDKs live in the same sub-path
import {
  PayPluginSDK,
  OptInPluginSDK,
  AsaMintPluginSDK,
  DAOPluginSDK,
  SocialPluginSDK,
  StakingPluginSDK,
  StakingPoolPluginSDK,
  MarketplacePluginSDK,
  AuctionPluginSDK,
  RafflePluginSDK,
  PollPluginSDK,
  RewardsPluginSDK,
  SubscriptionsPluginSDK,
  HyperSwapPluginSDK,
  GatePluginSDK,
  NFDPluginSDK,
  UpdateAkitaDAOPluginSDK,
} from 'akita-sdk/wallet'

// Every non-wallet SDK follows the same construction pattern; network helpers:
import { getNetworkAppIds, setCurrentNetwork } from 'akita-sdk'
```

## Create a wallet

Prefer `newWallet` — it creates the app and calls `register({ escrow: '' })` in one step.

```typescript
import { AlgorandClient } from '@algorandfoundation/algokit-utils'
import { newWallet } from 'akita-sdk/wallet'

const algorand = AlgorandClient.fromEnvironment()

const wallet = await newWallet({
  algorand,
  factoryParams: {
    appId: WALLET_FACTORY_APP_ID,   // from env or getNetworkAppIds()
    defaultSender: sender,
    defaultSigner: signer,
  },
  sender,
  signer,
  nickname: 'my_wallet',
  admin: customAdmin,                 // optional, defaults to sender
  referrer: referrerAddress,          // optional
})
```

Factory-then-register path (use when you need the raw result between create and register):

```typescript
const factory = new WalletFactorySDK({
  factoryParams: { appId: WALLET_FACTORY_APP_ID },
  algorand,
})
const wallet = await factory.new({ sender, signer, nickname: 'my_wallet' })
await wallet.register({ escrow: '' })
```

Load an existing wallet:

```typescript
const existing = await factory.get({ appId: walletAppId })
```

## Fund the wallet (MBR)

Every time you add a plugin / escrow / allowance, the wallet app needs more MBR. Compute it first, then fund.

```typescript
const mbr = await wallet.getMbr({
  escrow: '',
  methodCount: 0n,
  plugin: '',
  groups: 0n,
})

await wallet.client.appClient.fundAppAccount({
  amount: algokit.microAlgo(mbr.plugins),
})
```

## Install plugins

`addPlugin` is the single entry point for everything installable. Key option combinations:

```typescript
// Global: anyone can call through the wallet
await wallet.addPlugin({ client: payPlugin, global: true })

// Caller-specific, with a cooldown per method
await wallet.addPlugin({
  client: payPlugin,
  caller: userAddress,
  useRounds: true,
  methods: [{ name: payPlugin.pay(), cooldown: 100n }],
})

// Named plugin for easier reference
await wallet.addPlugin({
  client: stakingPlugin,
  global: true,
  name: 'staking',
})

// Remove
await wallet.removePlugin({
  plugin: payPlugin.appId,
  caller: ALGORAND_ZERO_ADDRESS_STRING,
  escrow: '',
})
```

Full option set:

| Option | Type | Meaning |
|---|---|---|
| `client` | SDK instance | Plugin SDK to install |
| `global` | `boolean` | If true, anyone can invoke |
| `caller` | `string` | Restrict to one address (ignored if `global`) |
| `name` | `string` | Named plugin alias |
| `methods` | `{ name, cooldown }[]` | Method-level cooldowns |
| `escrow` | `string` | Bind plugin to a named escrow |
| `allowances` | `Allowance[]` | Spending allowances (requires escrow) |
| `useRounds` | `boolean` | Cooldowns in rounds vs timestamps |
| `useExecutionKey` | `boolean` | Require pre-authorized execution keys |
| `coverFees` | `boolean` | Wallet pays txn fees |
| `cooldown` | `bigint` | Plugin-wide cooldown |
| `lastValid` | `bigint` | Expiration round |
| `admin` | `boolean` | Grant admin privileges |

## Invoke plugins

Every plugin call goes through `wallet.usePlugin({...})` with one or more `calls` composed from the plugin SDK.

```typescript
await wallet.usePlugin({
  global: true,
  consolidateFees: true,
  calls: [
    payPlugin.pay({
      payments: [{ receiver, asset: 0n, amount: 1_000_000n }],
    }),
  ],
})
```

Batching multiple calls in one group:

```typescript
await wallet.usePlugin({
  global: true,
  consolidateFees: true,
  calls: [
    optInPlugin.optIn({ assets: [assetId] }),
    payPlugin.pay({
      payments: [{ receiver, asset: assetId, amount: 100_000_000n }],
    }),
  ],
})
```

## Escrows

Escrows are named spending pools inside the wallet. Most useful with allowances.

```typescript
// Create by adding a plugin with an escrow name
await wallet.addPlugin({ client: payPlugin, global: true, escrow: 'budget' })

// Or create explicitly
await wallet.newEscrow({ name: 'savings' })

// Query
const all = await wallet.getEscrows()
const one = await wallet.getEscrow('budget')

// Lock/unlock, opt-in, reclaim
await wallet.toggleEscrowLock({ name: 'savings' })
await wallet.optinEscrow({ name: 'savings', assets: [assetId] })
await wallet.reclaimFunds({
  name: 'savings',
  funds: [[0n, 1_000_000n, false]],  // [asset, amount, closeOut]
})
```

## Allowances (flat, window, drip)

Three allowance shapes live under a `{ escrow, asset }` key. The type discriminator is `type: 'flat' | 'window' | 'drip'`.

```typescript
// Flat: one-time budget
await wallet.addPlugin({
  client: payPlugin,
  global: true,
  escrow: 'budget',
  allowances: [{ type: 'flat', asset: 0n, amount: 10_000_000n }],
})

// Window: amount per interval
await wallet.addPlugin({
  client: payPlugin,
  global: true,
  escrow: 'monthly',
  allowances: [{
    type: 'window',
    asset: 0n,
    amount: 10_000_000n,
    interval: 100n,
    useRounds: true,
  }],
})

// Drip: rate-based accrual capped at max
await wallet.addPlugin({
  client: payPlugin,
  global: true,
  escrow: 'salary',
  allowances: [{
    type: 'drip',
    asset: 0n,
    rate: 1_000_000n,
    interval: 10n,
    max: 10_000_000n,
    useRounds: true,
  }],
})
```

Spend against an allowance with `fundsRequest`:

```typescript
await wallet.usePlugin({
  escrow: 'budget',
  global: true,
  calls: [payPlugin.pay({ payments: [{ receiver, amount: 5_000_000n, asset: 0n }] })],
  fundsRequest: [{ amount: 5_000_000n, asset: 0n }],
})
```

Inspect an on-chain allowance and narrow by type:

```typescript
const info = await wallet.getAllowance(key)

if (isFlatAllowance(info)) {
  // info.amount, info.spent
} else if (isWindowAllowance(info)) {
  // info.amount, info.spent, info.interval
} else if (isDripAllowance(info)) {
  // info.rate, info.max, info.lastLeftover, info.interval
}
```

## Cost estimation with `prepare.usePlugin`

Preview the expected cost before sending. The returned object has a `send()` you can still call.

```typescript
const prepared = await wallet.prepare.usePlugin({
  global: true,
  calls: [payPlugin.pay({ payments: [{ receiver, amount: 1_000_000n, asset: 0n }] })],
})

console.log(prepared.expectedCost.networkFees)
console.log(prepared.expectedCost.totalAlgo)
console.log(prepared.expectedCost.subtotals)

const result = await prepared.send()
```

`expectedCost` shape:

```typescript
type ExpectedCost = {
  payments: AssetPayment[]                        // per-asset breakdown
  networkFees: bigint                             // sum of txn fees
  subtotals: { asset: bigint; amount: bigint }[]
  totalAlgo: bigint                               // total ALGO cost
  accountDeltas?: AccountDelta[]
}

type AssetPayment = {
  asset: bigint
  amount: bigint   // actual payment
  mbr: bigint      // MBR locked
  fee: bigint      // non-refundable fee
  total: bigint    // amount + mbr + fee
}
```

## Deferred execution via execution keys

Use `build.usePlugin` to compile a group of transactions, stash an execution key, and let a third party submit later.

```typescript
const execution = await wallet.build.usePlugin({
  sender: executorAddress,           // who will submit
  signer: executorSigner,
  lease: 'my_lease',
  windowSize: 2000n,                 // validity window (rounds)
  global: true,
  calls: [payPlugin.pay({ payments: [{ receiver, amount: 1_000_000n, asset: 0n }] })],
})

// Wallet owner registers the pre-authorized execution key
await wallet.addExecutionKey({
  lease: execution.lease,
  groups: execution.ids,
  firstValid: execution.firstValid,
  lastValid: execution.lastValid,
})

// Later, a third party submits one of the groups
await execution.atcs[0].submit(algorand.client.algod)
```

This pattern is exactly what `scripts/script-base.ts` uses for DAO-approved contract upgrades (`runUpdate`).

## Profile, admin, balance

```typescript
await wallet.setNickname({ nickname: 'alice' })
await wallet.setAvatar({ avatar: avatarAssetId })
await wallet.setBanner({ banner: bannerAssetId })
await wallet.setBio({ bio: 'Hello world' })

await wallet.changeAdmin({ newAdmin: newAdminAddress })
await wallet.verifyAuthAddress()

const balances = await wallet.balance([0n, usdcAssetId])
```

## SDK construction pattern (shared by every Akita SDK)

Every SDK in `akita-sdk/*` follows the same shape. The `appId` resolves in the order: explicit param > env var > network config.

```typescript
const sdk = new SomeSDK({
  algorand,
  factoryParams: {
    appId: APP_ID,
    defaultSender: sender,
    defaultSigner: signer,
  },
  readerAccount: readOnlyAddress,   // optional; for read-only queries
})
```

Network resolution:

```typescript
import { setCurrentNetwork, getCurrentNetwork, getNetworkAppIds } from 'akita-sdk'

setCurrentNetwork('mainnet')
const appIds = getNetworkAppIds()  // { dao, wallet, walletFactory, ... }
```

Environment variables recognized by the SDK:

```
ALGORAND_NETWORK=mainnet
WALLET_FACTORY_APP_ID=...
PAY_PLUGIN_APP_ID=...
DAO_APP_ID=...
# ...one per contract
```

## End-to-end example

Create a wallet, install Pay, estimate cost, send, verify the spend.

```typescript
import { AlgorandClient, microAlgo } from '@algorandfoundation/algokit-utils'
import { newWallet, PayPluginSDK } from 'akita-sdk/wallet'

const algorand = AlgorandClient.fromEnvironment()

const wallet = await newWallet({
  algorand,
  factoryParams: { appId: WALLET_FACTORY_APP_ID, defaultSender: sender, defaultSigner: signer },
  sender,
  signer,
  nickname: 'main',
})

const payPlugin = new PayPluginSDK({
  algorand,
  factoryParams: { appId: PAY_PLUGIN_APP_ID },
})

// Fund and install
const mbr = await wallet.getMbr({ escrow: '', methodCount: 0n, plugin: '', groups: 0n })
await wallet.client.appClient.fundAppAccount({ amount: microAlgo(mbr.plugins) })
await wallet.addPlugin({ client: payPlugin, global: true })

// Preview
const prepared = await wallet.prepare.usePlugin({
  global: true,
  calls: [payPlugin.pay({ payments: [{ receiver, asset: 0n, amount: 1_000_000n }] })],
})
console.log('Expected ALGO cost:', prepared.expectedCost.totalAlgo)

// Send
await prepared.send()
```

## Common mistakes to avoid

- **Forgetting to fund MBR** before `addPlugin` / `newEscrow` — the call will fail with a min-balance error.
- **Mixing `global: true` with `caller`** — `caller` is ignored when `global` is true; pick one model per plugin install.
- **Spending through an allowance without `fundsRequest`** — allowances only gate `fundsRequest` amounts; plain plugin calls bypass them and will hit the global spend check instead.
- **Reusing the same `lease` for `build.usePlugin`** — every execution key is keyed by `lease`; collisions overwrite. Append a timestamp (`Date.now() % 1_000_000`) as the pattern in `script-base.ts` does.
- **Assuming `getNetworkAppIds()` resolves without `setCurrentNetwork` or `ALGORAND_NETWORK`** — it returns localnet by default.
- **Passing plugin instances instead of `client.appId` to `removePlugin`** — `removePlugin` takes a raw app ID, not an SDK instance.

## Reference

- `WalletSDK` methods: `register`, `verifyAuthAddress`, `changeAdmin`, `getMbr`, `getGlobalState`, `getAdmin`, `balance`
- Plugins: `addPlugin`, `usePlugin`, `removePlugin`, `getPlugins`, `getPluginByKey`, `getNamedPlugins`, `getPluginByName`, `canCall`
- Escrows: `newEscrow`, `getEscrows`, `getEscrow`, `toggleEscrowLock`, `optinEscrow`, `reclaimFunds`
- Allowances: `addAllowances`, `removeAllowances`, `getAllowances`, `getAllowance`
- Profile: `setNickname`, `setAvatar`, `setBanner`, `setBio`
- Execution keys: `addExecutionKey`, `removeExecutionKey`, `getExecutions`, `getExecution`
- Advanced: `prepare.usePlugin`, `build.usePlugin`, `group()`
