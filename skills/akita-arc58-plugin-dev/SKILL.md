---
name: akita-arc58-plugin-dev
description: Build ARC-58 plugins for Akita abstracted account wallets using the `@akta/plugin` package (PuyaTs). Use when writing a new plugin contract, extending an existing plugin, implementing plugin methods that call inner transactions, adding rekey logic, or writing plugin E2E tests. Strong triggers include "write a plugin for the wallet", "new arc58 plugin", "plugin method signature", "getSpendingAccount", "rekeyBack", "rekeyAddress", "rekeyBackIfNecessary", "@akta/plugin", "plugin.algo.ts".
---

# Akita ARC-58 Plugin Development

Build plugins that extend Akita ARC-58 abstracted accounts. Plugins are PuyaTs smart contracts that the wallet invokes via inner transactions. The `@akta/plugin` package provides the `Plugin` base class and the helpers required to satisfy Akita's rekey contract.

## When to use

- You are writing a new plugin under `projects/akita-sc/smart_contracts/arc58/plugins/<plugin>/`.
- You are adding or modifying a method on an existing plugin.
- You are reviewing a plugin for correctness (method signature, rekey handling).
- You are writing plugin E2E tests that install the plugin on a wallet and call it.

## Mandatory plugin method contract

Every public plugin method **must** follow these rules. Violating any of them will either fail at runtime or permanently break the wallet's rekey state.

1. `wallet: Application` is the **first** parameter.
2. `rekeyBack: boolean` is the **second** parameter.
3. When `rekeyBack` is true, the **last** on-chain operation must rekey back to the wallet (`rekeyAddress(rekeyBack, wallet)` as `rekeyTo`).
4. Intermediate inner transactions in the same method must use `Global.zeroAddress` as `rekeyTo`, not the wallet address.
5. If the last step is not an inner transaction (e.g. a box write or state update), call `rekeyBackIfNecessary(rekeyBack, wallet)` at the end.

## `@akta/plugin` API (exact exports)

```typescript
// Base contract
export { Plugin } from './base.algo'

// Utility functions
export {
  getSpendingAccount,     // sender for itxns
  getOriginAccount,       // the real user who owns the wallet
  getReferrerAccount,     // referrer account, or zero address
  getAccounts,            // { walletAddress, origin, sender, referrer }
  rekeyAddress,           // use as rekeyTo on the LAST itxn
  rekeyBackIfNecessary,   // emit a no-op rekey when last step isn't an itxn
} from './functions'

// Types
export type { Arc58Accounts, Arc58PluginCallContext } from './types'

// Constants (wallet global-state keys, rarely needed directly)
export {
  ARC58_CONTROLLED_ADDRESS_KEY,  // 'controlled_address'
  ARC58_SPENDING_ADDRESS_KEY,    // 'spending_address'
  ARC58_REFERRER_KEY,            // 'referrer'
  ARC58_CURRENT_PLUGIN_KEY,      // 'current_plugin'
  ARC58_REKEY_INDEX_KEY,         // 'rekey_index'
} from './constants'
```

### Helper semantics

| Helper | Returns | When to use |
|---|---|---|
| `getSpendingAccount(wallet)` | `Account` rekeyed to the plugin | `sender` on every itxn |
| `getOriginAccount(wallet)` | The real owner `Account` | When the method needs the user's identity (e.g. social posts) |
| `getReferrerAccount(wallet)` | Referrer `Account` or zero address | Revenue-share plugins |
| `rekeyAddress(rekeyBack, wallet)` | `wallet.address` if `rekeyBack`, else `Global.zeroAddress` | `rekeyTo` on the **last** itxn only |
| `rekeyBackIfNecessary(rekeyBack, wallet)` | `void` | Call at the end of a method whose last op is **not** an itxn |

## Plugin skeleton

```typescript
import { Application, Contract, itxn, uint64, Global } from '@algorandfoundation/algorand-typescript'
import { Plugin, getSpendingAccount, rekeyAddress } from '@akta/plugin'

export class MyPlugin extends Plugin {
  doSomething(wallet: Application, rekeyBack: boolean, amount: uint64): void {
    const sender = getSpendingAccount(wallet)

    itxn
      .payment({
        sender,
        receiver: someReceiver,
        amount,
        rekeyTo: rekeyAddress(rekeyBack, wallet),
      })
      .submit()
  }
}
```

## Single-loop pattern with conditional rekey on the last iteration

Use this shape whenever a method loops over a list of operations. The last iteration gets `rekeyAddress(rekeyBack, wallet)`; every earlier iteration gets `Global.zeroAddress`.

```typescript
// pay/contract.algo.ts:7-34 (reference)
pay(wallet: Application, rekeyBack: boolean, payments: PayParams[]): void {
  const sender = getSpendingAccount(wallet)

  for (let i: uint64 = 0; i < payments.length; i++) {
    const { receiver, asset, amount } = payments[i]
    const rekeyTo = i < (payments.length - 1)
      ? Global.zeroAddress
      : rekeyAddress(rekeyBack, wallet)

    if (asset === 0) {
      itxn.payment({ sender, receiver, amount, rekeyTo }).submit()
    } else {
      itxn.assetTransfer({
        sender,
        assetReceiver: receiver,
        assetAmount: amount,
        xferAsset: asset,
        rekeyTo,
      }).submit()
    }
  }
}
```

## Multi-itxn composed groups

Use `itxnCompose` when multiple inner transactions must be submitted as one group (e.g. fund + abi call + asset transfer). Only the final `next(...)` before `submit()` carries the rekey.

```typescript
// dual-stake/contract.algo.ts:26-83 (reference)
mint(wallet: Application, rekeyBack: boolean, appId: Application, amount: uint64): void {
  const sender = getSpendingAccount(wallet)

  itxnCompose.begin(
    itxn.payment({ sender, receiver: appId.address, amount }),
  )

  const rate = abiCall<typeof DualStake.prototype.get_rate>({ sender, appId }).returnValue

  if (rate > 0) {
    itxnCompose.next<typeof DualStake.prototype.mint>({ sender, appId })

    const asaAmount = /* ... compute asa amount ... */
    itxnCompose.next(
      itxn.assetTransfer({
        sender,
        assetReceiver: appId.address,
        assetAmount: asaAmount,
        xferAsset: asaID,
        rekeyTo: rekeyAddress(rekeyBack, wallet),  // final itxn carries rekey
      }),
    )
    itxnCompose.submit()
    return
  }

  // Fallback branch: fewer itxns, rekey on the last one
  itxnCompose.next<typeof DualStake.prototype.mint>({
    sender,
    appId,
    rekeyTo: rekeyAddress(rekeyBack, wallet),
  })
  itxnCompose.submit()
}
```

## When the last operation is not an itxn

If the method's final action is a box write, global-state update, or any non-itxn op, call `rekeyBackIfNecessary(rekeyBack, wallet)` at the end. It emits a zero-value payment from the spending account with the correct `rekeyTo`.

```typescript
updateBox(wallet: Application, rekeyBack: boolean, value: bytes): void {
  const sender = getSpendingAccount(wallet)
  // Inner work that doesn't end in an itxn...
  this.myBox(sender).value = value
  rekeyBackIfNecessary(rekeyBack, wallet)
}
```

## Read-only methods

Read-only methods do not rekey. They still accept `wallet` and `rekeyBack` to keep the ABI uniform, but they simply do not emit any itxn. Do not call `rekeyBackIfNecessary` from a read-only method — it would make the call non-read-only.

## Expected directory layout

```
projects/akita-sc/smart_contracts/arc58/plugins/<plugin>/
├── contract.algo.ts         # Plugin contract
├── types.ts                 # ABI parameter structs
├── constants.ts             # Optional: plugin-specific constants
├── errors.ts                # Optional: error codes
└── <plugin>.e2e.spec.ts     # E2E test
```

`types.ts` example:

```typescript
// pay/types.ts
import { Account, uint64 } from '@algorandfoundation/algorand-typescript'

export type PayParams = {
  receiver: Account
  asset: uint64
  amount: uint64
}
```

## Building and generating clients

Each plugin has a dedicated build script in `projects/akita-sc/package.json`. Pattern:

```bash
# Compile + generate TS client for one plugin
pnpm --filter smart_contracts build:<pluginname>plugin

# Examples:
pnpm --filter smart_contracts build:payplugin
pnpm --filter smart_contracts build:asamintplugin
pnpm --filter smart_contracts build:socialplugin
```

The output lands in `projects/akita-sc/smart_contracts/artifacts/arc58/plugins/<plugin>/` with a `<PluginName>Client.ts` next to the ARC-56 spec.

## Testing plugins

Plugin E2E tests use `algorandFixture()` plus `buildAkitaUniverse()` to stand up the full DAO + wallet factory + plugin registry before running assertions.

### Fixture shape

```typescript
import { algorandFixture } from '@algorandfoundation/algokit-utils/testing'
import * as algokit from '@algorandfoundation/algokit-utils'
import { makeBasicAccountTransactionSigner } from 'algosdk'
import { newWallet } from 'akita-sdk/wallet'
import { buildAkitaUniverse } from '<tests utils>'  // tests/fixtures or tests/utils

const fixture = algorandFixture()

beforeAll(async () => {
  await fixture.beforeEach()
  const algorand = fixture.context.algorand
  const deployer = await fixture.context.generateAccount({ initialFunds: algokit.microAlgos(2_000_000_000) })
  const user = await fixture.context.generateAccount({ initialFunds: algokit.microAlgos(500_000_000) })

  const akitaUniverse = await buildAkitaUniverse({
    fixture,
    sender: deployer.addr,
    signer: makeBasicAccountTransactionSigner(deployer),
    apps: {},
  })

  const wallet = await newWallet({
    algorand,
    factoryParams: {
      appId: akitaUniverse.walletFactory.appId,
      defaultSender: user.addr,
      defaultSigner: makeBasicAccountTransactionSigner(user),
    },
    sender: user.addr,
    signer: makeBasicAccountTransactionSigner(user),
    nickname: 'Test Wallet',
  })

  // Fund the wallet and install the plugin under test
  const mbr = await wallet.getMbr({ escrow: '', methodCount: 0n, plugin: '', groups: 0n })
  await wallet.client.appClient.fundAppAccount({
    amount: algokit.microAlgo(mbr.plugins * 2n + 10_000_000n),
  })
  await wallet.addPlugin({ client: myPluginSdk, global: true })
})

beforeEach(fixture.newScope)
```

### Calling a plugin method

```typescript
const result = await wallet.usePlugin({
  global: true,
  calls: [
    payPluginSdk.pay({
      payments: [{ receiver: receiver.toString(), asset: 0n, amount: 1_000_000n }],
    }),
  ],
})
```

Run a single plugin test file:

```bash
pnpm --filter smart_contracts test:payplugin
# Or an arbitrary file:
pnpm --filter smart_contracts exec vitest run smart_contracts/arc58/plugins/<plugin>/contract.e2e.spec.ts
```

## Deployment vs update

Use the deploy scripts **only for brand-new plugin deployments**. Updates to existing plugins always go through the DAO via `runUpdate` in `script-base.ts` (see the `akita-deploy-scripts` skill for details).

A typical `deploy-<plugin>-plugin.ts` does this:

```typescript
import { parseBaseArgs, setupContext, runScript } from './script-base'
import { AuctionPluginFactory } from '../smart_contracts/artifacts/arc58/plugins/auction/AuctionPluginClient'
import { AuctionPluginSDK } from 'akita-sdk/wallet'

runScript(async () => {
  const options = parseBaseArgs('deploy-auction-plugin.ts')
  const ctx = await setupContext(options, { minBalance: 10_000_000n })

  const factory = ctx.algorand.client.getTypedAppFactory(AuctionPluginFactory, {
    defaultSender: ctx.sender,
    defaultSigner: ctx.signer,
  })

  const { appClient: client } = await factory.send.create.create({
    args: {
      version: options.version,
      factory: ctx.appIds.auctionFactory,
      akitaDao: ctx.appIds.dao,
    },
  })
  console.log(`IMPORTANT: update akita-sdk/src/networks.ts with new app ID: ${client.appId}n`)
})
```

## Common mistakes to avoid

- **Forgetting `getSpendingAccount(wallet)`** — sending itxns from `wallet.address` instead of the rekeyed spending account will fail.
- **Using `wallet.address` as `rekeyTo`** — use `rekeyAddress(rekeyBack, wallet)` so plugin chaining still works when `rekeyBack` is false.
- **Rekeying on an intermediate itxn** — only the last itxn in a method should rekey back.
- **Forgetting `rekeyBackIfNecessary`** — if the method ends with a box/state write and `rekeyBack` is true, the wallet will be left rekeyed to the plugin.
- **Mutating `wallet` as if it were the sender** — plugins never have signing authority over the wallet itself, only over the spending account.

## Reference: existing plugins by archetype

| Archetype | Example to copy from | Notes |
|---|---|---|
| Simple payment/transfer | `plugins/pay/contract.algo.ts` | Loop with last-iteration rekey |
| Single abi call | `plugins/staking/contract.algo.ts` | Use `rekeyAddress` on the itxn |
| Multi-txn composed group | `plugins/dual-stake/contract.algo.ts` | `itxnCompose.begin/next/submit` |
| Box write only | `plugins/social/contract.algo.ts` | Ends with `rekeyBackIfNecessary` |
| Asset mint/config | `plugins/asa-mint/contract.algo.ts` | `itxn.assetConfig` with `rekeyTo` |
