# Trust Wallet QR Display/Execution Mismatch — Investigation Log

> **Session:** Company Scout · Profile: `company-scout`  
> **Date Range:** June–July 2026  
> **Status:** ❌ EIP-681 `data` override BLOCKED — Dapp browser angle UNTESTED  
> **Portal:** `investigation-portal.html` (open to resume)

---

## 1. THE GOAL

Demonstrate a **display/execution mismatch** in Trust Wallet's QR scanner:

| Field | Display (what user sees) | Execute (what actually happens) |
|---|---|---|
| Amount | **1 USDT** (`uint256=1000000`) | **4 USDT** (`data` encodes `transfer(RCP,4000000)`) |
| Recipient | Address A (shown) | Address B (in data) |

Severity target: **High** (CVSS ~7.5) — HackerOne reportable.

---

## 2. ARCHITECTURE (From wallet-core Source)

### 2.1 Protobuf Types (Transaction Routing)

The wallet-core library has three relevant transaction protos:

| Proto | Has `data` field? | Triggered by | Notes |
|---|---|---|---|
| `ERC20Transfer` | **NO** | `amount` param OR `/transfer?address+uint256` | Builds own calldata from params — `data` IGNORED |
| `Transfer` (native ETH) | **YES** | No `amount`, no function match | `data` passes through but shows ETH amount |
| `ContractGeneric` | **YES** | No function match, unknown params | Shows generic "Contract Interaction" |

**Key rule:** `ERC20Transfer` has **no `data` field** in its proto definition. Any `data` from the URI is stripped during transaction building.

### 2.2 Routing Decision Tree

```
URI → EIP681Parser
  ├─ Has `amount` param? → ERC20Transfer (data ignored)
  ├─ Has `/transfer` + `address` + `uint256`? → ERC20Transfer (data ignored)
  ├─ Has `/approve` + `address` + `uint256`? → ContractGeneric (data MAY pass)
  ├─ Has `value` param (no amount)? → Transfer (data passes, shows ETH)
  └─ Has `data` only (no function match)? → ContractGeneric (data passes)
```

### 2.3 Critical Finding

When `data` is ADDED to a `/transfer` URI that also has `address`+`uint256`:
1. The parser CAN still match to ERC20Transfer (address+uint256 present)
2. ERC20Transfer proto has NO data field → data is **STRIPPED**
3. Wallet shows 1 USDT, but executes transfer(RCP, 1000000) = honest 1 USDT
4. NO mismatch occurs

v7/B confirmed: adding `data` to `/transfer` sometimes BREAKS routing, showing ETH instead of USDT — proving data breaks pattern matching, but doesn't pass through.

---

## 3. ALL TEST PAGES CREATED

| # | Page | What it tested | Result |
|---|---|---|---|
| 1 | `amount-data-test.html` | trust:// + amount + data together | Data ignored (erc20_transfer) |
| 2 | `mainnet-v2-no-address.html` | trust:// w/o address, amount + data | Data ignored |
| 3 | `mainnet-v3-final-angles.html` | memo, duplicate uint256, value confusion | None worked |
| 4 | `mainnet-v4-contract-generic.html` | trust:// send without asset param | Token picker → empty fields |
| 5 | `mainnet-v5-android-focus.html` | ethereum: with value + data, /drain | C showed 0 ETH, confirmed, paid gas, no USDT transfer |
| 6 | `mainnet-v6-android-pure-data.html` | pure data only, all formats | All showed 0 ETH or empty fields |
| 7 | `mainnet-v7-value-data.html` | value=0.0001 + data to enable confirm | 0 ETH shown, confirm disabled |
| 8 | `mainnet-v8-wei-params.html` | wei integers, param order, /approve | JS bug (unescaped quote), QR not rendering |
| 9 | `mainnet-v9-static.html` | 8 static tests: wei, data-first, /approve, trust:// | ALL — data stripped, no USDT transfer |
| 10 | `mainnet-v10-value-zero.html` | value=0 explicit, no /transfer | "Transaction likely to fail" — confirm disabled |
| 11 | `mainnet-v11-dapp-browser.html` | trust://browser?url= dapp angle | QRs broken (self-referencing) — **UNTESTED** |
| 12 | `mainnet-v12-trust-raw.html` | trust:// raw scheme | 404 — never committed |
| 13 | `mainnet-v13-definitive.html` | User's exact insight format + 7 variants | **READY TO TEST** |
| 14 | `mainnet-v14-dapp-browser.html` | dapp browser with Web3 eth_sendTransaction | **READY TO TEST** |
| — | `dapp-payload.html` | Actual dapp page for v14 | Committed |

---

## 4. TEST RESULTS (WHAT HAPPENED)

### 4.1 EIP-681 `/transfer?address=RCP&uint256=X&data=...` (THE TARGET FORMAT)

**Tested in:** v5/B, v7/B, v9/C/H, v13/A/B/C/D

| Observation | Count |
|---|---|
| Showed ETH instead of USDT | Confirmed (v7/B) |
| Showed USDT amount (honest, no override) | Confirmed (amount+asset tests) |
| Showed 0 ETH, confirm enabled, transaction executed | Confirmed (v5/C) |
| USDT transferred | **NEVER** |
| Gas paid with no USDT movement | Confirmed (v5/C) |

**Conclusion:** `data` is STRIPPED. ERC20Transfer proto has no data field.

### 4.2 `ethereum:0xCONTRACT@1?value=X&data=...` (VALUE + DATA)

**Tested in:** v5/A, v7/A, v9/A/B, v10/A/B

| Observation | Count |
|---|---|
| Shows "X ETH to CONTRACT" | Confirmed |
| "Transaction likely to fail" warning | Confirmed (USDT contract can't receive ETH) |
| Confirm button disabled | Confirmed |
| Any USDT transferred | **NEVER** |

**Conclusion:** Wallet pre-checks USDT contract interaction with ETH value > 0 and blocks. Even `value=0` shows "likely to fail".

### 4.3 `trust://send?asset=TOKEN&address=RCP&amount=X&data=...`

**Tested in:** v4/A/B, v6/C/D, v10/E

| Observation | Count |
|---|---|
| Camera prompts token selection (ETH or USDT) | Confirmed |
| Fields appear empty after selection | Confirmed |
| Performs honest erc20_transfer (no data override) | Confirmed |

**Conclusion:** `trust://` format is intercepted by camera scanner and re-routed to token send flow. Data lost.

### 4.4 `trust://send?address=CONTRACT&data=...` (NO ASSET, NO AMOUNT)

**Tested in:** v4/B, v9/F, v10/D

| Observation | Count |
|---|---|
| Camera prompts token selection | Confirmed |
| Fields empty after selection | Confirmed |

**Conclusion:** Same as above — camera intercepts trust:// and routes to token flow.

### 4.5 `ethereum:0xCONTRACT@1?data=...` (PURE DATA, NO FUNCTION)

**Tested in:** v6/A, v9/G, v10/A/D

| Observation | Count |
|---|---|
| Shows 0 ETH or "Contract Interaction" | Confirmed |
| Confirm button disabled ("likely to fail") | Confirmed |
| Data passed through? | **NEGATIVE** — no USDT transfer on forced confirm |

**Conclusion:** Even ContractGeneric proto doesn't preserve data in the unsigned transaction. Wallet pre-checks block.

### 4.6 `/approve?address=RCP&uint256=X&data=...`

**Tested in:** v9/E, v10/E

| Observation | Count |
|---|---|
| Shows "Approve" if /approve recognized | Partial |
| Data stripped | Confirmed |

**Conclusion:** Same issue — /approve maps to ContractGeneric which may lose data, or wallet builds own approve calldata.

---

## 5. WHAT ALMOST WORKED

| Test | Format | What happened | Why it failed |
|---|---|---|---|
| **v5/C** | `ethereum:0xUSDT@1/drain?address=RCP&uint256=1M&data=<4U>` | Transaction **EXECUTED** ($0.04 gas paid) | Data was stripped → 0 ETH sent to USDT = no-op |
| **v7/B** | `/transfer?addr=RCP&uint256=1&data=<4U>` | Wallet showed **ETH instead of USDT** | Proved data broke erc20 routing, but data still stripped in fallback |
| **v9/E** | `/approve?data=...` | Showed generic interaction | Data stripped — same root cause |

**Closest is v5/C** — the only test where a transaction was confirmed and gas was paid. If `data` had been included, 4 USDT would have transferred.

---

## 6. ROOT CAUSE

The wallet-core proto `ERC20Transfer` **has no `data` field**. Full stop.

```
message ERC20Transfer {
    string to = 1;          // recipient
    string amount = 2;      // token amount
    string contract_address = 3;
    // NO data field!
}
```

Compare with `Transfer` (native ETH):
```
message Transfer {
    string amount = 1;      // ETH amount in wei
    bytes data = 2;         // <-- HAS data field
}
```

And `ContractGeneric`:
```
message ContractGeneric {
    string amount = 1;      // in wei
    bytes data = 2;         // <-- HAS data field
    bool available = 3;
}
```

**The `data` override attack only works if the wallet app's Android/iOS code explicitly extracts `data` from the EIP-681 URI and adds it to the transaction request object — bypassing wallet-core's proto layer.** This would be a custom extension in the app, not in wallet-core.

---

## 7. UNTESTED ANGLES

### 7.1 Dapp Browser (v14) — HIGH POTENTIAL

**Format:** `trust://browser?url=https://.../dapp-payload.html`

**Mechanism:** Dapp browser has full Web3 provider access. The dapp calls `eth_sendTransaction` directly with:
```json
{
  "to": "0xUSDT",
  "value": "0x0",
  "data": "0xa9059cbb0000...<TRANSFER_CALLDATA>",
  "gas": "0x..."
}
```
The wallet shows the **real transaction** on its confirm screen. No EIP-681 parser involved. No data stripping.

**Status:** Page committed at `dapp-payload.html`. **Untested.**

### 7.2 Trust Wallet Browser Extension (Not Mobile)

The extension might handle EIP-681 differently than mobile.

### 7.3 Trust Wallet iOS vs Android Differences

All testing has been on Android. iOS might have a different EIP-681 handler.

### 7.4 Older Trust Wallet Versions

The vulnerability may have existed in older versions and been patched. Testing on an older APK might work.

### 7.5 Non-USDT ERC-20 Tokens

USDT specifically might have protections. Testing with a custom token on testnet could show different behavior.

---

## 8. ALL QR URI FORMATS TESTED

```
// EIP-681 with /transfer
ethereum:0xTOKEN@1/transfer?address=RCP&uint256=AMOUNT&data=CALLDATA

// EIP-681 with value + data  
ethereum:0xTOKEN@1?value=ETH_WEI&data=CALLDATA

// EIP-681 pure data
ethereum:0xTOKEN@1?data=CALLDATA

// EIP-681 value=0 + data
ethereum:0xTOKEN@1?value=0&data=CALLDATA

// EIP-681 with /drain (non-standard function)
ethereum:0xTOKEN@1/drain?address=RCP&uint256=AMOUNT&data=CALLDATA

// EIP-681 with /approve
ethereum:0xTOKEN@1/approve?address=RCP&uint256=AMOUNT&data=CALLDATA

// EIP-681 duplicate uint256 (value confusion)
ethereum:0xTOKEN@1/transfer?uint256=0&address=RCP&uint256=AMOUNT

// trust:// send without amount
trust://send?asset=TOKEN&address=RCP&data=CALLDATA

// trust:// send with amount + data
trust://send?asset=TOKEN&address=RCP&amount=X&data=CALLDATA

// trust:// raw (no asset, no amount)
trust://send?address=CONTRACT&data=CALLDATA

// trust:// browser dapp
trust://browser?url=https://.../dapp.html
```

---

## 9. RESOURCES

| Resource | URL |
|---|---|
| GitHub Repo | `github.com/strystark-hash/tw-qr-attack-test-suite` |
| Investigation Portal | `investigation-portal.html` |
| v13 (EIP-681 definitive) | `mainnet-v13-definitive.html` |
| v14 (Dapp browser) | `mainnet-v14-dapp-browser.html` |
| Dapp payload | `dapp-payload.html` |
| Original generator | `mainnet-transfer-4.html` |
| Wallet-core source | `github.com/trustwallet/wallet-core` |
| Protocol buffers | `src/proto/Ethereum.proto` |
| EIP-681 parser | `src/Ethereum/EIP681Parser.cpp` (verify path) |
| Transaction builder | `src/Ethereum/TxBuilder.cpp` (verify path) |

---

## 10. QUICK RESUME

To resume this investigation:

1. Open `investigation-portal.html` on GitHub
2. The portal page has links to all pages and the current status
3. Test priority: **v14 dapp browser** → **v13 definitive** → **older wallet version**
4. If dapp browser works: the exploit is via `eth_sendTransaction` with crafted `data`, not EIP-681
5. If nothing works on current version: try an older Trust Wallet APK (pre-2024)

---

*End of investigation log. Created 2026-07-02.*