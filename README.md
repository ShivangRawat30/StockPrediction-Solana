# Stock Prediction — Solana DApp

A decentralized prediction betting application built on the Solana blockchain. Players stake SOL by predicting the price of stocks or crypto assets at a future time. The prediction closest to the real oracle price at expiry wins the pot.

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Smart Contract](#smart-contract)
  - [Accounts & State](#accounts--state)
  - [Instructions](#instructions)
  - [Bet Lifecycle](#bet-lifecycle)
  - [Price Settlement Logic](#price-settlement-logic)
- [Frontend](#frontend)
  - [Component Overview](#component-overview)
  - [State Management](#state-management)
  - [Blockchain Connection](#blockchain-connection)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment & Configuration](#environment--configuration)
- [Supported Assets](#supported-assets)

---

## Overview

Stock Prediction is a peer-to-peer on-chain betting game where two players compete on price predictions. There is no house — all funds go directly to the winner. Settlement is trustless, powered by the **Pyth Network** price oracle.

Key properties:
- **Non-custodial**: funds are held in a program-derived account (PDA), never by a third party
- **Oracle-settled**: winners are determined by Pyth real-time prices, not self-reported data
- **Permissionless**: anyone with a Phantom wallet and SOL can create or enter a bet
- **Devnet**: currently deployed on Solana Devnet for testing

---

## How It Works

```
Player A                                Player B
   |                                        |
   |-- Creates bet -----------------------> |
   |   (stake SOL + price prediction        |
   |    + duration + asset)                 |
   |                                        |
   |                    <------------------ |
   |                    Enters bet          |
   |                    (match stake + own  |
   |                     price prediction)  |
   |                                        |
   |          [Bet Expiry Time Reached]     |
   |                                        |
   |-- Anyone calls ClaimBet ------------> |
   |   (Pyth oracle price fetched on-chain)|
   |   Closest prediction wins 2x stake    |
   |   Draw? Both get stake back           |
```

1. **Player A** creates a bet by choosing an asset, staking an amount of SOL, submitting a price prediction, and setting a duration (e.g. 1 hour).
2. **Player B** sees the open bet and joins it with their own price prediction, matching the exact SOL stake.
3. When the bet duration expires, **anyone** can call ClaimBet. The contract queries the Pyth oracle for the current price on-chain and awards the pot to whichever player's prediction was closest. If it's a tie, both get their stake returned.
4. Players can also **close** expired or unclaimed bets to recover funds under specific conditions.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Frontend (Next.js)                │
│                                                     │
│  ┌──────────┐  ┌───────────┐  ┌─────────────────┐  │
│  │  Header  │  │   Chart   │  │  AvailableBets  │  │
│  │ (wallet) │  │(Portfolio)│  │ (Enter/Claim/   │  │
│  └──────────┘  └───────────┘  │      Close)     │  │
│                                └─────────────────┘  │
│                                                     │
│  ┌────────────────────────────────────────────────┐ │
│  │              GlobalContext (state/global.js)   │ │
│  │  fetchBets · createBet · enterBet              │ │
│  │  claimBet · closeBet · fetchMasterAccount      │ │
│  └────────────────────┬───────────────────────────┘ │
└───────────────────────┼─────────────────────────────┘
                        │ Anchor SDK + IDL
                        │
┌───────────────────────▼─────────────────────────────┐
│              Solana Blockchain (Devnet)              │
│                                                     │
│  ┌────────────────────────────────────────────────┐ │
│  │         Smart Contract (Anchor / Rust)         │ │
│  │  Program ID: 3vY5F41z5htZdEv3AF5vppzamEYwXFWG │ │
│  │                                                │ │
│  │  PDAs:                                         │ │
│  │  • Master Account  [seeds: "master"]           │ │
│  │  • Bet Account     [seeds: "bet", bet_id]      │ │
│  └────────────────────┬───────────────────────────┘ │
└───────────────────────┼─────────────────────────────┘
                        │ CPI (Cross-Program Invocation)
                        │
┌───────────────────────▼─────────────────────────────┐
│              Pyth Network Price Oracle              │
│    (real-time on-chain asset prices + exponent)    │
└─────────────────────────────────────────────────────┘
```

---

## Smart Contract

Located in [Contract/src/](Contract/src/). Built with the **Anchor** framework (v0.28).

### Accounts & State

**Master** (`state.rs`) — singleton PDA tracking global bet ID counter.
```rust
pub struct Master {
    pub last_bet_id: u64,   // incremented on each new bet
}
```

**Bet** (`state.rs`) — PDA holding all data for a single bet.
```rust
pub struct Bet {
    pub id:           u64,
    pub players:      [BetPrediction; 2],  // player A and B
    pub amount:       u64,                 // stake in lamports
    pub pyth_key:     Pubkey,              // Pyth feed account
    pub expiry_time:  i64,                 // Unix timestamp
    pub state:        BetState,
}

pub struct BetPrediction {
    pub player: Pubkey,
    pub price:  i64,   // predicted price (Pyth exponent-adjusted)
}

pub enum BetState {
    Created,      // waiting for Player B
    Started,      // both players in, waiting for expiry
    PlayerAWon,
    PlayerBWon,
    Draw,
}
```

**PDA Seeds:**
| Account | Seeds |
|---------|-------|
| Master  | `[b"master"]` |
| Bet     | `[b"bet", bet_id as 8-byte little-endian]` |

---

### Instructions

Defined in [Contract/src/lib.rs](Contract/src/lib.rs):

| Instruction | Caller | Description |
|-------------|--------|-------------|
| `create_master` | Anyone (once) | Initializes the global Master account |
| `create_bet` | Player A | Creates a new bet, transfers stake to PDA |
| `enter_bet` | Player B | Joins a bet, transfers matching stake to PDA |
| `claim_bet` | Anyone | Settles bet using Pyth oracle, transfers winnings |
| `close_bet` | Varies | Closes and cleans up a bet under allowed conditions |

---

### Bet Lifecycle

```
              create_bet
[Master] ──────────────────► [Bet: Created]
                                    │
                              enter_bet (Player B)
                                    │
                                    ▼
                             [Bet: Started]
                                    │
                    ┌───────────────┼──────────────────┐
                    │               │                  │
              claim_bet       claim_bet           claim_bet
            (A closer)       (B closer)           (equal)
                    │               │                  │
                    ▼               ▼                  ▼
            [PlayerAWon]    [PlayerBWon]           [Draw]
                    │               │                  │
                    └───────────────┴──────────────────┘
                                    │
                              close_bet
                                    │
                            [Account closed,
                             lamports returned
                             to creator]
```

**Timing constraints** (`constants.rs`):
- `MINIMUM_REMAINING_TIME_UNTIL_EXPIRY`: **120 seconds** — Player B cannot enter within 2 minutes of expiry
- `MAXIMUM_CLAIMABLE_PERIOD`: **300 seconds** — ClaimBet must be called within 5 minutes after expiry

---

### Price Settlement Logic

```
1. Fetch Pyth price data for the configured feed account
2. Verify the Pyth account key matches the one stored in the Bet
3. Read: pyth_price (i64) and exponent (i32)
4. multiplier = 10^(-exponent)   // converts raw price to USD
5. adjusted_A = prediction_A × multiplier
6. adjusted_B = prediction_B × multiplier
7. diff_A = |pyth_price − adjusted_A|
8. diff_B = |pyth_price − adjusted_B|
9. if diff_A < diff_B  → Player A wins, receives 2× stake
   if diff_B < diff_A  → Player B wins, receives 2× stake
   if diff_A == diff_B → Draw, each player receives 1× stake back
```

---

## Frontend

A Next.js application in the root directory.

### Component Overview

| File | Purpose |
|------|---------|
| [pages/_app.js](pages/_app.js) | App wrapper — wallet + RPC providers, GlobalState |
| [pages/index.js](pages/index.js) | Main page layout |
| [components/Header.js](components/Header.js) | Navigation bar with Phantom wallet connect button |
| [components/PortfolioChart.js](components/PortfolioChart.js) | Line chart of asset price history (Chart.js) |
| [components/DropDown.js](components/DropDown.js) | Collapsible asset selector menu |
| [components/Asset.js](components/Asset.js) | Asset card showing symbol, price, % change, mini chart |
| [components/AvailableBets.js](components/AvailableBets.js) | Lists open/active bets with Enter / Claim / Close actions |
| [components/CustomModal.js](components/CustomModal.js) | Modal for submitting a price prediction to enter a bet |

### State Management

[state/global.js](state/global.js) exposes a `GlobalContext` React context that wraps the entire app and provides:

```js
{
  fetchMasterAccount,   // read master PDA, get last bet ID
  fetchBets,            // fetch all bet accounts from chain
  createBet,            // send CreateBet transaction
  enterBet,             // send EnterBet transaction
  claimBet,             // send ClaimBet transaction (anyone can call)
  closeBet,             // send CloseBet transaction
}
```

All functions use `react-hot-toast` for real-time user feedback (loading, success, error).

### Blockchain Connection

```
_app.js
  └── ConnectionProvider (RPC endpoint: Solana Devnet via QuikNode)
        └── WalletProvider (Phantom)
              └── WalletModalProvider
                    └── GlobalState
                          └── App
```

**Program initialization** ([utils/program.js](utils/program.js)):
- Reads [utils/idl.json](utils/idl.json) — the auto-generated Anchor IDL
- Creates an `@project-serum/anchor` `Program` instance tied to the connected wallet
- Provides helper functions: `getPrgram()`, `getMasterAccountPk()`, `getBetAccountPk(betId)`

**Transaction flow:**
1. User connects Phantom wallet
2. Frontend derives PDA addresses client-side (seeds are deterministic)
3. Constructs and sends Anchor transaction with the correct account array
4. Waits for confirmation on devnet
5. Refreshes bet list and shows toast notification

---

## Tech Stack

### Smart Contract
| Technology | Version | Role |
|------------|---------|------|
| Rust | stable | Language |
| Anchor | 0.28 | Solana program framework |
| Pyth SDK | — | Price oracle integration |

### Frontend
| Technology | Version | Role |
|------------|---------|------|
| Next.js | 12.1.0 | React framework / SSR |
| React | 17.0.2 | UI library |
| @project-serum/anchor | — | TypeScript bindings for contract |
| @solana/web3.js | — | Solana RPC & transaction building |
| @solana/wallet-adapter | — | Phantom wallet integration |
| Tailwind CSS | 3.0.23 | Styling |
| Chart.js / react-chartjs-2 | 3.7.1 | Price charts |
| react-hot-toast | 2.4.0 | Toast notifications |
| BigNumber.js | 9.1.1 | Large number handling |

---

## Project Structure

```
StockPrediction-Solana/
├── Contract/                   # Rust smart contract (Anchor)
│   ├── Cargo.toml
│   └── src/
│       ├── lib.rs              # Program entry point and 5 instructions
│       ├── state.rs            # Master, Bet, BetPrediction, BetState
│       ├── error.rs            # Custom error codes
│       ├── constants.rs        # Timing constants, PDA seeds
│       └── utils.rs            # Validation logic
│
├── components/                 # React UI components
│   ├── Header.js
│   ├── PortfolioChart.js
│   ├── DropDown.js
│   ├── Asset.js
│   ├── AvailableBets.js
│   └── CustomModal.js
│
├── pages/
│   ├── _app.js                 # Provider setup
│   └── index.js                # Home page
│
├── state/
│   └── global.js               # GlobalContext + all blockchain calls
│
├── hooks/
│   └── useGlobalState.js       # Hook to consume GlobalContext
│
├── utils/
│   ├── program.js              # Anchor program + PDA helpers
│   ├── constants.js            # Program ID, RPC endpoint
│   ├── utils.js                # SOL/lamport conversion helpers
│   └── idl.json                # Anchor IDL (auto-generated)
│
├── data/
│   └── asset.seed.js           # Asset list with symbols and Pyth feed keys
│
├── styles/
│   ├── globals.css
│   └── Home.module.css
│
├── package.json
├── next.config.js
├── tailwind.config.js
└── README.md
```

---

## Getting Started

### Prerequisites

- Node.js >= 16
- A [Phantom Wallet](https://phantom.app/) browser extension
- Devnet SOL (free from [faucet.solana.com](https://faucet.solana.com))

### Install & Run

```bash
# Install dependencies
npm install

# Start development server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

### Build for Production

```bash
npm run build
npm start
```

---

## Environment & Configuration

All network config lives in [utils/constants.js](utils/constants.js):

```js
// Solana Devnet RPC (QuikNode)
export const RPC_ENDPOINT = "https://snowy-twilight-lambo.solana-devnet.discover.quiknode.pro/";

// Deployed program ID on Devnet
export const PROGRAM_ID = new PublicKey("3vY5F41z5htZdEv3AF5vppzamEYwXFWGVatWRhZuRbBC");
```

Make sure your Phantom wallet is set to **Devnet** before connecting.

---

## Supported Assets

Assets are seeded in [data/asset.seed.js](data/asset.seed.js). Each asset maps to a **Pyth price feed account** used for on-chain settlement:

| Symbol | Type |
|--------|------|
| BTC | Crypto |
| SOL | Crypto |
| AMC | Stock |
| AMZN | Stock |
| GOOG | Stock |
| BSE | Index |
| NSE | Index |

---

## Key Constraints & Rules

| Rule | Value |
|------|-------|
| Minimum time before expiry for Player B to enter | 120 seconds |
| Maximum window to call ClaimBet after expiry | 300 seconds (5 min) |
| Both players must stake identical SOL amounts | Enforced on-chain |
| Pyth feed account must match the one set at bet creation | Enforced on-chain |
| Winner takes | 2× the staked amount |
| Draw result | Each player receives their original stake |

---

## License

MIT
