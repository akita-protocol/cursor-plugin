# Akita Accounts — Cursor Plugin

Connect [Cursor](https://cursor.com) to **Akita Accounts** (ARC-58 agent wallets). This plugin ships an MCP server for live wallet operations—payments, staking, DAO, social, and marketplace actions—plus curated skills for wallet and plugin development.

## Install

Install from **Cursor Plugins** / your team marketplace using this repository:

`https://github.com/akita-protocol/cursor-plugin`

See the [Cursor plugins reference](https://cursor.com/docs/reference/plugins) for marketplace and local install options.

## Setup

1. **Create an agent wallet + `.env`**  
   Run the Akita agent-wallet setup wizard to deploy an escrow and generate credentials:

   ```bash
   npx @akta/agent-wallet-cli
   # or, from the agent-wallet repo: pnpm setup
   ```

   This writes a `.env` with Intermezzo/Vault credentials, `WALLET_APP_ID`, and `ESCROW_NAME`.

2. **Install this plugin** in Cursor (Plugins → install from this repo / marketplace).

3. **Set `ENV_FILE`**  
   Open **Plugins → Configure** for *Akita Accounts* and set `ENV_FILE` to the **absolute path** of the `.env` from step 1.

4. **Run Intermezzo**  
   Ensure the Intermezzo Docker stack used by the agent wallet is running before invoking MCP tools.

## What ships

### MCP server

| Server | Package / entry | Role |
|--------|-----------------|------|
| `akita-agent-wallet` | `@akta/agent-wallet-mcp` | Live ARC-58 wallet ops via MCP |

### Skills

| Skill | Use when |
|-------|----------|
| `akita-agent-wallet` | Driving live wallet actions through the MCP server |
| `akita-wallet-sdk` | Writing TypeScript against `akita-sdk/wallet` |
| `akita-arc58-plugin-dev` | Building or extending ARC-58 plugins |
| `akita-social-stack` | Social / governance plugin flows |
| `akita-connect` | Connect / auth integration patterns |

## Local MCP override

If the published npm package is not available yet, point the MCP server at a local build instead of `npx @akta/agent-wallet-mcp`. For example, in `mcp.json` (or your Cursor MCP override):

```json
{
  "mcpServers": {
    "akita-agent-wallet": {
      "command": "node",
      "args": ["/absolute/path/to/agent-wallet/packages/mcp/dist/index.js"],
      "env": {
        "ENV_FILE": "${ENV_FILE}"
      }
    }
  }
}
```

Replace the path with your checkout of [`akita-protocol/agent-wallet`](https://github.com/akita-protocol/agent-wallet) after building `packages/mcp`.

## Links

- Docs: [https://docs.akita.community](https://docs.akita.community)
- Skills: [https://github.com/akita-protocol/skills](https://github.com/akita-protocol/skills)
- Agent wallet: [https://github.com/akita-protocol/agent-wallet](https://github.com/akita-protocol/agent-wallet)
- Cursor plugins: [https://cursor.com/docs/reference/plugins](https://cursor.com/docs/reference/plugins)

## License

MIT — see [LICENSE](./LICENSE).
