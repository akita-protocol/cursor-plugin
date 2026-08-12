---
name: akita-social-stack
description: Use the Akita social contracts — `AkitaSocial`, `AkitaSocialGraph`, `AkitaSocialImpact`, `AkitaSocialModeration` — and the `SocialPluginSDK` for on-chain posts, votes, reactions, follows, impact scoring, and moderation. Use when building a social frontend, invoking social actions through a wallet, reading social state, writing a social gate, or extending the moderator flow. Strong triggers include "post a thought", "follow user", "react to post", "SocialSDK", "SocialPluginSDK", "PostRef", "getUserImpact", "isFollowing", "flagPost", "ban user", "initMeta", "socialActivityGate", "socialImpactGate", "socialModeratorGate".
---

# Akita Social Stack

The Akita social layer is four cooperating contracts plus a wallet plugin. They share state via AVM reads, not opaque APIs, so the SDK and gates can consume each other's data directly.

## The four contracts

| Contract | Source | Network key | Responsibility |
|---|---|---|---|
| **AkitaSocial** | `smart_contracts/social/contract.algo.ts` | `social` | Posts, replies, votes, reactions, meta, paywalls |
| **AkitaSocialGraph** | `smart_contracts/social/graph.algo.ts` | `socialGraph` | Follows, blocks, follower indexing |
| **AkitaSocialImpact** | `smart_contracts/social/impact.algo.ts` | `socialImpact` | Engagement / reputation score |
| **AkitaSocialModeration** | `smart_contracts/social/moderation.algo.ts` | `socialModeration` | Moderators, bans, post flags |

App IDs live in `akita-sdk/src/networks.ts` under `social`, `socialGraph`, `socialImpact`, `socialModeration`. Env var keys: `SOCIAL_APP_ID`, `SOCIAL_GRAPH_APP_ID`, `SOCIAL_IMPACT_APP_ID`, `SOCIAL_MODERATION_APP_ID`.

## SDK construction

The `SocialSDK` from `akita-sdk/social` aggregates all four contracts behind a single client.

```typescript
import { AlgorandClient } from '@algorandfoundation/algokit-utils'
import { SocialSDK } from 'akita-sdk/social'
import { getNetworkAppIds } from 'akita-sdk'

const algorand = AlgorandClient.fromEnvironment()
const ids = getNetworkAppIds()

const social = new SocialSDK({
  algorand,
  daoAppId: ids.dao,
  socialFactoryParams:     { appId: ids.social,           defaultSender: sender, defaultSigner: signer },
  graphFactoryParams:      { appId: ids.socialGraph,      defaultSender: sender, defaultSigner: signer },
  impactFactoryParams:     { appId: ids.socialImpact,     defaultSender: sender, defaultSigner: signer },
  moderationFactoryParams: { appId: ids.socialModeration, defaultSender: sender, defaultSigner: signer },
  readerAccount: myAddress,  // optional
  ipfsUrl: 'https://ipfs.example.com',  // optional; required for post content resolution
})
```

## PostRef — the canonical post identity

Every post is addressed by a 32-byte reference derived from the creator, a unix timestamp, and a nonce.

```typescript
// akita-sdk/src/social/types.ts
export type PostRef = Uint8Array  // 32-byte sha256 hash
```

Helpers on `SocialSDK`:

```typescript
SocialSDK.generatePostNonce(): Uint8Array                        // 24 random bytes
social.computePostKey(creator, timestamp, nonce): Uint8Array     // sha256(creator || ts || nonce)
social.computeEditKey(creator, originalKey, newCid): Uint8Array
```

`refType` is an enum passed alongside `ref` to disambiguate posts vs replies vs edits:

```typescript
enum RefType { Post, Reply, Edit, /* ... */ }
```

## Read methods (SocialSDK)

Direct on-chain reads — no signer required:

```typescript
// Social state
await social.getSocialFees()
await social.getMeta(userAddress)            // { lastActive, ... }
await social.getPost(ref)                    // decoded post value
await social.getVote(account, ref)
await social.getVotes([{ account, ref }, ...])
await social.isBanned(account)               // from moderation

// Graph
await social.isFollowing(follower, user)
await social.getFollowIndex(follower, user)

// Impact
await social.getUserImpact(address)
await social.getImpactMeta(user)             // { lastActive, score metadata }
```

## Write methods (SocialSDK)

These are admin / DAO-only in most cases. **Regular user actions should go through the wallet plugin** (see below), not direct SDK writes.

```typescript
// Moderation (moderator / DAO only)
await social.ban({ address, expiration })
await social.unban({ address })
await social.flagPost({ ref })

// Direct post / vote / follow are available on the SDK but require fee coverage
// and are typically called through the plugin. Use only for admin scripts.
```

## SocialPluginSDK — user actions via the wallet

All user-initiated social actions should flow through `SocialPluginSDK`, which the ARC-58 wallet invokes. This keeps the wallet as the on-chain spender and lets gates apply.

```typescript
import { SocialPluginSDK } from 'akita-sdk/wallet'

const socialPlugin = new SocialPluginSDK({
  algorand,
  factoryParams: { appId: ids.socialPlugin },
})

await wallet.addPlugin({ client: socialPlugin, global: true })

// Post
await wallet.usePlugin({
  global: true,
  calls: [socialPlugin.post({ cid: cidBytes /* 36 bytes */ })],
})

// Reply
await wallet.usePlugin({
  global: true,
  calls: [socialPlugin.reply({ cid, ref: parentRef, refType: RefType.Post })],
})

// Vote (up/down)
await wallet.usePlugin({
  global: true,
  calls: [socialPlugin.vote({ ref: postRef, refType: RefType.Post, isUp: true })],
})

// React (with an NFT)
await wallet.usePlugin({
  global: true,
  calls: [socialPlugin.react({ ref: postRef, refType: RefType.Post, nft: nftAssetId })],
})

// Follow
await wallet.usePlugin({
  global: true,
  calls: [socialPlugin.follow({ address: targetUserAddress })],
})
```

### Full plugin surface

```
post / editPost / reply / gatedReply
vote / editVote
react / gatedReact / deleteReaction
follow / gatedFollow / unfollow
block / unblock
addModerator / removeModerator
ban / unban
flagPost / unflagPost
addAction / removeAction
initMeta / updateMeta
```

The `gated*` variants accept a `gateTxn` argument — a pre-built gate check transaction. Use them when the corresponding action is behind a gate (see `akita-gate-composition`).

## `initMeta` is required before most actions

Before a user can post, follow, or react, their social meta must exist. Call `initMeta` once per user. Most frontends do this lazily on first write.

```typescript
// Via SDK directly (admin / setup)
await social.initMeta({ sender: user.addr, signer })

// Or via the plugin on a newly-created wallet
await wallet.usePlugin({
  global: true,
  calls: [socialPlugin.initMeta({})],
})
```

## AkitaSocialGraph — follow graph

Follows are stored as a box map `{user, follower} → followIndex`. The index lets gates check "is address X the n-th follower of Y".

```typescript
// graph.algo.ts:29
follows = BoxMap<FollowsKey, uint64>({ keyPrefix: AkitaSocialBoxPrefixFollows })

// SDK wrappers
await social.isFollowing(follower, user)       // boolean
await social.getFollowIndex(follower, user)    // bigint index (or throws if not following)
```

## AkitaSocialImpact — reputation

Impact is an aggregate score per user. The contract reads votes received, engagement, subscription state, and NFD / NFT holdings, then writes the score to a box.

```typescript
await social.getUserImpact(address)     // bigint score
await social.getImpactMeta(user)        // { lastActive, score metadata }
```

Impact is consumed by `socialImpactGate` and can be read by any other contract via inner app call.

## AkitaSocialModeration — bans, flags, moderators

Moderation state lives in three box maps:

```typescript
// moderation.algo.ts
moderators    = BoxMap<Account, uint64>()       // { lastActive } per moderator
banned        = BoxMap<Account, uint64>()       // ban expiration timestamp
flaggedPosts  = BoxMap<bytes<32>, uint64>()     // flag timestamp per post ref
```

- `addModerator(address)` — DAO-only; grants moderator role.
- `ban(address, expiration)` — moderator action; blocks user until a timestamp.
- `isBanned(account)` — anyone can read.
- `flagPost(ref)` — marks a post for review.

## End-to-end flow (from tests)

```typescript
import { makeBasicAccountTransactionSigner } from 'algosdk'
import { SocialSDK } from 'akita-sdk/social'
import { RefType } from 'akita-sdk/social'

// 1. Deploy contracts (or use an existing universe)
//    socialFactory / graphFactory / impactFactory / moderationFactory .send.create.create({...})

// 2. Construct SDK
const sdk = new SocialSDK({
  algorand,
  socialFactoryParams:     { appId: socialAppId,     defaultSender, defaultSigner },
  graphFactoryParams:      { appId: graphAppId,      defaultSender, defaultSigner },
  impactFactoryParams:     { appId: impactAppId,     defaultSender, defaultSigner },
  moderationFactoryParams: { appId: moderationAppId, defaultSender, defaultSigner },
})

// 3. Bootstrap meta for both users
await sdk.initMeta({ sender: user1.addr, signer: makeBasicAccountTransactionSigner(user1) })
await sdk.initMeta({ sender: user2.addr, signer: makeBasicAccountTransactionSigner(user2) })

// 4. Follow
await sdk.follow({
  sender: user1.addr,
  signer: makeBasicAccountTransactionSigner(user1),
  address: user2.addr.toString(),
})
const isFollowing = await sdk.isFollowing({
  follower: user1.addr.toString(),
  user: user2.addr.toString(),
})

// 5. Post from user1
const { postKey } = await sdk.post({
  sender: user1.addr,
  signer: makeBasicAccountTransactionSigner(user1),
  cid: new Uint8Array(36).fill(1),   // IPFS CID bytes
})

// 6. React from user2
await sdk.react({
  sender: user2.addr,
  signer: makeBasicAccountTransactionSigner(user2),
  ref: postKey,
  refType: RefType.Post,
  nft: reactionNftId,
})

// 7. Check impact
const impact = await sdk.getUserImpact({ address: user2.addr.toString() })
```

## Social gates

Five sub-gates consume social state. They all use the standard operator enum: `Equal | GreaterThan | GreaterThanOrEqualTo | LessThan | LessThanOrEqualTo | NotEqual`.

| Gate | Reads | Gates on |
|---|---|---|
| `socialActivityGate` | `AkitaSocial.getMeta(user).lastActive` | "User posted within N rounds" |
| `socialFollowerCountGate` | Social meta follower count | Minimum / maximum followers |
| `socialFollowerIndexGate` | `AkitaSocialGraph.getFollowIndex(follower, user)` | "Address is the n-th follower" |
| `socialImpactGate` | `AkitaSocialImpact.getUserImpact(address)` | Impact threshold |
| `socialModeratorGate` | `AkitaSocialModeration.moderators(addr).exists` | User is a moderator |

Gate contracts live under `smart_contracts/gates/sub-gates/social-*`. They're installed once per sub-gate type and referenced by gate ID in any contract that checks access.

## IPFS / content resolution

Posts store 36-byte CIDs on-chain, not bodies. `SocialSDK` accepts an optional `ipfsUrl` to resolve bodies client-side. If you're building a UI:

1. Read the post: `await social.getPost(ref)` → `{ cid, ... }`.
2. Fetch `${ipfsUrl}/${cidString}` to get the JSON body.
3. Cache aggressively — on-chain posts are immutable; edits create a new `EditKey`.

## Common mistakes to avoid

- **Calling writes before `initMeta`** — most actions assert the user's meta exists. The error surfaces as an AVM opcode failure, not a clear "missing meta" message.
- **Calling `SocialSDK.post` directly from a user wallet** — it works but the user pays all fees and bypasses plugin gates. Use `SocialPluginSDK.post` through `wallet.usePlugin` instead.
- **Passing a string to `ref`** — `PostRef` is always `Uint8Array`. Convert from hex / base64 first.
- **Forgetting to pass `gateTxn` to `gatedFollow` / `gatedReact`** — gated variants require a gate-check transaction to be composed first. Without it the plugin call will fail the gate check.
- **Reading `getUserImpact` without the impact contract deployed** — the SDK won't error immediately; it will return 0. Verify `ids.socialImpact` is non-zero in the current network.
- **Assuming follow is reciprocal** — the graph stores one-way edges. Use `isFollowing(a, b)` and `isFollowing(b, a)` separately.
