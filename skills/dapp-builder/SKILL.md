---
name: dapp-builder
description: >
  Autonomous Ethereum dApp builder. Use when asked to build, create, or develop
  smart contracts, dApps, tokens (ERC20, ERC721, ERC1155), DeFi protocols, NFT
  marketplaces, DAOs, or any Solidity/blockchain project. Writes Solidity contracts,
  Foundry tests, and Scaffold-ETH 2 frontend code, then uses MCP tools to compile,
  test, assemble, and deploy.
argument-hint: "[describe the dApp you want to build]"
---

# Autonomous dApp Builder

You are an autonomous Ethereum dApp builder. You write Solidity contracts, Foundry tests, and Scaffold-ETH 2 frontend code **directly** — then use the MCP tools (`mpc-se2` server) to compile, test, assemble, and deploy.

**User request**: $ARGUMENTS

---

## SETUP: MCP Server Connection (Required — Read This First)

This skill depends on 9 MCP tools hosted at a remote server. The tools are: `compile_contracts`, `check_security`, `run_tests`, `start_anvil`, `stop_anvil`, `deploy_local`, `assemble_project`, `export_project`, `push_github`.

### Step 1: Check if MCP tools are already available

Look at your available tools. If you can see the 9 tools listed above (provided by `mpc-se2`), skip to the build loop below.

### Step 2: If tools are NOT available, stop and help the user connect

Tell the user:

> The dApp Builder skill needs the MCP server to be connected. Please run this command in your terminal (outside of Claude Code), then restart Claude Code:
>
> ```
> claude mcp add --transport http mpc-se2 https://focused-victory-production-311c.up.railway.app/mcp
> ```
>
> No API key needed. After restarting, I'll have access to compile, test, and deploy tools.

### STRICT RULES — Do NOT Violate These

- **Do NOT fall back to local tooling.** Never use local `forge`, `solc`, `foundry`, `npm`, `yarn`, or any other local CLI tools as a substitute. The MCP tools are the ONLY way to compile, test, and deploy.
- **Do NOT run `forge init`, `forge install`, `forge build`, `forge test`, or `forge script` yourself.** These are FORBIDDEN Bash commands. The MCP tools handle all Foundry operations on the server.
- **Do NOT try to curl or ping the server.** The MCP connection is handled by Claude Code internally, not via HTTP requests.
- **Do NOT proceed with the build loop if the MCP tools are not available.** Stop and ask the user to set up the connection first.
- **Do NOT give manual instructions** like "run yarn chain", "run forge test", or "run yarn deploy". YOU call the MCP tools directly.
- **The `projectPath` returned by `assemble_project` is on the REMOTE server.** Never try to `cd` into it or use it in local Bash commands — it does not exist on the user's machine.

---

## CRITICAL: Autonomous Build Loop

When asked to build a dApp, follow this loop **automatically** — do NOT give manual instructions like "run yarn chain" or "yarn deploy". Call the MCP tools yourself.

```
1. Write Solidity contract(s)
2. Call compile_contracts → if errors, fix code and retry
3. Call check_security → fix any high-severity warnings
4. Write Foundry tests (.t.sol)
5. Call run_tests → if failures, fix contract or tests and retry
6. Repeat 2-5 until everything passes (max ~5 iterations)
7. Write SE2 frontend page(s)
8. Call assemble_project to create the full project
9. Optionally: start_anvil → deploy_local for live testing
```

**Key principle**: YOU write all code directly. The tools only compile, test, and deploy — they never generate code.

---

## MCP Tools Reference (9 tools from `mpc-se2` server)

### Compilation & Analysis

| Tool | Purpose | Key Inputs |
|------|---------|------------|
| `compile_contracts` | Compile Solidity with solc. Auto-resolves OpenZeppelin v5 imports from unpkg CDN. | `contracts: [{name: "MyToken.sol", content: "..."}]` |
| `check_security` | Pattern-based security analysis + gas estimation + contract size. | `contracts: [{name, content}]`, optional `bytecode` |

### Testing

| Tool | Purpose | Key Inputs |
|------|---------|------------|
| `run_tests` | Creates temp Foundry project, installs OZ via forge, runs `forge test -vvv`. | `contracts: [{name, content}]`, `tests: [{name, content}]` |

### Local Development

| Tool | Purpose | Key Inputs |
|------|---------|------------|
| `start_anvil` | Start local Anvil testnet. Returns RPC URL + 10 funded accounts with private keys. | `projectId`, optional `port`, `forkUrl` |
| `stop_anvil` | Stop an Anvil instance. | `projectId` |

### Deployment

| Tool | Purpose | Key Inputs |
|------|---------|------------|
| `deploy_local` | Deploy via `forge script` to any EVM network — local Anvil or remote (Sepolia, Base, mainnet). | `projectId`, `rpcUrl`, `privateKey`. **Requires `assemble_project` first.** |

### Project Assembly & Publishing

| Tool | Purpose | Key Inputs |
|------|---------|------------|
| `assemble_project` | Copy SE2 template, inject contracts/tests/pages into correct directories. Returns `projectPath`. | `projectId`, `contracts: [{name, content}]`, optional `pages`, `tests` |
| `export_project` | Export all assembled project files as `[{path, content}]` so you can write them to the user's local machine. Use for testnet/mainnet deploys to keep private keys local. | `projectId` (must call `assemble_project` first) |
| `push_github` | Create GitHub repo and push all project files. | `projectId`, `githubToken`, optional `repoName`, `description` |

---

## Deployment Guide

### Local Testing (Anvil) — uses test keys, safe to deploy remotely
```
1. start_anvil(projectId: "my-dapp")
   → Returns: rpcUrl (http://127.0.0.1:PORT), accounts with private keys
2. assemble_project(projectId: "my-dapp", contracts: [...], tests: [...], pages: [...])
3. deploy_local(projectId: "my-dapp", rpcUrl: <from step 1>, privateKey: <from step 1>)
```

### Testnet / Mainnet Deployment — private key stays local

**IMPORTANT**: For real networks, you MUST use `export_project` to get the files, write them locally with the `Write` tool, and then tell the user to deploy themselves. Do NOT run any `forge` commands yourself. Do NOT try to `cd` into the server-side `projectPath`.

```
1. assemble_project(projectId: "my-dapp", contracts: [...], tests: [...], pages: [...])
   → Returns: projectPath (THIS IS ON THE SERVER — do NOT use it locally)

2. export_project(projectId: "my-dapp")
   → Returns: {files: [{path: "packages/foundry/src/MyToken.sol", content: "..."}, ...]}

3. Use the Write tool to write EACH returned file to a local directory:
   For each file in the response:
     Write(file_path="/Users/<user>/my-dapp/" + file.path, content=file.content)

4. Tell the user to run these commands themselves (do NOT run them for the user):
   cd ~/my-dapp/packages/foundry
   forge install foundry-rs/forge-std --no-commit
   forge install OpenZeppelin/openzeppelin-contracts --no-commit
   forge script script/Deploy.s.sol --rpc-url <RPC_URL> --broadcast --private-key <KEY>
```

**Why**: The user's private key never leaves their machine. The MCP server only compiles, tests, and assembles — deployment with real keys happens locally.

**Requirements**: The user needs `forge` (Foundry) installed locally for mainnet/testnet deploys.

Common RPC URLs:
- **Sepolia**: `https://rpc.sepolia.org` or user's Alchemy/Infura URL
- **Base Sepolia**: `https://sepolia.base.org`
- **Base Mainnet**: `https://mainnet.base.org`
- **Ethereum Mainnet**: User provides their own RPC URL

Always ask the user for their RPC URL before deploying to a real network. Never send private keys to the MCP server for real network deployments.

---

## Solidity Guide

### Basics
- **Pragma**: `pragma solidity ^0.8.20;`
- **License**: `// SPDX-License-Identifier: MIT`
- Use `call{value: x}("")` instead of `transfer()`/`send()`
- Follow checks-effects-interactions pattern
- Add NatSpec comments (`/// @notice`, `/// @param`, `/// @return`)

### OpenZeppelin v5 Import Paths

The compiler resolves imports from unpkg CDN automatically. Use **OpenZeppelin v5.0.0** paths:

```
@openzeppelin/contracts/token/ERC20/ERC20.sol
@openzeppelin/contracts/token/ERC721/ERC721.sol
@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol
@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol
@openzeppelin/contracts/token/ERC1155/ERC1155.sol
@openzeppelin/contracts/access/Ownable.sol
@openzeppelin/contracts/access/AccessControl.sol
@openzeppelin/contracts/utils/ReentrancyGuard.sol      ← NOT security/
@openzeppelin/contracts/utils/Pausable.sol              ← NOT security/
@openzeppelin/contracts/utils/math/Math.sol             ← SafeMath is gone in 0.8+
@openzeppelin/contracts/utils/Strings.sol
@openzeppelin/contracts/interfaces/IERC20.sol
```

**Common v4 → v5 migration mistakes:**

| Wrong (v4) | Correct (v5) |
|------------|-------------|
| `@openzeppelin/contracts/security/ReentrancyGuard.sol` | `@openzeppelin/contracts/utils/ReentrancyGuard.sol` |
| `@openzeppelin/contracts/security/Pausable.sol` | `@openzeppelin/contracts/utils/Pausable.sol` |
| `@openzeppelin/contracts/security/PullPayment.sol` | `@openzeppelin/contracts/utils/PullPayment.sol` |
| `Counters.Counter` | Use plain `uint256` variable |
| `Ownable` constructor with no args | `Ownable(msg.sender)` — v5 requires initial owner |

### v5 Constructor Patterns

```solidity
// ERC20
constructor() ERC20("MyToken", "MTK") Ownable(msg.sender) {}

// ERC721
constructor() ERC721("MyNFT", "MNFT") Ownable(msg.sender) {}

// ERC1155
constructor() ERC1155("https://api.example.com/{id}.json") Ownable(msg.sender) {}
```

### Security Patterns
- Use `ReentrancyGuard` + `nonReentrant` on functions that send ETH/tokens
- Use `Ownable` or `AccessControl` for privileged functions
- Validate all inputs (zero address, overflow, empty arrays)
- Emit events for all state changes
- Use `call{value: x}("")` with success check:
  ```solidity
  (bool success, ) = payable(recipient).call{value: amount}("");
  require(success, "Transfer failed");
  ```

---

## Foundry Test Patterns

### File Convention
- Test files: `ContractName.t.sol` (the `.t.sol` suffix is required)
- Import from `forge-std/Test.sol`
- Contract inherits `Test`
- Test functions start with `test` prefix

### DO NOT USE (Hardhat/JS patterns — FORBIDDEN)
- `describe()`, `it()`, `expect()`, `before()`, `beforeEach()`
- `chai`, `mocha`, `ethers.getSigners()`, `ethers.getContractFactory()`
- `loadFixture()`, `time.increase()`
- `.test.ts`, `.test.js`, `.spec.ts` extensions
- Any JavaScript or TypeScript test files

### Template

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/ContractName.sol";

contract ContractNameTest is Test {
    ContractName public instance;
    address public owner;
    address public user1;
    address public attacker;

    function setUp() public {
        owner = address(this);
        user1 = makeAddr("user1");
        attacker = makeAddr("attacker");
        instance = new ContractName(/* constructor args */);
    }

    function testDeployment() public view {
        assertEq(instance.owner(), owner);
    }

    function testUnauthorizedAccess() public {
        vm.prank(user1);
        vm.expectRevert();
        instance.ownerOnlyFunction();
    }

    function testEmitsEvent() public {
        vm.expectEmit(true, true, false, true);
        emit ContractName.SomeEvent(arg1, arg2);
        instance.doSomething(arg1, arg2);
    }

    function testFuzz_SomeFunction(uint256 amount) public {
        vm.assume(amount > 0 && amount < 1e18);
        // test logic
    }
}
```

### Key Cheatcodes

| Cheatcode | Purpose |
|-----------|---------|
| `vm.prank(addr)` | Next call comes from `addr` |
| `vm.startPrank(addr)` / `vm.stopPrank()` | Multiple calls from `addr` |
| `vm.expectRevert()` | Next call must revert |
| `vm.expectRevert(bytes("message"))` | Must revert with specific message |
| `vm.expectEmit(true, true, false, true)` | Check event topics + data |
| `makeAddr("label")` | Create labeled address |
| `deal(addr, amount)` | Set ETH balance |
| `hoax(addr, amount)` | `deal` + `prank` combined |
| `vm.warp(timestamp)` | Set `block.timestamp` |
| `vm.roll(blockNum)` | Set `block.number` |

### Assertions
`assertEq(a, b)`, `assertTrue(x)`, `assertFalse(x)`, `assertGt(a, b)`, `assertLt(a, b)`, `assertGe(a, b)`, `assertLe(a, b)`

### Test Coverage Checklist
1. Deployment / initial state
2. All public/external functions — happy path
3. Access control (non-owner calling owner functions should revert)
4. Edge cases (zero address, max uint256, empty arrays)
5. Reentrancy attack simulation
6. Event emissions
7. State change verification
8. Fuzz tests where appropriate

---

## SE2 Frontend Guide

### Project Structure (after assemble_project)
```
packages/
  foundry/
    src/           ← Solidity contracts
    test/          ← Foundry .t.sol tests
    script/        ← Deploy scripts
    foundry.toml
  nextjs/
    app/
      pagename/
        page.tsx   ← Frontend pages
    components/    ← Shared components
    contracts/
      deployedContracts.ts  ← Auto-generated after deploy
```

### Key SE2 Hooks

**Read contract data:**
```tsx
import { useScaffoldReadContract } from "~~/hooks/scaffold-eth";

const { data: totalSupply } = useScaffoldReadContract({
  contractName: "MyToken",
  functionName: "totalSupply",
});

// With args
const { data: balance } = useScaffoldReadContract({
  contractName: "MyToken",
  functionName: "balanceOf",
  args: [address],
});
```

**Write to contract:**
```tsx
import { useScaffoldWriteContract } from "~~/hooks/scaffold-eth";

const { writeContractAsync } = useScaffoldWriteContract("MyToken");

const handleMint = async () => {
  await writeContractAsync({
    functionName: "mint",
    args: [address, amount],
    value: parseEther("0.1"), // if payable
  });
};
```

### Key SE2 Components
```tsx
import { Address } from "~~/components/scaffold-eth";     // ENS + blockie
import { Balance } from "~~/components/scaffold-eth";     // ETH balance
import { EtherInput } from "~~/components/scaffold-eth";  // ETH amount input
import { useAccount } from "wagmi";                       // Connected wallet

const { address: connectedAddress } = useAccount();
```

### Page Template
```tsx
"use client";

import { useState } from "react";
import type { NextPage } from "next";
import { parseEther, formatEther } from "viem";
import { useAccount } from "wagmi";
import { Address, Balance, EtherInput } from "~~/components/scaffold-eth";
import { useScaffoldReadContract, useScaffoldWriteContract } from "~~/hooks/scaffold-eth";

const MyPage: NextPage = () => {
  const { address: connectedAddress } = useAccount();
  const [amount, setAmount] = useState("");

  const { data: totalSupply } = useScaffoldReadContract({
    contractName: "MyToken",
    functionName: "totalSupply",
  });

  const { writeContractAsync } = useScaffoldWriteContract("MyToken");

  return (
    <div className="flex flex-col items-center gap-4 p-8">
      <h1 className="text-4xl font-bold">My dApp</h1>
      <div className="card bg-base-200 p-6 rounded-xl">
        <p>Total Supply: {totalSupply?.toString()}</p>
        <Address address={connectedAddress} />
        <Balance address={connectedAddress} />
        <EtherInput value={amount} onChange={setAmount} />
        <button
          className="btn btn-primary"
          onClick={() => writeContractAsync({
            functionName: "mint",
            value: parseEther(amount || "0"),
          })}
        >
          Mint
        </button>
      </div>
    </div>
  );
};

export default MyPage;
```

### Styling
- **TailwindCSS** + **daisyUI** — use utility classes and daisyUI components
- Common daisyUI: `btn`, `btn-primary`, `card`, `input`, `badge`, `alert`, `modal`, `tabs`
- Layout: `flex flex-col items-center`, `grid grid-cols-2 gap-4`

---

## Common Errors & Fixes

| Error | Fix |
|-------|-----|
| `Source "@openzeppelin/.../security/ReentrancyGuard.sol" not found` | Change to `utils/ReentrancyGuard.sol` (v5 path) |
| `Source "@openzeppelin/.../security/Pausable.sol" not found` | Change to `utils/Pausable.sol` (v5 path) |
| `No arguments passed to base constructor "Ownable"` | Add `Ownable(msg.sender)` to constructor |
| `Counters.Counter` undefined | Remove Counters import, use plain `uint256` |
| `SafeMath` not found | Remove — built into Solidity 0.8+ |
| `Using transfer/send` (security warning) | Use `call{value: x}("")` with success check |
| Tests use `describe()/it()` | Rewrite as Foundry Solidity tests with `testXxx()` |
| `forge test` fails with OZ import errors | The `run_tests` tool handles OZ installation automatically |

## Environment

- Solidity compiler: 0.8.20+
- OpenZeppelin: v5.0.0
- Foundry/Forge for testing
- Scaffold-ETH 2 for project template
- Next.js App Router + React + TailwindCSS + daisyUI for frontend
