# Using UMA Optimistic Oracle V3 on Arc Testnet  A General Guide

A practical, project-agnostic guide for any developer who wants **trustless, decentralized "truth"** in their Arc dApp. UMA's Optimistic Oracle lets a smart contract ask "is this statement true?" and get an answer that is economically guaranteed to be honest  without any trusted admin.

This guide covers what UMA is, why you'd use it, how to deploy it on Arc Testnet, and how to integrate it into your own contracts.

---

## 1. What is the Optimistic Oracle?

An **oracle** brings off-chain truth on-chain. UMA's **Optimistic Oracle V3 (OOv3)** does this with a simple, powerful idea: *assume any claim is true unless someone disputes it.*

The flow has three steps:

1. **Assert**  Someone posts a statement ("Team A won the match") and locks a **bond** (collateral).
2. **Challenge window**  For a set period (the *liveness*), anyone can dispute the statement by posting an equal bond.
3. **Settle**  If nobody disputes, the statement is accepted as true and the asserter gets their bond back. If disputed, an oracle vote decides; the loser's bond is forfeited to the winner.

Because lying costs you your bond, honest assertions dominate. This is the same oracle pattern that secures large prediction markets and insurance protocols.

### When to use it

- Prediction markets ("which outcome happened?")
- Insurance / parametric payouts ("did the flight get delayed?")
- KPI options, grants, bug bounties ("was the milestone met?")
- Any contract that needs a human-verifiable fact that can't come from a price feed.

### When **not** to use it

- Real-time price data (use a price oracle instead).
- Facts needed instantly (the challenge window adds latency).
- Purely on-chain data your contract can already read.

---

## 2. Why Arc is a good fit

Arc is EVM-compatible and uses **USDC as its native/gas token**. That means UMA bonds can be denominated in real USDC rather than a throwaway token, which matches how production prediction markets actually work. Standard EVM tooling (Foundry, Hardhat, viem, ethers) works out of the box.

Key Arc Testnet parameters:

| Item | Value |
|---|---|
| Chain ID | `5042002` |
| RPC | `https://rpc.testnet.arc.network` |
| Explorer | `https://testnet.arcscan.app` |
| Faucet | `https://faucet.circle.com` |
| USDC (ERC-20) | `0x3600000000000000000000000000000000000000` (6 decimals) |

---

## 3. Deploying UMA on Arc Testnet

UMA Labs has **not** officially deployed OOv3 on Arc yet, so you deploy your own **sandbox** environment using UMA's official quickstart. This gives you a fully functional oracle for development and testing.

### Step 1  Clone the sandbox

```bash
git clone https://github.com/UMAprotocol/dev-quickstart-oov3.git
cd dev-quickstart-oov3
forge install
```

### Step 2  Critical: fit OOv3 under Arc's size limit

OptimisticOracleV3's bytecode is larger than the EVM 24,576-byte contract-size limit when compiled normally, and Arc enforces this limit. Enable aggressive optimization so it fits:

```toml
# foundry.toml
[profile.default]
optimizer = true
optimizer_runs = 1
via_ir = true
```

`optimizer_runs = 1` optimizes for **size** (not speed), and `via_ir = true` uses the IR pipeline for tighter bytecode. This typically brings OOv3 well under the limit (~15 KB). Note: `via_ir` compiles slowly  several minutes is normal.

Verify the size before deploying:

```bash
forge build --sizes | grep OptimisticOracleV3   # should be < 24,576
```

### Step 3  Deploy the sandbox

```bash
export PRIVATE_KEY=0xYOUR_KEY
export MINIMUM_BOND=0   # optional: simplifies early testing

forge script script/OracleSandbox.s.sol \
  --rpc-url https://rpc.testnet.arc.network \
  --private-key $PRIVATE_KEY \
  --broadcast --legacy --skip-simulation
```

> Tip: `--skip-simulation` is important on Arc. Arc's native USDC uses special precompiles (e.g. a blocklist check) that Foundry's local simulation can't replicate, causing false `StackUnderflow` reverts. Skipping simulation sends the real transaction, which succeeds on-chain.

This deploys and wires together: `Finder`, `Store`, `AddressWhitelist`, `IdentifierWhitelist`, `MockOracleAncillary`, a `TestnetERC20`, and **`OptimisticOracleV3`**. Save the logged addresses  especially OOv3's.

### Step 4  Confirm OOv3 is live

```bash
cast code <OOV3_ADDRESS> --rpc-url https://rpc.testnet.arc.network | head -c 20
# a long 0x60806040... means success; bare "0x" means it didn't deploy
```

---

## 4. Using USDC as the bond currency

By default the sandbox uses its own `TestnetERC20` for bonds. To use **real Arc USDC** instead, you must (a) whitelist it and (b) set a final fee.

```bash
# Whitelist USDC as a valid bond currency
cast send <ADDRESS_WHITELIST> "addToWhitelist(address)" \
  0x3600000000000000000000000000000000000000 \
  --rpc-url https://rpc.testnet.arc.network --private-key $PRIVATE_KEY --legacy

# Set a final fee (e.g. 1 USDC = 1000000, since USDC has 6 decimals)
cast send <STORE> "setFinalFee(address,(uint256))" \
  0x3600000000000000000000000000000000000000 "(1000000)" \
  --rpc-url https://rpc.testnet.arc.network --private-key $PRIVATE_KEY --legacy
```

---

## 5. Integrating OOv3 into your contract

The interface you need is small:

```solidity
interface IOptimisticOracleV3 {
    function defaultIdentifier() external view returns (bytes32);
    function getMinimumBond(address currency) external view returns (uint256);
    function assertTruth(
        bytes calldata claim,
        address asserter,
        address callbackRecipient,
        address escalationManager,
        uint64 liveness,
        address currency,
        uint256 bond,
        bytes32 identifier,
        bytes32 domainId
    ) external returns (bytes32 assertionId);
    function settleAndGetAssertionResult(bytes32 assertionId) external returns (bool);
}
```

### Making an assertion

```solidity
function assertSomething(bytes memory claim, uint256 outcome) external {
    uint256 bond = oo.getMinimumBond(address(usdc));
    usdc.transferFrom(msg.sender, address(this), bond); // pull bond from proposer
    usdc.approve(address(oo), bond);                    // let OO take it

    assertionId = oo.assertTruth(
        claim,             // human-readable statement
        msg.sender,        // asserter
        address(this),     // callbackRecipient (your contract)
        address(0),        // escalationManager (none)
        7200,              // liveness in seconds (2 hours)
        address(usdc),     // bond currency
        bond,
        oo.defaultIdentifier(),
        bytes32(0)         // domainId
    );
}
```

### Settling after the challenge window

```solidity
function settle() external {
    bool truthful = oo.settleAndGetAssertionResult(assertionId);
    require(truthful, "assertion was not accepted");
    // ... act on the verified result here
}
```

### Optional callbacks

Implement these to react automatically when UMA resolves or disputes your assertion:

```solidity
function assertionResolvedCallback(bytes32 id, bool truthful) external {
    require(msg.sender == address(oo), "only oo");
    // finalize your logic based on `truthful`
}
function assertionDisputedCallback(bytes32 id) external {
    require(msg.sender == address(oo), "only oo");
    // a dispute was raised; final outcome comes via resolvedCallback later
}
```

---

## 6. The end-to-end usage flow

Once deployed and integrated, using the oracle looks like this for end users:

1. **Propose**  A user calls your assert function, approving the USDC bond first. Your contract forwards the claim to OOv3.
2. **Wait**  The liveness window (e.g. 2 hours) runs. Anyone can dispute during this time.
3. **Settle**  After the window, anyone calls your settle function. If undisputed, the result is accepted and your contract acts on it; the proposer's bond is returned.
4. **Disputes**  If disputed, the bonds are held until the oracle layer resolves the question; the wrong party loses their bond.

---

## 7. Honest limitations on testnet

Be transparent about scope:

- The **full mechanism** (bonding, liveness, assert/dispute/settle, incentives) runs on-chain and is real.
- The **final dispute-resolution vote** in a self-deployed sandbox is handled by a **MockOracle** you control  not UMA's decentralized voting network (DVM), which only exists where UMA is officially deployed.
- On a network with an official UMA deployment, you'd point your contract at the canonical OOv3 address and the real DVM handles disputes  usually a one-line change.
- Everything here is **testnet only**. Use test USDC; never real funds.

---

## 8. Useful references

- UMA quickstart: `github.com/UMAprotocol/dev-quickstart-oov3`
- UMA docs: `docs.uma.xyz`
- Arc docs: `docs.arc.io`
- Sandbox pattern: UMA's `OracleSandbox.s.sol` deploys a complete local oracle environment.

---

*This guide is project-agnostic: the same steps let any Arc dApp  prediction markets, insurance, grants, KPI options  obtain trustless off-chain truth via UMA's Optimistic Oracle V3.*
