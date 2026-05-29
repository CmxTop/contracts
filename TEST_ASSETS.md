# Test Assets for Testnet Market Initialization

This document describes the test assets used for testnet market initialization in the SO4.market protocol.

## Overview

The protocol uses Stellar Asset Contracts (SACs) for test assets on testnet. These are wrapped versions of real assets that allow testing without using real value.

### Standard Test Assets

| Asset Code | Description | Decimals | Purpose |
|------------|-------------|----------|---------|
| **TWBTC** | Test Wrapped Bitcoin | 7 | Long token for markets |
| **TUSDC** | Test USD Coin | 7 | Short token for markets |

## Asset Properties

All test assets use Stellar's standard 7-decimal precision for consistency with the protocol's math:

- 1 TWBTC = 10,000,000 base units (7 decimals)
- 1 TUSDC = 10,000,000 base units (7 decimals)

## Creating Test Assets

### Quick Start (Both Tokens)

Create both TWBTC and TUSDC in one command:

```bash
make market-tokens NETWORK=testnet SOURCE=alice
```

This will:
1. Generate issuer keypairs for TWBTC and TUSDC
2. Deploy Stellar Asset Contracts for both tokens
3. Trustline and mint 100,000,000,000 (10B) units of each token to the SOURCE account
4. Save contract IDs to `.deployed/tokens-testnet.env`

### Individual Token Creation

Create a single test token:

```bash
# Create TWBTC
make token-bootstrap CODE=TWBTC TO=alice NETWORK=testnet SOURCE=alice

# Create TUSDC
make token-bootstrap CODE=TUSDC TO=alice NETWORK=testnet SOURCE=alice
```

### Step-by-Step Token Creation

For more control over the process:

```bash
# 1. Generate issuer keypair
make token-issuer CODE=TWBTC NETWORK=testnet

# 2. Deploy the Stellar Asset Contract
make token-deploy CODE=TWBTC NETWORK=testnet SOURCE=alice

# 3. Establish trustline
make token-trust CODE=TWBTC TO=alice NETWORK=testnet SOURCE=alice

# 4. Mint tokens
make token-mint CODE=TWBTC TO=alice AMOUNT=1000000000 NETWORK=testnet SOURCE=alice

# 5. Check balance
make token-balance CODE=TWBTC NETWORK=testnet SOURCE=alice
```

## Using Test Assets in Market Initialization

### Full Testnet Bootstrap

The standard testnet deployment workflow uses TWBTC and TUSDC:

```bash
# 1. Create test tokens
make market-tokens NETWORK=testnet SOURCE=alice LONG_CODE=TWBTC SHORT_CODE=TUSDC

# 2. Deploy protocol contracts
make deploy-all NETWORK=testnet SOURCE=alice

# 3. Bootstrap the market (grants roles, creates market, sets config)
make bootstrap NETWORK=testnet SOURCE=alice KEEPER=keeper LONG_CODE=TWBTC SHORT_CODE=TUSDC

# 4. Submit initial oracle prices
bash scripts/submit_prices.sh testnet keeper
```

### Custom Token Configuration

You can use different test assets by overriding the defaults:

```bash
make market-tokens NETWORK=testnet SOURCE=alice LONG_CODE=TCUSTOM SHORT_CODE=TOTHER
make bootstrap NETWORK=testnet SOURCE=alice LONG_CODE=TCUSTOM SHORT_CODE=TOTHER
```

## Token Environment File

After creating tokens, their contract IDs are saved to `.deployed/tokens-testnet.env`:

```bash
TWBTC=CA3D5KRYM6CB7OWQ6TWY7QZ2J7L4KQZJ7L4KQZJ7L4KQZJ7L4KQZJ7L4KQZJ7L4K
TWBTC_ASSET=TWBTC:GD5KRYM6CB7OWQ6TWY7QZ2J7L4KQZJ7L4KQZJ7L4KQZJ7L4KQZJ7L4K
TUSDC=CB7OWQ6TWY7QZ2J7L4KQZJ7L4KQZJ7L4KQZJ7L4KQZJ7L4KQZJ7L4KQZJ7L4KQZJ7L4
TUSDC_ASSET=TUSDC:GD5KRYM6CB7OWQ6TWY7QZ2J7L4KQZJ7L4KQZJ7L4KQZJ7L4KQZJ7L4K
```

The bootstrap script reads this file to get the token contract IDs.

## Minting Additional Tokens

To mint more tokens after initial creation:

```bash
# Mint 100 TWBTC (1,000,000,000 base units)
make token-mint CODE=TWBTC TO=alice AMOUNT=1000000000 NETWORK=testnet SOURCE=alice

# Mint 1000 TUSDC (10,000,000,000 base units)
make token-mint CODE=TUSDC TO=alice AMOUNT=10000000000 NETWORK=testnet SOURCE=alice
```

## Checking Token Balances

```bash
make token-balance CODE=TWBTC NETWORK=testnet SOURCE=alice
make token-balance CODE=TUSDC NETWORK=testnet SOURCE=alice
```

## Market Configuration Variables

When bootstrapping markets, you can configure:

| Variable | Default | Description |
|----------|---------|-------------|
| `LONG_CODE` | `TWBTC` | Ticker code for the long token |
| `SHORT_CODE` | `TUSDC` | Ticker code for the short token |
| `SEED_LONG` | `10000000` | Long token amount for initial liquidity (1 token) |
| `SEED_SHORT` | `10000000` | Short token amount for initial liquidity (1 token) |

Example with custom seed amounts:

```bash
make bootstrap NETWORK=testnet SOURCE=alice \
  LONG_CODE=TWBTC SHORT_CODE=TUSDC \
  SEED_LONG=50000000 SEED_SHORT=50000000
```

## Important Notes

### Testnet Only

These test assets are for testnet only. For mainnet:
- Do not use `token-bootstrap`
- Deploy or look up existing SACs for real Stellar assets
- Configure markets with real asset addresses

### Token Issuers

Each test token has its own issuer keypair:
- TWBTC issuer: `TWBTC-issuer`
- TUSDC issuer: `TUSDC-issuer`

These are generated automatically and stored in your Stellar CLI key store.

### Trustlines

Before minting or transferring tokens, the recipient must have a trustline established. The `token-bootstrap` target handles this automatically.

### Amount Format

All amounts are in base units (7 decimals):
- `10000000` = 1 token
- `100000000` = 10 tokens
- `1000000000` = 100 tokens

## Troubleshooting

### Missing TOKEN_ENV

If you see "Missing .deployed/tokens-testnet.env", run:

```bash
make market-tokens NETWORK=testnet SOURCE=alice
```

### Token Not Found in TOKEN_ENV

Ensure you used the same `CODE` when creating and bootstrapping:

```bash
# Create with CODE=TWBTC
make token-bootstrap CODE=TWBTC NETWORK=testnet SOURCE=alice

# Bootstrap with LONG_CODE=TWBTC
make bootstrap NETWORK=testnet SOURCE=alice LONG_CODE=TWBTC SHORT_CODE=TUSDC
```

### Insufficient Balance

Mint more tokens:

```bash
make token-mint CODE=TWBTC TO=alice AMOUNT=1000000000 NETWORK=testnet SOURCE=alice
```

## References

- Make targets: `contracts/mx/tokens.mk`
- Bootstrap script: `contracts/scripts/bootstrap.sh`
- Deployment guide: `contracts/mx/README.md`
- Main README: `contracts/README.md`
