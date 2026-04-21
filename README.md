# Polymarket Copy Trading Bot (TypeScript)

[![GitHub](https://img.shields.io/badge/GitHub-AlphaLedger--Labs%2Fpolymarket--copy--trading--bot-181717?logo=github)](https://github.com/AlphaLedger-Labs/polymarket-copy-trading-bot)

A TypeScript bot for monitoring a target Polymarket account and optionally mirroring its trades.

## Overview
This project connects to Polymarket APIs and does two things:

- **Account monitoring:** Polls active positions for a target account and prints a concise status view.
- **Copy trading (optional):** When the target account opens a position, the bot places a corresponding **BUY** order. When the target position disappears, it places a **SELL** order to close.

Order execution uses Polymarket's CLOB via `@polymarket/clob-client`.

## Verified Target & Performance Evidence

This bot has been used to monitor and mirror activity from the following profiles:
- **My profile:** your trading wallet (the address derived from `PRIVATE_KEY` when copy trading is enabled).
- **Target wallet profile:** [Polymarket profile](https://polymarket.com/profile/0xf38190909d9f72d4d3274dc5fa51ad8e42ca2b5d) — set the same address as `TARGET_ADDRESS` (`0xf38190909d9f72d4d3274dc5fa51ad8e42ca2b5d`).
### Live PnL & Portfolio Snapshots

Portfolio value: **$17,680.95** · Available cash: **$12,431.26** · 1D PnL: **+$2,241.76 (+15.0%)**

#### Positions (top of list)
![Polymarket positions - top](img/1.png)

#### Positions (continued)
![Polymarket positions - continued](img/2.png)

### Trading History

Portfolio: **$17,720.36** · 1D PnL: **+$2,307.23 (+15.0%)** · Mirrored activity on *Bitcoin Up or Down* minute markets.

#### Recent buys & claims
![Copy trading history - recent activity](img/3.png)

#### Additional claim/buy flow
![Copy trading history - claim flow](img/4.png)

#### Earlier activity in the session
![Copy trading history - earlier activity](img/5.png)

### Features
- Polling-based monitoring of a single `TARGET_ADDRESS`
- Optional copy-trading controlled by `COPY_TRADING_ENABLED`
- Dry-run mode (`DRY_RUN=true`) to simulate order decisions without posting orders
- Guardrails for position/trade sizing (min/max USD thresholds)

## Prerequisites
- Node.js 18+
- A Polygon wallet private key (only required for copy trading)

## Setup
1. Clone this repository:

   ```bash
   git clone https://github.com/AlphaLedger-Labs/polymarket-copy-trading-bot.git
   cd polymarket-copy-trading-bot
   ```

2. Install dependencies:

   ```bash
   npm install
   ```

3. Create a `.env` file:

   You can base it on `.env.example` if it exists, or use the example below.

4. Start in development mode:

   ```bash
   npm run dev
   ```

## Environment Variables

### Required
- `TARGET_ADDRESS` (string): Address to monitor and (optionally) copy-trade

### Copy trading (optional)
- `COPY_TRADING_ENABLED` (`true`/`false`): Set to `true` to enable copy trading
- `PRIVATE_KEY` (string): Required when `COPY_TRADING_ENABLED=true`

### Tuning
- `POLL_INTERVAL` (milliseconds, default: `2000`)
- `DRY_RUN` (`true`/`false`, default: `false`): If enabled, the bot simulates order execution.
- `POSITION_SIZE_MULTIPLIER` (default: `1.0`)
- `MAX_POSITION_SIZE` (default: `10000`): USD cap used as a guardrail
- `MAX_TRADE_SIZE` (default: `5000`): USD cap used as a guardrail
- `MIN_TRADE_SIZE` (default: `1`): USD minimum to execute a trade
- `SLIPPAGE_TOLERANCE` (default: `1.0`): Reserved/wired but not enforced in the current order parameters
- `DEBUG` (`true`/`false`, optional): Enables additional logging in API client code

### Example `.env`

```env
TARGET_ADDRESS=0x1234567890123456789012345678901234567890

# Enable copy trading (monitoring works even when false)
COPY_TRADING_ENABLED=false

# Required only when COPY_TRADING_ENABLED=true
PRIVATE_KEY=0xYourPrivateKeyHere

# Execution controls
DRY_RUN=true
POLL_INTERVAL=2000

# Sizing controls (USD-based)
POSITION_SIZE_MULTIPLIER=1.0
MAX_POSITION_SIZE=10000
MAX_TRADE_SIZE=5000
MIN_TRADE_SIZE=1

# Wired config (not enforced in current order params)
SLIPPAGE_TOLERANCE=1.0

# Optional
DEBUG=false
```

## Run Modes

### Monitoring only
Set `COPY_TRADING_ENABLED=false` (default behavior) and run:

```bash
npm run dev
```

### Copy-trading dry run
Set `COPY_TRADING_ENABLED=true` and `DRY_RUN=true`, then run:

```bash
npm run dev
```

The bot will show what BUY/SELL orders it *would* post without sending live orders.

### Live copy trading
After validating behavior in dry run, set `DRY_RUN=false` and run:

```bash
npm run dev
```

## Copy-Trading Behavior

When `COPY_TRADING_ENABLED=true`, the bot wraps the account monitor and performs the following loop:

1. Poll the target account’s **active positions** at `POLL_INTERVAL`.
2. Compare the newly fetched open positions against the previous snapshot.
3. For each **newly observed open position**, submit a **CLOB BUY** order.
4. For each **position that disappears** from the target snapshot, submit a **CLOB SELL** order to close.

### Order sizing and guardrails

For BUY/SELL execution, the bot uses the target position’s `quantity` and `price` to compute a USD value and applies your risk controls:

- `tradeQuantity = position.quantity * POSITION_SIZE_MULTIPLIER`
- `tradeValue = tradeQuantity * position.price`
- Execute only if:
  - `tradeValue >= MIN_TRADE_SIZE`
  - `tradeValue <= MAX_TRADE_SIZE`
  - `position.value * POSITION_SIZE_MULTIPLIER <= MAX_POSITION_SIZE`

### Important limitations

- State is kept **in memory only**. If you restart the process, it treats currently open positions as "new" and may re-submit BUY orders (and will not reliably know which positions it previously opened). Run a dry test first and consider external operational controls.
- `SLIPPAGE_TOLERANCE` is wired into configuration, but is **not applied to the current posted order parameters** in this version.
- `enableWebSocket` exists as an option name in the code, but monitoring is currently implemented as **polling**.

## What You’ll See While Running

### Account monitoring output

The account monitor prints a formatted view with a timestamp and a list of up to 10 open positions, including:

- outcome name
- share quantity
- current price
- market question (truncated for readability)

### Copy-trading logging

When new positions are detected, logs show the intended BUY action (or a simulated action in `DRY_RUN`). When positions close on the target, logs show SELL actions as the bot attempts to close corresponding positions on your wallet.

## Project Structure

- `src/index.ts`: Entry point. Loads `dotenv`, validates `TARGET_ADDRESS`, optionally validates `PRIVATE_KEY`, then starts either the monitor-only mode or copy-trading mode.
- `src/api/polymarket-client.ts`: Fetches user positions from Polymarket’s Data API and normalizes market/position fields.
- `src/monitor/account-monitor.ts`: Poll-based monitor that detects changes in open positions and renders formatted status output.
- `src/trading/copy-trading-monitor.ts`: Compares snapshots and coordinates BUY/SELL execution.
- `src/trading/trade-executor.ts`: Creates the wallet + CLOB client and posts orders (or simulates in `DRY_RUN`).

## Available NPM Scripts

- `npm run dev`: Run directly from TypeScript (`ts-node src/index.ts`)
- `npm run watch`: TypeScript compile in watch mode (`tsc --watch`)
- `npm run build`: Build to `dist/` (`tsc`)
- `npm start`: Run the compiled bot (`node dist/index.js`)

## Build & Production
- Build TypeScript:

  ```bash
  npm run build
  ```

- Start production build:

  ```bash
  npm start
  ```

## Safety Notes
- Do **not** commit your `.env` file (it is ignored by `.gitignore`).
- Test with `DRY_RUN=true` before enabling live trading.
- Double-check your sizing guardrails (`MIN_TRADE_SIZE`, `MAX_TRADE_SIZE`, `MAX_POSITION_SIZE`).

## Troubleshooting
- **Invalid `PRIVATE_KEY`:** The bot validates that the key is not missing or a placeholder.
- **API errors / timeouts:** Enable `DEBUG=true` for more logging; verify network access to Polymarket endpoints.
- **No positions detected:** Confirm `TARGET_ADDRESS` has active positions.

## License
MIT (per `package.json`)

## Contact
@Tokidokitrader
