# Clarity 4 Features - Deep Dive

This document provides an in-depth exploration of how each Clarity 4 feature is implemented in the lending protocol.

## Feature #1: `contract-hash?` - On-chain Contract Verification

### What It Does
Returns a `(buff 32)` hash of another contract's source code, enabling on-chain verification of contract implementations.

### Why It Matters
Before Clarity 4, contracts had no way to verify that another contract followed a specific implementation. This made trustless interactions difficult, especially for:
- Cross-chain bridges
- NFT marketplaces supporting arbitrary collections
- DeFi protocols calling external liquidators

### Our Implementation

**Location:** `contracts/lending-pool.clar:71-83`

```clarity
(define-public (register-verified-liquidator (liquidator principal))
  (begin
    (asserts! (is-eq tx-sender (var-get admin)) err-owner-only)
    (match (contract-hash? liquidator)
      hash-value
        (begin
          (var-set verified-liquidator-hash (some hash-value))
          (ok hash-value))
      err-contract-verification-failed)))
```

**How It Works:**
1. Admin registers a liquidator contract after auditing its code
2. `contract-hash?` generates a hash of the liquidator's source code
3. Hash is stored on-chain
4. Before executing liquidation, the pool verifies the hash matches
5. If code changes, the hash changes, preventing unaudited code execution

**Real-World Impact:**
- Only audited liquidators can interact with user funds
- Community can verify which liquidator versions are approved
- Prevents rug-pull attacks via malicious liquidators

---

## Feature #2: `restrict-assets?` - Contract Post-Conditions

### What It Does
Sets post-conditions that automatically revert transactions if the called contract moves assets beyond allowed limits.

### Why It Matters
When calling external contracts (like liquidators or DEX routers), you need to ensure they only move the expected assets. Pre-Clarity 4, this required complex logic and trust assumptions.

### Our Implementation

**Location:** `contracts/lending-pool.clar:310-325`

```clarity
(let (
  (restriction-result (restrict-assets? 
    (contract-of liquidator)
    (list { 
      asset: 'STX, 
      amount: liquidation-amount,
      sender: (as-contract tx-sender)
    }))))
  
  (asserts! restriction-result err-asset-restriction-failed)
  (try! (contract-call? liquidator liquidate borrower total-debt))
  ...)
```

**How It Works:**
1. Pool calculates exact liquidation amount (debt + 10% bonus)
2. `restrict-assets?` sets a hard limit on STX the liquidator can move
3. Liquidator executes its logic
4. If liquidator tries to move more STX than allowed, entire transaction reverts
5. Pool funds remain protected even if liquidator is malicious

**Security Benefits:**
- Automatic protection against over-payment
- No need to trust external contracts
- Composability without security trade-offs

---

## Feature #3: `stacks-block-time` - Block Timestamps

### What It Does
Returns the timestamp (in seconds since Unix epoch) of the current Stacks block.

### Why It Matters
Time-based DeFi logic is essential for:
- Interest accrual
- Vesting schedules
- Expiration dates
- Lockup periods
- Oracle price freshness

### Our Implementations

#### Interest Accrual (Lending Pool)
**Location:** `contracts/lending-pool.clar:147-158`

```clarity
(define-read-only (calculate-current-interest (user principal))
  (match (map-get? user-loans { user: user })
    loan-data
      (let (
        (time-elapsed (- stacks-block-time (get last-interest-update loan-data)))
        (principal-amt (get principal-amount loan-data))
        (new-interest (/ (* (* principal-amt INTEREST-RATE-BPS) time-elapsed) u315360000000)))
        (ok (+ (get interest-accrued loan-data) new-interest)))
    (ok u0)))
```

**Calculation:**
- Tracks when loan was last updated
- Calculates seconds elapsed using `stacks-block-time`
- Applies interest rate proportionally: `(principal * rate * time) / seconds-per-year`
- Compounds interest continuously

#### Price Freshness (Oracle)
**Location:** `contracts/oracle/price-oracle.clar:50-60`

```clarity
(define-read-only (get-price-status (asset (string-ascii 10)))
  (let (
    (price-data (unwrap! (map-get? prices { asset: asset }) ...))
    (age (- stacks-block-time (get last-updated price-data)))
    (is-fresh (< age MAX-PRICE-AGE)))
    
    (if is-fresh
      (ok "ACTIVE: Price feed is fresh and reliable")
      (ok "STALE: Price feed requires update"))))
```

**Price Staleness:**
- Each price update is timestamped with `stacks-block-time`
- Before using price, check age: `current_time - last_updated`
- If age > 1 hour, price is considered stale
- Prevents using outdated prices for liquidations

#### Governance Timelock
**Location:** `contracts/governance/protocol-governance.clar:150-155`

```clarity
(define-public (execute-proposal (proposal-id uint))
  (let ((proposal (unwrap! (map-get? proposals { proposal-id: proposal-id }) ...)))
    (asserts! (>= stacks-block-time (get execution-time proposal))
      err-timelock-not-expired)
    ...))
```

**Time-Locked Execution:**
- Proposals have 24-hour delay before execution
- `execution-time` = proposal creation time + 24 hours
- Can only execute when `stacks-block-time >= execution-time`
- Prevents instant malicious changes

---

## Feature #4: `to-ascii?` - Value to ASCII Conversion

### What It Does
Converts simple values (booleans, uints, principals) to ASCII strings for human-readable output.

### Why It Matters
- Generate readable transaction messages
- Create audit logs
- Enable cross-chain communication
- Improve UX with descriptive outputs

### Our Implementations

#### Loan Status Report
**Location:** `contracts/lending-pool.clar:345-365`

```clarity
(define-read-only (get-loan-status-ascii (user principal))
  (match (map-get? user-loans { user: user })
    loan-data
      (let (
        (principal-ascii (unwrap! (to-ascii? (get principal-amount loan-data)) ...))
        (interest-ascii (unwrap! (to-ascii? (calculate-current-interest user)) ...))
        (health-ascii (unwrap! (to-ascii? health) ...)))
        
        (ok {
          principal: principal-ascii,
          interest: interest-ascii,
          health-factor: health-ascii,
          status: (if (< health MIN-HEALTH-FACTOR) "LIQUIDATABLE" "HEALTHY")
        }))
    (ok { principal: "0", interest: "0", health-factor: "0", status: "NO_LOAN" })))
```

**Output Example:**
```clarity
{
  principal: "500000",
  interest: "12500",
  health-factor: "180",
  status: "HEALTHY"
}
```

#### Passkey Challenge Message
**Location:** `contracts/auth/passkey-signer.clar:48-65`

```clarity
(define-read-only (generate-challenge-message 
  (user principal) (action (string-ascii 20)) (amount uint))
  (let (
    (user-str (unwrap! (to-ascii? user) ...))
    (amount-str (unwrap! (to-ascii? amount) ...)))
    
    (ok {
      message: "Authenticate transaction",
      user: user-str,
      action: action,
      amount: amount-str
    })))
```

---

## Feature #5: `secp256r1-verify` - Passkey Support

### What It Does
Verifies secp256r1 (P-256) signatures on-chain, enabling WebAuthn/FIDO2/passkey authentication.

### Why It Matters
- Hardware wallet integration (YubiKey, Ledger)
- Biometric authentication (TouchID, FaceID)
- No seed phrase needed
- Better UX for non-crypto users

### Our Implementation

**Location:** `contracts/auth/passkey-signer.clar:95-110`

```clarity
(define-public (verify-passkey-signature
  (user principal) (message-hash (buff 32)) (signature (buff 64)))
  (let (
    (passkey-data (unwrap! (map-get? user-passkeys { user: user }) ...))
    (public-key (get public-key passkey-data)))
    
    (asserts! (secp256r1-verify message-hash signature public-key)
      err-invalid-signature)
    (ok true)))
```

**How It Works:**
1. User registers their passkey's secp256r1 public key
2. When executing sensitive action, user signs with hardware device
3. Contract verifies signature matches registered public key
4. If valid, action proceeds; if not, transaction reverts

**Multi-Sig Support:**
Users can register multiple passkeys (YubiKey, phone, laptop) and use any for authentication.

---

##Feature #6: Dimension-Specific Tenure Extensions (SIP-034)

### What It Does
Allows Stacks signers to reset individual budget dimensions (read/write/runtime) without resetting others.

### Why It Matters
- Higher transaction throughput
- More efficient block space usage
- Better performance for complex DeFi operations

### Impact on This Project
Our lending pool involves complex operations (interest calculations, liquidations, oracle checks). SIP-034 ensures these operations can execute efficiently even when individual dimensions are maxed out.

---

## Combined Power: Real-World Scenarios

### Scenario 1: Secure Liquidation
1. **Contract Verification:** Liquidator is verified with `contract-hash?`
2. **Asset Protection:** `restrict-assets?` limits what liquidator can move
3. **Time-Based Check:** `stacks-block-time` ensures interest is current
4. **Readable Report:** `to-ascii?` generates liquidation report

### Scenario 2: Passkey-Protected Withdrawal
1. **User Action:** Initiate withdrawal via frontend
2. **Challenge:** Contract generates challenge with `to-ascii?`
3. **Sign:** User signs with passkey (hardware wallet)
4. **Verify:** `secp256r1-verify` validates signature
5. **Execute:** Withdrawal proceeds if valid

### Scenario 3: Governed Upgrade
1. **Proposal:** Create proposal with target contract
2. **Verification:** `contract-hash?` stores new contract hash
3. **Vote:** Community votes over 1 week period
4. **Timelock:** 24-hour delay using `stacks-block-time`
5. **Execute:** Verify hash matches before upgrade
6. **Report:** `to-ascii?` generates execution summary

---

## Best Practices

### Using `contract-hash?`
✅ **Do:** Verify contracts before critical operations
✅ **Do:** Store hashes on-chain for transparency
❌ **Don't:** Trust external contracts without verification

### Using `restrict-assets?`
✅ **Do:** Set restrictive limits on external calls
✅ **Do:** Calculate exact amounts beforehand
❌ **Don't:** Set overly permissive limits "just in case"

### Using `stacks-block-time`
✅ **Do:** Use for time-based logic (interest, expiration)
✅ **Do:** Account for timestamp accuracy (~10 minutes)
❌ **Don't:** Use for sub-minute precision requirements

### Using `to-ascii?`
✅ **Do:** Generate readable reports and logs
✅ **Do:** Handle conversion failures gracefully
❌ **Don't:** Use for complex JSON generation

### Using `secp256r1-verify`
✅ **Do:** Support multiple passkeys per user
✅ **Do:** Allow passkey deactivation
❌ **Don't:** Rely solely on passkeys (support regular auth too)

---

## Conclusion

The Clarity 4 features in this project represent a major step forward for Bitcoin DeFi:
- **Security:** Contract verification and asset restrictions
- **Functionality:** Time-based logic and passkey support
- **UX:** Readable outputs and better tooling

These features enable sophisticated, secure, user-friendly DeFi applications built on Bitcoin.
