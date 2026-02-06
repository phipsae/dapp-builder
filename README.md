# dApp Builder

Autonomous Ethereum dApp builder skill for AI coding agents. Writes Solidity contracts, Foundry tests, and Scaffold-ETH 2 frontend code — then compiles, tests, and deploys using MCP tools. No manual CLI commands needed.

## Install

**Step 1: Install the skill**

```bash
npx skills add <your-username>/dapp-builder
```

**Step 2: Connect the MCP server** (provides compile, test, deploy tools)

```bash
claude mcp add --transport http mpc-se2 https://focused-victory-production-311c.up.railway.app/mcp
```

## Usage

Open Claude Code in any project and either:

- **Slash command:** `/dapp-builder build me a no-loss lottery`
- **Just ask:** "Build me an ERC20 token" — the skill triggers automatically

Claude will autonomously loop through: write contract → compile → security check → write tests → run tests → fix issues → build frontend → assemble project → deploy.

## What's included

**8 MCP tools** (hosted, no local setup needed):

| Tool | What it does |
|------|-------------|
| `compile_contracts` | Compile Solidity with solc. Auto-resolves OpenZeppelin v5 imports. |
| `check_security` | Security analysis + gas estimation. |
| `run_tests` | Run Foundry tests in a temp project. |
| `start_anvil` | Start local Anvil testnet with funded accounts. |
| `stop_anvil` | Stop Anvil instance. |
| `deploy_local` | Deploy to any EVM network (local or remote). |
| `assemble_project` | Scaffold-ETH 2 project assembly. |
| `push_github` | Create GitHub repo and push project. |

## Verify

```bash
# Check skill is installed
# Type "/" in Claude Code — you should see "dapp-builder"

# Check MCP tools are connected
claude mcp list
# Should show "mpc-se2" with 8 tools
```

## License

MIT
