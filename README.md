# BitLynx DEX Protocol Documentation

## Overview

BitLynx is an Automated Market Maker (AMM) decentralized exchange protocol built on Stacks Layer 2, leveraging Bitcoin's security through blockchain finality. Designed for seamless trading of SIP-010 tokens, it combines capital efficiency with institutional-grade safety mechanisms like protocol-controlled liquidity and MEV-resistant oracles.

---

## Key Features

### 1. **Bitcoin-Finalized Security**

- All transactions inherit Bitcoin's immutable security via Stacks L2 settlements.
- Critical operations (pool creation, governance) require Bitcoin block confirmations.

### 2. **Concentrated Liquidity Pools**

- Liquidity providers (LPs) earn fees proportional to capital efficiency.
- Dynamic share calculation:  
  `shares = min((amountX * totalShares)/reserveX, (amountY * totalShares)/reserveY)`

### 3. **Protocol-Controlled Liquidity**

- 0.3% swap fee retained by protocol (adjustable via `PROTOCOL-FEE`).
- Fees distributed to governance token holders or reinvested into pools.

### 4. **Emergency Safeguards**

- **Circuit Breakers**: Contract owner can trigger emergency shutdown via `set-emergency-shutdown`.
- Deadline enforcement on all transactions (`deadline` parameter).

### 5. **SIP-010 Token Standard Compliance**

- Whitelist system via `approved-tokens` map.
- Token validity enforced in swaps/liquidity operations.

### 6. **MEV-Resistant Oracles**

- Time-Weighted Average Price (TWAP) tracking through cumulative price storage:
  - `cumulative-price-x/y` updated on every swap.
- Oracle data accessible via `get-pool-details`.

### 7. **Governance-Minimized Design**

- Admin-restricted functions: pool creation, token approval, emergency controls.
- Optional upgrade path via `governance-token` for future improvements.

---

## Technical Specification

### Core Data Structures

#### **Pools**

```clarity
{token-x, token-y} => {
  liquidity: uint,       // Pool activity counter
  reserve-x: uint,       // Token X reserves
  reserve-y: uint,       // Token Y reserves
  total-shares: uint,    // LP shares outstanding
  last-stacks-block-height: uint,
  cumulative-price-x: uint,  // TWAP numerator (X denominated in Y)
  cumulative-price-y: uint   // TWAP numerator (Y denominated in X)
}
```

#### **Liquidity Providers**

```clarity
{pool-id, provider} => {shares: uint}
```

#### **Price Oracles**

```clarity
principal => {
  price: uint,           // Last recorded price
  last-update: uint,     // Block height of update
  valid-period: uint     // Price validity duration
}
```

---

## Core Functions

### 1. Pool Management

#### `create-pool`

- **Purpose**: Initialize new trading pair
- **Requirements**:
  - Caller = `CONTRACT-OWNER`
  - Tokens must be SIP-010 approved
  - `token-x ≠ token-y`
- **Storage**: Creates entry in `pools` map

### 2. Liquidity Operations

#### `add-liquidity`

- **Workflow**:
  1. Validate pool existence & token approval
  2. Calculate LP shares via geometric mean (initial) or proportional contribution
  3. Transfer tokens from provider to contract
  4. Update reserves and mint shares

#### `remove-liquidity` (Implied)

- _Note: While not explicitly shown, standard AMM logic would allow burning shares to withdraw proportional reserves_

### 3. Swaps

#### `swap-exact-tokens`

- **Price Calculation**:
  ```
  amountOut = (inputAmount * (1000 - 3) * outputReserve) /
              (inputReserve * 1000 + inputAmount * (1000 - 3))
  ```
- **Safety Checks**:
  - `amountOut ≥ min-amount-out`
  - `block-height ≤ deadline`
  - Oracle price updates during swap

### 4. Governance

#### `set-emergency-shutdown`

- Immediately halts all swaps when activated
- Caller must be `CONTRACT-OWNER`

#### `approve-token`

- Whitelist new SIP-010 tokens for trading
- Restricted to contract owner

---

## Error Handling

| Code                    | Description                     |
| ----------------------- | ------------------------------- |
| `ERR-NOT-AUTHORIZED`    | Unauthorized governance action  |
| `ERR-POOL-EXISTS`       | Duplicate pool creation         |
| `ERR-SLIPPAGE-TOO-HIGH` | Price impact exceeds user limit |
| `ERR-DEADLINE-PASSED`   | Transaction expired             |

---

## Security Architecture

### 1. Transaction Safety

- **Reentrancy Protection**: Native Clarity safety + atomic transfers
- **Input Validation**:
  - Zero-amount checks (`ERR-ZERO-AMOUNT`)
  - Token pair validity (`ERR-INVALID-PAIR`)

### 2. Oracle Integrity

- TWAP calculated using cumulative price ratios:
  ```
  priceX = cumulative-price-x / (currentBlock - lastUpdateBlock)
  ```
- Minimum update intervals enforced via `valid-period`

### 3. Liquidity Protections

- Minimum liquidity burn (`MIN-LIQUIDITY = 1000`)
- Front-running resistance through deadline enforcement

---

## Integration Guide

### Querying Pool Data

```clarity
(get-reserves 'STACKS-TOKEN 'BITCOIN-TOKEN)
;; Returns {reserve-x: u50000, reserve-y: u3200}
```

### Performing Swaps

```clarity
(swap-exact-tokens token-in token-out
  u1000000  ;; 1.0 tokens in
  u950000   ;; Min 0.95 tokens out
  u189000)  ;; Deadline block
```

### LP Position Management

```clarity
(add-liquidity stx-token btc-token
  u10000000  ;; 10 STX
  u500000    ;; 0.5 BTC
  u999       ;; Min shares accepted
  u189000)
```

---

## Governance Model

### Admin Privileges

- **Contract Owner** (`CONTRACT-OWNER`):
  - Pool creation
  - Emergency shutdown
  - Token whitelisting

### Future Upgrades

- `governance-token` allows transition to DAO control
- Upgrade process requires token holder voting (implementation pending)
