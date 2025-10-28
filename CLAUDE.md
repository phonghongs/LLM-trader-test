# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DeepSeek Paper Trading Bot - A cryptocurrency paper trading system that leverages DeepSeek AI via OpenRouter API for automated trading decisions on Binance Futures. The bot operates on 3-minute intervals, analyzing ETH, SOL, XRP, BTC, DOGE, and BNB using technical indicators (EMA, RSI, MACD) and executes trades based on AI-generated JSON responses.

**Live Demo**: [llmtest.coininspector.pro](https://llmtest.coininspector.pro/)

## Architecture

### Core Components

**bot.py** (~1700 lines)
- Main trading loop orchestrating market data collection, AI decision-making, and trade execution
- Stateless between restarts: loads/saves state from `portfolio_state.json` and persists `iteration_counter`
- Global state: `balance`, `positions` (Dict[coin, position_info]), `trade_history`, `equity_history`
- 3-minute polling cycle: fetch candles → compute indicators → format prompt → call DeepSeek → execute decisions
- Telegram notification integration sends iteration summaries via `record_iteration_message()` and `send_telegram_message()`

**dashboard.py** (~500 lines)
- Streamlit dashboard for real-time monitoring and historical analysis
- Displays portfolio metrics, equity curves with BTC buy-and-hold benchmark, trade history, and AI decision logs
- Reads from CSV files in `data/`: `portfolio_state.csv`, `trade_history.csv`, `ai_decisions.csv`, `ai_messages.csv`
- Calculates Sharpe ratio from closed trades and Sortino ratio from equity curve

### Data Flow

```
Binance API → fetch_market_data() → calculate_indicators() → format_prompt_for_deepseek()
    ↓
DeepSeek API (via OpenRouter) → call_deepseek_api() → parse JSON decisions
    ↓
execute_entry() / execute_close() → update positions/balance → log_trade() → CSV persistence
    ↓
log_portfolio_state() → STATE_CSV, STATE_JSON → dashboard visualization
```

### Key Data Structures

**Position Schema** (bot.py:~400-450):
```python
positions[coin] = {
    "side": "long" | "short",
    "quantity": float,
    "entry_price": float,
    "profit_target": float,
    "stop_loss": float,
    "leverage": int,
    "confidence": float,
    "entry_fee": float,
    "invalidation_condition": str,
    "justification": str
}
```

**Decision Contract** (bot.py:931-1043):
DeepSeek responds with JSON per asset: `{"ETH": {"signal": "entry|close|hold", ...}}`. The `signal` field determines action:
- `entry`: Opens new position with specified leverage, stops, and targets
- `close`: Closes existing position and calculates realized PnL
- `hold`: No action; logs justification to `ai_decisions.csv`

### Trading System Prompt

The bot enforces risk-first trading rules (bot.py:59-104):
- Never risk >1-2% capital per trade
- Mandatory stop-loss orders on every position
- Trend-following bias with confirmation
- Position sizing scaled by leverage and risk tolerance
- Patience over overtrading

## Development Commands

### Environment Setup
```bash
# Create .env from template (required for API access)
cp .env.example .env
# Edit .env with real credentials:
# BN_API_KEY, BN_SECRET (Binance), OPENROUTER_API_KEY (DeepSeek)
# Optional: TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID for push notifications
```

### Docker Workflow
```bash
# Build image (Python 3.13.3-slim with gcc/g++ for numpy/pandas)
docker build -t tradebot .

# Prepare local data directory
mkdir -p ./data

# Run bot (mounts data/ for persistence between runs)
docker run --rm -it \
  --env-file .env \
  -v "$(pwd)/data:/app/data" \
  tradebot

# Run dashboard on port 8501
docker run --rm -it \
  --env-file .env \
  -v "$(pwd)/data:/app/data" \
  -p 8501:8501 \
  tradebot \
  streamlit run dashboard.py
```

### Local Development
```bash
# Install dependencies (requires Python 3.13+)
pip install -r requirements.txt

# Run bot directly (creates data/ in project root if missing)
python bot.py

# Run dashboard locally
streamlit run dashboard.py
# Access at http://localhost:8501

# Override data directory location
export TRADEBOT_DATA_DIR=/custom/path
python bot.py
```

### Data Persistence

The bot writes to `data/` (configurable via `TRADEBOT_DATA_DIR`):
- `portfolio_state.csv`: Timestamped equity snapshots for performance tracking
- `portfolio_state.json`: Latest balance, positions, iteration counter (reloaded on restart)
- `trade_history.csv`: All entry/close executions with PnL and fees
- `ai_decisions.csv`: Every AI decision (including holds) with justifications
- `ai_messages.csv`: Raw request/response logs for DeepSeek API debugging
- `equity_history.json`: Incremental equity snapshots for Sortino ratio calculation

**State Recovery**: On startup, `load_state()` (bot.py:385-449) restores `balance`, `positions`, and `iteration_counter` from `portfolio_state.json`. The bot will resume where it left off, maintaining continuity across Docker restarts.

## Technical Details

### API Integration

**Binance Client** (bot.py:167-202):
- Initialized with `python-binance` library using testnet=False (live market data only)
- Fetches 3-minute candles via `client.get_klines()` for indicator computation
- Graceful degradation: logs warning and retries on connection failures
- Never places live orders (paper trading only simulates fills at current market price)

**DeepSeek API** (bot.py:931-1043):
- Uses OpenRouter endpoint: `https://openrouter.ai/api/v1/chat/completions`
- Model: `deepseek/deepseek-r1` with `response_format={"type": "json_object"}` to enforce JSON
- Error handling: Retries on HTTP 408/429/503/504 with exponential backoff (up to 5 attempts)
- Logs all requests/responses to `ai_messages.csv` for post-mortem debugging
- Telegram notifications on critical failures via `notify_error()` helper (bot.py:366-381)

### Indicator Calculations

All indicators computed from 3-minute candles (bot.py:498-568):
- **EMA 20**: Exponential moving average (trend direction)
- **RSI 14**: Relative Strength Index (overbought/oversold, 0-100 scale)
- **MACD 12/26/9**: Trend momentum with histogram divergence
- **ATR**: Average True Range for volatility estimation (used in risk sizing)

Indicators are formatted into the DeepSeek prompt with 2-4 decimal precision (bot.py:631-761).

### Fee Structure

**Fee Rates** (bot.py:119-120):
- Maker: 0.0% (limit orders, not used in paper trading)
- Taker: 0.0275% (market orders, applied on entry and exit)

**Fee Calculation** (bot.py:1076-1087, 1167-1173):
- Entry fee: `notional_value * TAKER_FEE_RATE` (deducted from balance on position open)
- Exit fee: `exit_value * TAKER_FEE_RATE` (deducted from realized PnL on close)
- Fees compound over round-trip trades and impact net returns

### Performance Metrics

**Sortino Ratio** (bot.py:1088-1116, dashboard.py:resolve_risk_free_rate):
- Calculated from `equity_history` snapshots taken every iteration
- Penalizes downside volatility only (negative returns below risk-free rate)
- Configurable risk-free rate via `.env`: `SORTINO_RISK_FREE_RATE` or `RISK_FREE_RATE` (default: 0.0)
- Formula: `(annualized_return - risk_free_rate) / downside_std`

**Sharpe Ratio** (dashboard.py only):
- Derived from closed trade returns in `trade_history.csv`
- Standard deviation includes both upside and downside volatility
- Less informative with small sample sizes (Sortino preferred for early-stage evaluation)

### Risk Management

**Position Sizing** (bot.py:1167-1173):
- Calculates notional exposure: `quantity * entry_price * leverage`
- Deducts entry fee from available balance before opening position
- Enforces capital preservation: no position exceeds available balance after fees

**Stop-Loss Enforcement** (bot.py:1196-1218):
- Passive monitoring: checks if current price breached stop-loss or hit profit target
- Auto-closes position and logs `reason: "stop-loss"` or `reason: "profit target"`
- Exit fees deducted from realized PnL before balance update

**Portfolio Constraints**:
- One position per coin maximum (new entry rejected if coin already has open position)
- Available balance tracked separately from total equity (margin locked in open positions)

## Development Practices

### Code Style
- Type hints throughout (Python 3.13+ required for `from __future__ import annotations`)
- Logging via `logging.info/warning/error` for operational events
- Colorama for terminal output with ANSI escape code stripping for Telegram
- No external linting config; follow existing patterns for consistency

### Error Handling Philosophy
- **Network Failures**: Binance/DeepSeek API errors trigger warnings, not crashes; bot sleeps 60s and retries
- **State Corruption**: Missing CSV files auto-initialize with headers on first run
- **Invalid AI Responses**: Malformed JSON or missing fields logged as errors; iteration skipped with warning
- **Data Directory**: Created automatically on startup if missing; never assumes pre-existing files

### Debugging Workflow
1. **Check `ai_messages.csv`**: Contains raw DeepSeek prompts and responses with timestamps
2. **Review `ai_decisions.csv`**: Every signal (entry/close/hold) with AI justifications
3. **Examine `trade_history.csv`**: Execution prices, PnL, fees, and stop/target compliance
4. **Monitor Telegram**: Real-time iteration summaries sent if `TELEGRAM_BOT_TOKEN` configured
5. **Inspect `portfolio_state.json`**: Latest balance, positions, and iteration counter for state recovery

### Testing Strategy
- **No automated tests included**: Bot designed for live deployment with manual validation
- **Paper Trading as QA**: All trades simulated; no risk to real capital during development
- **Sample Data**: Repository includes example CSVs in `data/` for immediate dashboard testing
- **Manual verification**: Compare DeepSeek decisions against prompt context in `ai_messages.csv`

## Key Behavioral Notes

### Stateful Iteration Counter
The `iteration_counter` persists across bot restarts (bot.py:387-399). It's incremented every loop iteration and saved to `portfolio_state.json`. This ensures the bot can track total runtime iterations even after Docker container restarts. The counter is displayed in logs and Telegram notifications.

### Telegram Integration
If `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` are set in `.env`, the bot sends a message after each iteration containing:
- Positions opened/closed with PnL
- Portfolio summary (balance, equity, return %, Sortino ratio)
- Stop-loss/profit target hits
All ANSI color codes are stripped from console output before sending (bot.py:334-336).

### Dashboard Caching
Streamlit dashboard uses `@st.cache_data(ttl=15)` for CSV loads (dashboard.py:86-145). This reduces file I/O during rapid refreshes but may delay new trade visibility by up to 15 seconds. The TTL is intentional for production deployment performance.

### DeepSeek Response Parsing
The bot expects strict JSON from DeepSeek (bot.py:970-1043). If JSON parsing fails or required fields are missing:
1. Logs error with raw response to `ai_messages.csv`
2. Skips trade execution for that iteration
3. Continues main loop without crashing
This ensures resilient operation despite occasional AI hallucinations.

### No Authentication
The bot and dashboard have no login system. When deploying publicly (e.g., llmtest.coininspector.pro), secure environment variables via `.env` and restrict network access via firewall/reverse proxy. The dashboard is read-only and displays historical data only.

## Maintenance Considerations

### CSV Schema Migrations
If column names change in `STATE_COLUMNS` or trade logging functions (bot.py:221-230, 289-307), existing CSV files must be manually migrated. The bot never auto-rewrites headers; mismatched schemas will cause pandas errors in the dashboard.

### Binance API Rate Limits
The bot fetches 3-minute candles every 3 minutes (6 symbols = 6 requests per cycle). At default `CHECK_INTERVAL=180`, this stays well below Binance rate limits (1200 requests/minute). Avoid reducing `CHECK_INTERVAL` below 60 seconds without implementing request batching.

### DeepSeek API Costs
Each iteration sends ~2-4KB prompts to OpenRouter. At 3-minute intervals, expect ~480 requests/day. Monitor OpenRouter billing dashboard for cost tracking. The bot logs every API call to `ai_messages.csv` for billing reconciliation.

### Disk Space Management
CSV files grow unbounded. A bot running 24/7 generates:
- `portfolio_state.csv`: ~1 row every 3 minutes = ~20KB/day
- `ai_messages.csv`: ~2 rows (request+response) every 3 minutes = ~500KB/day (largest file)
- `trade_history.csv`: Variable (depends on trade frequency)
Plan for ~1-2 GB/year with moderate trading activity. Implement log rotation if running long-term.

## Quick Reference: Critical Files

| File | Purpose | Restart Behavior |
|------|---------|------------------|
| `bot.py` | Main trading loop | Reloads state from `portfolio_state.json` |
| `dashboard.py` | Streamlit monitoring UI | Reads CSV files, no state persistence |
| `data/portfolio_state.json` | Latest balance, positions, iteration counter | **Survives restarts** (bot resumes where it left off) |
| `data/equity_history.json` | Sortino ratio calculation data | **Survives restarts** (history preserved) |
| `data/*.csv` | All logs and historical data | **Survives restarts** (append-only) |
| `.env` | API credentials (ignored by git) | Must exist before first run |

## Common Operations

**Change trading symbols**: Edit `SYMBOLS` and `SYMBOL_TO_COIN` dicts at bot.py:49-57. Ensure Binance supports the symbol with 3m candles.

**Adjust check interval**: Modify `CHECK_INTERVAL` at bot.py:108. Must match `INTERVAL` for candle closure alignment (e.g., `INTERVAL="5m"` → `CHECK_INTERVAL=300`).

**Reset bot state**: Delete `data/portfolio_state.json` and `data/equity_history.json`. The bot will start fresh with `START_CAPITAL` on next run.

**Customize system prompt**: Edit `TRADING_RULES_PROMPT` at bot.py:59-104. Changes take effect immediately on next iteration.

**Change leverage limits**: DeepSeek controls leverage per decision. To enforce hard caps, add validation in `execute_entry()` at bot.py:1167-1193.
