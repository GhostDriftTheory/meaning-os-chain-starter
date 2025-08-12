# Meaning OS × Blockchain — A-grade Starter (Commit–Reveal + OCW + JumpLog)

This is the **minimal, production-tractable** starter for anchoring Meaning OS
decisions (SHP and g-direction jumps) on-chain with:

- **Commit–Reveal** (tamper-proof provenance)
- **OCW (Open Challenge Window)** — “余白” for objections
- **Immutable JumpLog** — emits `Jump(id, s_prev, s_next, g, dphi, meta)` on acceptance

> Off-chain does φ/g evaluation. On-chain anchors *evidence & procedure* only.

## Contract
- `contracts/SHPRegistry.sol` — single-file solidity (MIT, ^0.8.20)

### Rules (v0)
- `commitSHP(id, h, ocwSeconds, sPrev, meta)`
- `revealSHP(id, s, g, dphi, proof, salt)` must satisfy `keccak256(s,g,dphi,proof,salt) == h`
- Anyone may `challenge(id, evidence)` *during OCW*
- After OCW, `finalize(id)` sets `accepted = (no challenge) && (dphi < 0)`
- If accepted ⇒ emits `Jump(...)`

You can later swap `proof` to a SNARK/STARK pointer.

## Quick Use (Hardhat)
```bash
npm init -y && npm i -D hardhat @nomicfoundation/hardhat-toolbox
npx hardhat init
# copy contracts/SHPRegistry.sol into your project, then:
npx hardhat compile
```

## Minimal Flow (CLI-ish)
1) Choose `id` and compute `h = keccak256(s,g,dphi,proof,salt)` off-chain.
2) `commitSHP(id, h, ocwSeconds, sPrev, meta)`
3) `revealSHP(id, s, g, dphi, proof, salt)`
4) Wait until `ocwDeadline(id)` (no challenge) then `finalize(id)`
5) Watch `Jump` events as your immutable Meaning OS trail.

## MinorityGuard (Γ-floor)
- `contracts/MinorityGuard.sol` implements the **minimum minority mass** guard.
- Deploy with `theta` (e.g., number of members or weighted score).
- `postGamma(id, mass, cert)` — certifies group mass and links to decision id.
- `hasGamma(id)` — check if active and meets threshold.

Integrate with `SHPRegistry`:
- During `challenge(...)` in SHPRegistry, you can query `MinorityGuard.hasGamma(id)`
  to grant extended OCW or fee waivers.


---

## MinorityGuard (Γ-floor)

Adds a guardian that can extend OCW for revealed records when a posted Γ-certificate
meets threshold `θ`.

- Deploy `SHPRegistry` (v1.1)
- Deploy `MinorityGuard(registry, θ, extendSeconds)`
- Call `registry.setGuardian(minorityGuard)`
- Flow: minority calls `postGamma(id, cert, mass)`; if `mass ≥ θ`, guard calls `extendOCW(id, extendSeconds)`

**Caps:** cumulative extension per record is limited to `MAX_OCW_EXTENSION` in the registry.


### EIP-712 Γ-certificate (Multisig)

`contracts/MinorityGuard712.sol` implements a simple EIP-712 multisig:

- Admin whitelists signers via `setSigners(address[], enabled)`
- A message `Gamma{id, mass, nonce}` is signed by each signer
- Client collects `signatures[]` and calls `postGamma712(id, mass, nonce, signatures)`
- If **unique approvals ≥ θ**, OCW is extended by `extendSeconds`

**Domain**: name=`MinorityGuard`, version=`1`, chainId and verifyingContract bound.
`TYPEHASH = keccak256("Gamma(bytes32 id,uint256 mass,uint256 nonce)")`

## Minimal CLI

Install deps:
```bash
npm i
```

Set env (copy `.env.example` → `.env` and fill). Then:

```bash
# 1) Prepare (computes commitment hash h and saves local state)
node cli/cli.js prepare --id case1 --s "S*" --g "g^" --dphi -1 --ocw 3600

# 2) Commit
node cli/cli.js commit --id case1

# 3) Reveal
node cli/cli.js reveal --id case1

# 4) (Optional) Sign Γ on two different signer keys to meet θ=2
#    Use different PRIVATE_KEY per signer to produce signatures
node cli/cli.js sign-gamma --id case1 --mass 7 --nonce 1

# 5) Post Γ (collector account submits both signatures)
node cli/cli.js post-gamma --id case1 --mass 7 --nonce 1 --sigs 0xSigA,0xSigB

# 6) Finalize (after OCW deadline; check deadline with `deadline`)
node cli/cli.js finalize --id case1
```

Local state is saved in `.meaning-cli/state/<id>.json`.


## Foundry Fuzz / Invariant Tests

We ship a Foundry test suite focused on **invariants & fuzzing** for `SHPRegistry`.

Setup:
```bash
forge --version   # ensure Foundry is installed
forge install foundry-rs/forge-std
forge test -vv
```

Config comes from `foundry.toml` (src=`contracts`, test=`foundry/test`).

Properties covered:
- **Deadline monotonicity & cap**: OCW extension strictly increases the deadline and never exceeds `MAX_OCW_EXTENSION`.
- **Finalize rule**: accepted iff (no challenge) & (dphi < 0).
- **Challenge window**: challenge reverts after `ocwDeadline`.
- **Commit–Reveal binding**: any mismatch on reveal reverts.
