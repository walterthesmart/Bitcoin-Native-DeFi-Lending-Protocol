# Bitcoin-Native DeFi Lending Protocol (Clarity 4)

A sophisticated decentralized lending and borrowing protocol built with **Clarity 4**, showcasing all major new features introduced in the Stacks SIP-033 upgrade.

## 🚀 Clarity 4 Features Showcased

This project demonstrates **all 6 major Clarity 4 features** in production-ready implementations:

### 1. **`contract-hash?`** - On-chain Contract Verification
**Location:** `contracts/lending-pool.clar`

The lending pool verifies liquidator contracts before allowing them to execute liquidations:
```clarity
;; Register and verify liquidator contracts
(define-public (register-verified-liquidator (liquidator principal))
  (match (contract-hash? liquidator)
    hash-value (var-set verified-liquidator-hash (some hash-value))
    err-contract-verification-failed))
```

**Why it matters:** Ensures only audited, verified liquidation logic can interact with user funds.

### 2. **`restrict-assets?`** - Contract-Level Post-Conditions
**Location:** `contracts/lending-pool.clar`

Protects pool funds when calling external liquidator contracts:
```clarity
;; Restrict assets that external contract can move
(restrict-assets? 
  (contract-of liquidator)
  (list { asset: 'STX, amount: liquidation-amount, sender: (as-contract tx-sender) }))
```

**Why it matters:** Automatically rolls back transactions if external contracts try to move more assets than allowed.

### 3. **`stacks-block-time`** - Block Timestamp Access
**Locations:** `contracts/lending-pool.clar`, `contracts/oracle/price-oracle.clar`, `contracts/governance/protocol-governance.clar`

Enables time-based logic for interest accrual, price freshness, and governance timelocks:
```clarity
;; Calculate interest based on elapsed time
(let ((time-elapsed (- stacks-block-time (get last-interest-update loan-data))))
  (calculate-interest principal-amt time-elapsed))
```

**Why it matters:** Essential for DeFi features like yield calculations, lockups, and expiration conditions.

### 4. **`to-ascii?`** - Convert Values to ASCII Strings
**Locations:** All contracts

Generate human-readable status messages and reports:
```clarity
;; Generate readable loan status
(define-read-only (get-loan-status-ascii (user principal))
  (let (
    (principal-ascii (to-ascii? principal-amount))
    (health-ascii (to-ascii? health-factor)))
    (ok { principal: principal-ascii, health: health-ascii, status: "HEALTHY" })))
```

**Why it matters:** Improves cross-chain communication and generates readable on-chain messages.

### 5. **`secp256r1-verify`** - Passkey Integration
**Location:** `contracts/auth/passkey-signer.clar`

Enables WebAuthn/FIDO2 authentication for hardware wallets:
```clarity
;; Verify passkey signature on-chain
(define-public (verify-passkey-signature (user principal) (message-hash (buff 32)) (signature (buff 64)))
  (asserts! (secp256r1-verify message-hash signature public-key) err-invalid-signature))
```

**Why it matters:** Opens the door to hardware-secured wallets and biometric transaction signing.

### 6. **Dimension-Specific Tenure Extensions** (SIP-034)
Allows high-throughput operations without resetting all budget dimensions.

**Why it matters:** Enables more transactions per block for DeFi applications.

## 📋 Project Structure

```
Bitcoin-Native DeFi Lending Protocol/
├── contracts/
│   ├── lending-pool.clar                 # Main lending pool (160 lines)
│   ├── traits/
│   │   ├── lending-pool-trait.clar       # Standard lending interface
│   │   └── liquidator-trait.clar         # Standard liquidator interface
│   ├── liquidators/
│   │   └── simple-liquidator.clar        # Verified liquidator implementation
│   ├── oracle/
│   │   └── price-oracle.clar             # Price feeds with timestamps
│   ├── auth/
│   │   └── passkey-signer.clar           # Passkey authentication
│   ├── governance/
│   │   └── protocol-governance.clar      # Time-locked governance
│   └── utils/
│       └── math-helpers.clar             # Math utilities
├── tests/
│   └── lending-pool_test.clar            # Comprehensive test suite
├── frontend/
│   ├── index.html                        # Modern lending UI
│   ├── js/contract-interactions.js       # Contract integration
│   └── css/styles.css                    # Premium styling
├── docs/
│   ├── ARCHITECTURE.md                   # Architecture overview
│   └── CLARITY4_FEATURES.md              # Feature deep-dive
├── Clarinet.toml                         # Project configuration
└── settings/Devnet.toml                  # Network settings
```

## 🔧 Core Features

### Lending & Borrowing
- **Deposit STX** to earn yield
- **Borrow** against collateral (150% collateralization ratio)
- **Time-based interest** accrual using `stacks-block-time`
- **Health factor** monitoring for loan safety

### Liquidations
- **Verified liquidators** using `contract-hash?`
- **Asset protection** with `restrict-assets?`
- **10% liquidation bonus** for liquidators

### Price Oracle
- **Multi-asset price feeds** (sBTC, STX, USD)
- **Freshness checks** using `stacks-block-time`
- **Human-readable status** messages with `to-ascii?`

### Passkey Authentication
- **Hardware wallet support** via `secp256r1-verify`
- **Multi-signature** capabilities
- **Biometric authentication** ready

### Governance
- **Time-locked proposals** (24-hour delay)
- **Contract verification** before upgrades
- **Quorum-based voting** (20% threshold)

## 🚀 Getting Started

### Prerequisites
- [Clarinet](https://github.com/hirosystems/clarinet) installed
- Node.js (for frontend)
- A modern web browser

### Installation

1. **Clone or navigate to the project:**
   ```bash
   cd "C:\Users\HomePC\Desktop\Blockchain\Stacks\Bitcoin-Native DeFi Lending Protocol"
   ```

2. **Check contracts compile:**
   ```bash
   clarinet check
   ```

3. **Run tests:**
   ```bash
   clarinet test
   ```

4. **Start Clarinet console for interactive testing:**
   ```bash
   clarinet console
   ```

### Testing in Console

```clarity
;; Deploy contracts
(contract-call? .lending-pool deposit u1000000)

;; Add collateral
(contract-call? .lending-pool add-collateral u2000000 "STX")

;; Borrow against collateral
(contract-call? .lending-pool borrow u500000)

;; Check loan status
(contract-call? .lending-pool get-loan-status-ascii tx-sender)

;; Update price oracle
(contract-call? .price-oracle update-price "sBTC" u51000000000 "manual-update")

;; Get price with freshness check
(contract-call? .price-oracle get-fresh-price "sBTC")

;; Register a passkey (example public key)
(contract-call? .passkey-signer register-passkey 
  0x02a1234... ;; 33-byte compressed secp256r1 public key
  "YubiKey 5")
```

## 📖 Smart Contract API

### Lending Pool

- `deposit(amount)` - Deposit STX into the pool
- `withdraw(amount)` - Withdraw STX from the pool
- `add-collateral(amount, asset)` - Add collateral to enable borrowing
- `borrow(amount)` - Borrow against collateral
- `repay(amount)` - Repay loan with interest
- `liquidate(borrower, liquidator)` - Liquidate undercollateralized position
- `get-health-factor(user)` - Get user's loan health (150+ = healthy)
- `get-loan-status-ascii(user)` - Get human-readable loan status

### Price Oracle

- `update-price(asset, price, source)` - Update asset price
- `get-price(asset)` - Get current price
- `get-fresh-price(asset)` - Get price with staleness check
- `get-price-status(asset)` - Get human-readable price status
- `is-price-fresh(asset)` - Check if price is fresh

### Passkey Authentication

- `register-passkey(public-key, device-name)` - Register a passkey
- `verify-passkey-signature(user, message-hash, signature)` - Verify signature
- `execute-with-passkey(...)` - Execute protected action with passkey
- `deactivate-passkey(key-index)` - Disable a passkey
- `get-auth-summary(user)` - Get authentication summary

### Governance

- `create-proposal(title, description, target-contract)` - Create proposal
- `vote(proposal-id, vote-for)` - Vote on proposal
- `execute-proposal(proposal-id)` - Execute after timelock
- `get-proposal-status-ascii(proposal-id)` - Get readable status
- `can-execute(proposal-id)` - Check if ready to execute

## 🎨 Frontend

Open `frontend/index.html` in a browser to access the lending interface:

- **Connect Wallet** (including passkey support)
- **Deposit & Borrow** interface
- **Real-time health factor** monitoring
- **Interest accrual** visualization
- **Premium dark mode** design

## 📚 Documentation

- [ARCHITECTURE.md](docs/ARCHITECTURE.md) - System architecture and design
- [CLARITY4_FEATURES.md](docs/CLARITY4_FEATURES.md) - Deep dive into Clarity 4 features

## 🔒 Security Features

1. **Contract Verification** - Only verified liquidators can execute
2. **Asset Restrictions** - Post-conditions protect user funds
3. **Time Locks** - Governance changes have 24-hour delay
4. **Health Monitoring** - Automatic liquidation below 120%
5. **Passkey Auth** - Hardware wallet security

## 🏗️ Architecture Highlights

- **Modular design** with trait-based interfaces
- **Separation of concerns** (lending, oracle, auth, governance)
- **Extensible liquidator** system
- **Time-based calculations** using block timestamps
- **Human-readable outputs** for better UX

## 📈 Use Cases

- **Bitcoin Holders** - Earn yield on sBTC deposits
- **Borrowers** - Access liquidity without selling Bitcoin
- **Liquidators** - Earn 10% bonus on liquidations
- **Developers** - Learn Clarity 4 best practices

## 🤝 Contributing

This project serves as a reference implementation for Clarity 4 features. Feel free to:
- Study the code patterns
- Extend functionality
- Build new features on top
- Submit improvements

## 📄 License

MIT License - Built for educational and development purposes

## 🔗 Resources

- [Clarity 4 Documentation](https://docs.stacks.co/whats-new/clarity-4-is-now-live)
- [SIP-033 Specification](https://github.com/stacksgov/sips/pull/218)
- [SIP-034 Specification](https://github.com/314159265359879/sips/blob/9b45bf07b6d284c40ea3454b4b1bfcaeb0438683/sips/sip-034/sip-034.md)
- [Clarity Reference](https://docs.stacks.co/reference/clarity/)

---

**Built with ❤️ using Clarity 4** - Showcasing the future of Bitcoin DeFi
