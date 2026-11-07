# BTC Trading Agent Eval — micro1 Eval Hackathon

This project is my submission for the **micro1 Eval Hackathon**. It explores a practical limitation of current frontier AI models: an AI can generate plausible trading code, but it cannot reliably create a profitable, production-ready trading strategy from a vague prompt alone.

The project is a low-latency BTC trading signal engine written in Rust, with a MetaTrader 5 Expert Advisor (EA) bridge. It uses live BTC market data, short-horizon price/volume features, trade filtering, simulated trade tracking, and execution through the Headway Trading platform.

## The limitation being evaluated

Trading is not solved by producing code that compiles or by achieving a high win rate on a small sample. A usable trading agent must also handle:

- noisy and non-stationary market behavior;
- order-book and tick-data interpretation;
- timing, spread, slippage, and execution differences;
- broker symbol rules, minimum stop distances, and fill prices;
- risk sizing and protection after an order is filled; and
- commissions and other costs that can turn a grossly profitable strategy into a net-loss strategy.

AI models can write a convincing first version of a trading bot, but they cannot independently create and prove that the bot will remain profitable in live markets. They usually produce code from patterns in their training data and the instructions they receive. They do not know which market conditions the strategy was designed for, whether the backtest is overfitted, whether the broker will fill orders as expected, or whether fees will remove the trading edge. I had to provide the market knowledge, identify the execution problems, and repeatedly test and adjust the strategy.

The specific limitation I am evaluating is this: AI cannot make a consistently profitable trading algorithm or trading bot by itself. It can generate an indicator, strategy, backtest, or exchange API integration, but it cannot independently guarantee positive net profit after changing volatility, latency, rejected orders, and drawdown. A bot that looks successful in a backtest or has a high win rate can still lose money when it is connected to a real broker. Building something that works under those conditions requires expert decisions and validation outside the model’s generated code.

The same limitation applies to other high-risk automated strategies such as options trading, order-book trading, flash-loan arbitrage, and token-sniping: apparent strategy success is not the same as reliable, risk-adjusted profit after execution costs.

## What this project demonstrates

The engine can generate BTC signals and connect them to MT5. In my testing, the strategy produced a gross trade success rate of approximately 80% under the conditions I tested. The broker I used did not add slippage or other trading fees; the main cost was commission. That commission reduced the profit because the strategy makes frequent trades and the profit per trade is small. If the broker also added significant slippage or additional fees, the algorithm would not remain successful under those conditions.

The important failure mode is transaction cost. Because this is a high-frequency strategy, the platform’s commission, spread, and slippage can consume the small per-trade edge. The system can therefore be directionally successful while still being unprofitable after fees. This is the central practical result of the project.

## Architecture

```text
Live BTC trade stream
          |
          v
Rust signal engine (src/faster.rs)
  - tick and volume processing
  - velocity / momentum / trend filters
  - simulated trade and performance tracking
  - HTTP signal endpoint: POST /signal on port 8081
          |
          v
MetaTrader 5 Expert Advisor (TradeEngine.mq5)
  - sends MT5 ticks to Rust
  - receives LONG / SHORT / WATCH responses
  - places market orders
  - attaches TP/SL using the actual fill price when required
```

## Requirements

- Rust 1.75 or newer with Cargo
- MetaTrader 5 connected to the Headway Trading platform, if EA execution is desired
- Internet access for the market-data connection
- A demo account is strongly recommended

This software is experimental research code. It is not financial advice and must not be used with money you cannot afford to lose.

## Build and run the Rust engine

Open PowerShell or a terminal in this project directory. First install and build the Rust packages in release mode:

```bash
cargo build --release
```

After the build finishes successfully, start the trading engine:

```bash
cargo run --bin faster
```

The `faster` binary starts the terminal dashboard and the local MT5 bridge. Keep this process running while MT5 is connected to it. The EA expects:

```text
http://127.0.0.1:8081/signal
```

`127.0.0.1` means localhost: the Rust server and MT5 must be running on the same computer. Do not replace it with a website URL.

## Complete MetaTrader 5 setup

### 1. Open a demo account and find the BTC symbol

Use a demo account first. The symbol used for this project on Headway Trading is `BTC`. In MT5:

1. Open **View → Market Watch** (or press **Ctrl+M**).
2. Right-click inside Market Watch and choose **Show All**.
3. Search for the `BTC` symbol on Headway Trading.
4. Right-click the symbol and choose **Chart Window**.

Use `BTC` in the EA’s `InpSymbol` input, or leave `InpSymbol` empty so the EA uses the chart symbol.

### 2. Copy the EA into MT5

1. Open MT5.
2. Select **File → Open Data Folder**.
3. Open `MQL5`, then open `Experts`.
4. Copy [`TradeEngine.mq5`](TradeEngine.mq5) into the `Experts` folder.
5. Return to MT5 and press **F4** to open MetaEditor.
6. In MetaEditor, open `TradeEngine.mq5` under **Experts**.
7. Press **F7** or select **Build → Compile**.
8. Confirm the Toolbox shows `0 error(s)`. Warnings should be reviewed before use.
9. Return to MT5, open the Navigator with **Ctrl+N**, and right-click **Expert Advisors → Refresh**.

### 3. Allow the localhost connection

The EA uses MT5 `WebRequest()` to send tick data to the Rust HTTP server. MT5 blocks WebRequest URLs until they are explicitly allowed.

1. In MT5, select **Tools → Options** (or press **Ctrl+O**).
2. Open the **Expert Advisors** tab.
3. Check **Allow WebRequest for listed URL**.
4. Click **Add**.
5. Enter exactly:

   ```text
   http://127.0.0.1:8081
   ```

6. Click **OK** to close the URL dialog.
7. Click **OK** again to save MT5 options.

The URL must match the Rust server address. The EA’s default endpoint is `http://127.0.0.1:8081/signal`; adding the host and port `http://127.0.0.1:8081` grants access to that endpoint.

### 4. Start Rust before attaching the EA

In the project directory, run:

```bash
cargo run --bin faster
```

Wait until the Rust dashboard is running and the bridge is listening. Leave the terminal open. Closing it stops the localhost server.

### 5. Attach the Expert Advisor to the chart

1. Open the `BTC` chart identified earlier on Headway Trading.
2. In Navigator, expand **Expert Advisors**.
3. Drag `TradeEngine` onto the `BTC` chart, or double-click it.
4. In the EA settings window, open the **Common** tab.
5. Check **Allow Algo Trading**.
6. Open the **Inputs** tab and review:

   | Input | Meaning | Recommended first test |
   |---|---|---|
   | `InpLotSize` | Trade size | Use the smallest demo size, such as `0.01` |
   | `InpSymbol` | Symbol to trade | Leave empty for the chart symbol |
   | `InpInterval` | Polling interval in milliseconds | `50` |
   | `InpServerURL` | Rust signal endpoint | `http://127.0.0.1:8081/signal` |
   | `InpMinStopDist` | Minimum TP/SL distance in price units | Keep the default until broker rules are checked |

7. Click **OK** to attach the EA.

### 6. Enable Algo Trading

Algo Trading must be enabled both globally and for the EA:

1. On the MT5 top toolbar, click **Algo Trading** so it is enabled/green.
2. If the button is disabled, open **Tools → Options → Expert Advisors** and verify that automated trading is allowed.
3. Open the EA properties again with **F7** while the chart is active, select **Common**, and confirm **Allow Algo Trading** is checked.
4. A smiling EA icon should appear in the chart’s upper-right corner. A disabled icon means the EA is not allowed to trade.

Do not enable live trading until the connection and signals have been tested on a demo account.

### 7. Verify that everything is connected

Use all three checks:

1. In the **Experts** and **Journal** tabs at the bottom of MT5, look for `Rust Bridge 1.80 Started` and the selected symbol.
2. Confirm there are no `URL NOT ALLOWED` or `WebRequest Failed` messages.
3. Confirm the Rust dashboard is receiving ticks and the EA is polling the `/signal` endpoint.

The EA sends JSON resembling:

```json
{"p":95000.0,"v":1.0,"b":true}
```

The Rust bridge responds with a signal such as:

```json
{"s":"W","p":95000.0,"tp":10.0,"sl":10.0,"l":0.010}
```

`W` means watch, `L` means long, and `S` means short. The EA ignores repeated signals and processes new trade signals through its order and protection logic.

### 8. Understand the order-protection behavior

The EA checks the broker’s minimum stop distance. If necessary, it increases TP/SL distances to satisfy the broker’s rules.

For brokers that permit protection with the initial order, the EA sends the order with TP and SL. For market-execution symbols or when that order is rejected, it:

1. opens the market order without TP/SL;
2. reads the actual fill price;
3. calculates TP and SL from that actual fill; and
4. modifies the position to attach protection.

Watch the Experts log for `PROTECTION FAILED` messages. A failed protection modification should be treated as a serious execution problem.

## Troubleshooting

### `URL NOT ALLOWED` / MT5 error 4014

Open **Tools → Options → Expert Advisors** and add `http://127.0.0.1:8081` to the allowed WebRequest list. Restart or reattach the EA after saving.

### WebRequest failed / connection refused

Confirm that `cargo run --bin faster` is still running, the URL is exactly `http://127.0.0.1:8081/signal`, and no firewall is blocking the local process.

### No signals or no trades

Confirm Algo Trading is enabled, **Allow Algo Trading** is checked in the EA properties, the correct BTC symbol is selected, and the Experts log contains no broker rejection. A `W` response means the strategy is watching and has not generated a trade signal.

### Invalid stops or TP/SL rejected

The broker may require a larger minimum stop distance. Check the symbol’s contract specifications and increase `InpMinStopDist` only after verifying the price units used by that broker.

### Wrong symbol

Leave `InpSymbol` empty when attaching the EA directly to the intended BTC chart. Otherwise enter the exact Market Watch symbol, including any broker suffix.

## Reproducibility and evaluation notes

1. Build and start the Rust engine.
2. Attach the EA to a demo `BTC` chart on Headway Trading.
3. Observe the dashboard, MT5 Experts log, generated signals, and simulated trade summary.
4. Record gross results and then subtract commission, spread, and slippage.

A strong evaluation should report both gross and net results over a fixed period and include out-of-sample data. A win rate alone is insufficient because it does not measure expectancy, drawdown, execution quality, or fees.

## Walkthrough video

Video walkthrough: **[Watch the complete demonstration here](https://drive.google.com/file/d/15lOz-90yKRoY9ZaMkaNmi1waDSQJWJqN/view?usp=drive_link)**

It demonstrates the Rust engine running, localhost WebRequest permission, EA installation, chart attachment, Algo Trading activation, signal/connection logs, and how the trading process works from generating a signal to executing the trade in MT5.

## Repository layout

| File | Purpose |
|---|---|
| `src/faster.rs` | Main Rust trading engine, dashboard, simulator, and MT5 HTTP bridge |
| `TradeEngine.mq5` | MetaTrader 5 EA for tick forwarding and execution |
| `Cargo.toml` | Rust package and dependency configuration |
| `Cargo.lock` | Locked dependency versions |
| `src/` | Additional strategy and simulation implementations |

## Disclaimer

Automated trading is high risk. Historical or simulated performance does not predict future results. I recommend testing this project on a demo account, verifying the broker behavior, and accounting for all trading costs before considering real deployment.
