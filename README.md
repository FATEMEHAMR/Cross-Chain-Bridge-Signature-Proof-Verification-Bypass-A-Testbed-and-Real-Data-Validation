# Cross-Chain Bridge Signature/Proof Verification Bypass: A Testbed and Real-Data Validation

A Foundry-based testbed that reproduces and hardens against the three main subtypes of
signature and proof verification bypass in cross-chain bridges, and validates the results
against real contracts and real chain state on Ethereum mainnet.

This repository is the practical artifact of a BSc thesis on bridge security. It accompanies
a STRIDE-based classification of real-world incidents and a quantitative gas-overhead study
of a hardening mechanism.

---

## What this project does

1. **Classifies** three subtypes of the signature/proof verification bypass attack family,
   grounded in documented real-world bridge incidents and mapped to the STRIDE threat model.
2. **Reproduces** each subtype on a minimal vulnerable bridge contract in a controlled
   two-chain testbed, and proves a shared hardening mechanism blocks all three without
   breaking the legitimate user path (a before/after protocol).
3. **Measures** the gas overhead of the hardening mechanism, including a controlled
   "gas ladder" that isolates the net cost of each security check.
4. **Validates** the findings on real data: it reproduces the real Nomad Bridge attack on
   its original contract via a mainnet fork, proves a root-cause fix blocks it, and measures
   legitimate-path gas on real historical user transactions.

The attack subtypes:

| Subtype | Reference incident | Root cause |
|---|---|---|
| Signature forgery | Nomad, Chainswap | zero-address / default value accepted as a valid attestation |
| Message replay | Polygon Plasma | no nonce or used-message tracking |
| Valid signature, wrong message | Gnosis Omni, cross-chain replay | hash omits chainId and contract address |

The hardening mechanism combines four checks in one path: reject the zero address, enforce
EIP-712 domain-separated hashing (binding chainId and contract address), enforce nonce
uniqueness, and bound signature `s` to the lower half of the group (EIP-2, malleability).
Checks 1, 2 and 4 use the official OpenZeppelin Contracts v5.1.0 libraries directly.

---

## Repository layout

This repository has two parts.

### Part 1: synthetic testbed (`bridge-testbed/`)

```
src/
  SourceBridge.sol                 lock/burn on the source chain
  vulnerable/                      one contract per attack subtype
  hardened/                        EIP-712 + ECDSA hardened bridge (single and 2-of-3 multisig)
  gascontrol/                      controlled gas ladder (L0..L4) and zero references
test/                              forge tests: attacks, defenses, gas, cross-chain flow
script/                            deploy and relayer scripts
results/                          recorded output of real runs
```

### Part 2: real-data validation (`bridge-realdata/`)

```
test/fork/
  NomadReal.t.sol                  real Nomad attack + state-fix before/after (mainnet fork)
  GasReplayReal.t.sol              gas of real legitimate user transactions
  ChainswapReal.t.sol              template for the Chainswap case
scripts/
  fetch_incidents.py               build the incident dataset from the DefiLlama Hacks API
  fetch_legit_txs.py               collect real legitimate transactions from Etherscan (V2)
  analyze_gas.py                   mean/median/stdev of measured gas
```

---

## Requirements

- [Foundry](https://book.getfoundry.sh) (tested on `1.5.1`)
- Solidity `0.8.20` (resolved by Foundry)
- For Part 2 only: an archive mainnet RPC (Alchemy free tier works), an Etherscan API key,
  and Python 3

---

## Running Part 1 (synthetic testbed)

```bash
cd bridge-testbed
forge test                 # run all tests
forge test --gas-report    # include the gas tables used in the thesis
```

All tests are self-contained and need no network.

## Running Part 2 (real-data validation)

```bash
cd bridge-realdata
cp .env.example .env        # fill in MAINNET_RPC_URL and ETHERSCAN_API_KEY
forge build

# Real Nomad attack + defense on real mainnet state
forge test --match-path test/fork/NomadReal.t.sol --rpc-url $MAINNET_RPC_URL -vvv

# Gas of real legitimate user transactions
python scripts/fetch_legit_txs.py
forge test --match-test test_ReplayLegitTxs_MeasureGas --rpc-url $MAINNET_RPC_URL -vv
python scripts/analyze_gas.py

# Incident dataset (no RPC needed)
python scripts/fetch_incidents.py
```

The fork tests require an archive RPC. The first run is slow because state is downloaded
from the RPC; Foundry caches it afterwards.

---

## Results summary

**Defense effectiveness (synthetic testbed).** All three attack subtypes succeed on the
vulnerable contracts and are blocked on the hardened contract, while the legitimate user
path stays open. The 2-of-3 multisig variant preserves all three defenses.

**Gas overhead (synthetic testbed).** The hardening mechanism adds roughly 14% gas on
average across the three subtypes. The controlled gas ladder shows the net cost of the
security checks themselves is about 4,435 gas, over 84% of which is the `ecrecover` call.
The multisig variant adds roughly 11% on top of the single-attester hardened bridge.

**Real-data validation.** The real Nomad attack was reproduced on its original contract at
block 15,259,100 (minting 100 WBTC to the attacker) and blocked by a root-cause state fix.
Legitimate-path gas measured on six real user transactions gives a median of 165,763 gas
(mean 223,584, standard deviation 137,082), confirming a real, non-degenerate distribution.

---

## Sources and attribution

- Incident patterns and attack calldata: [DeFiHackLabs](https://github.com/SunWeb3Sec/DeFiHackLabs)
- Incident dataset: [DefiLlama Hacks API](https://api.llama.fi/hacks)
- Classification framework: Notland et al., "SoK: Cross-Chain Bridging Architectural Design
  Flaws and Mitigations" ([arXiv:2403.00405](https://arxiv.org/abs/2403.00405))
- Cryptographic standards: EIP-2, EIP-712
- Libraries: OpenZeppelin Contracts v5.1.0 (`ECDSA.sol`, `EIP712.sol`, `ERC20.sol`)

See `bridge-testbed/SOURCES_AND_ETHICS.md` for a full per-file attribution.

---

## Ethics and scope

All vulnerable contracts exist only inside this local testbed and are never deployed
publicly. They are for educational and research use, to understand how these attacks work
and to demonstrate effective defenses. The threat model assumes an external attacker with
full knowledge of the contract code but no control over attester private keys, so key-leak
incidents (such as Ronin) are out of scope, as are incidents that cannot be reproduced with
an EVM fork (such as Wormhole on Solana and the BNB Chain IAVL-proof case).

---

## License

Code is provided for academic and educational use. See individual files for license headers.
Vendored OpenZeppelin and forge-std retain their original licenses.
