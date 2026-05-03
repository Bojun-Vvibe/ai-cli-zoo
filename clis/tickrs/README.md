# tickrs

> **Real-time stock-ticker terminal dashboard** — a Rust TUI
> built on `tui-rs` that streams Yahoo Finance quotes + intraday
> charts for an arbitrary watchlist of symbols (equities, ETFs,
> indices, crypto, FX), renders sparkline price charts at
> 1m / 5m / 15m / 30m / 1h / 1D / 1W / 1M / 6M / 1Y / 5Y
> intervals, supports pre-/post-market data, custom y-axis
> bounds, percentage-change colouring, options-chain views,
> summary tape mode, and reads symbols either from `--symbols`
> CLI flag or `~/.config/tickrs/config.yml`. Pinned to **v0.15.0**
> (commit `fc3095505d6b96add9e2e2062423dcb72224c66f`,
> [LICENSE](https://github.com/tarkah/tickrs/blob/master/LICENSE),
> MIT).

Source: <https://github.com/tarkah/tickrs>

## TL;DR

`tickrs AAPL NVDA SPY BTC-USD ^GSPC` opens a full-screen TUI
that polls Yahoo Finance every few seconds for the listed
symbols and renders, per symbol, a sparkline price chart with
configurable interval, current quote, percentage change vs the
day open, day high / low, and (optionally) volume. Toggle
between a per-symbol detail view (one big chart) and a summary
tape view (`s` key) showing all symbols in a compact table.
Time range cycles through `1` → `5` → `15` → `30` → `60` (min)
→ `D` → `W` → `M` → `6M` → `Y` → `5Y` with single keystrokes.
Pre-market and after-hours data render as a separate trace
when the market is closed (`-p` flag). Options chain (`o` key)
pulls the live calls / puts strikes for the focused symbol.
Ships as a single static Rust binary; no API key required
(uses Yahoo's public endpoints, same shape as the discontinued
Yahoo Finance API).

## Install

```bash
# Homebrew (macOS / Linux)
brew install tickrs

# Cargo
cargo install tickrs --locked

# Arch
yay -S tickrs

# release tarball
curl -L https://github.com/tarkah/tickrs/releases/download/v0.15.0/tickrs-v0.15.0-x86_64-apple-darwin.tar.gz | tar xz
sudo install tickrs /usr/local/bin/

# verify
tickrs --version    # tickrs 0.15.0
```

A persistent watchlist lives at `~/.config/tickrs/config.yml`:

```yaml
symbols:
  - AAPL
  - NVDA
  - GOOGL
  - SPY
  - QQQ
  - BTC-USD
  - ETH-USD
  - ^GSPC
  - ^IXIC
  - EURUSD=X
update-interval: 5      # seconds
time-frame: 1D
chart-type: line        # line | candle | kagi
show-volumes: true
show-x-labels: true
trunc-pre: false
enable-pre-post: true
```

`tickrs` (no args) loads it. CLI flags override per-invocation.

## License

MIT — see
[LICENSE](https://github.com/tarkah/tickrs/blob/master/LICENSE).
Permissive, no attribution required for binaries.

## One Concrete Example

```bash
# 1. one-shot watch list, defaults
tickrs --symbols AAPL,NVDA,SPY,BTC-USD

# 2. with pre-/post-market data and a custom interval
tickrs --enable-pre-post --time-frame 1D \
       --symbols AAPL,MSFT,GOOGL,NVDA,SPY,QQQ

# 3. summary mode (compact tape view of all symbols)
tickrs --summary --symbols AAPL,NVDA,SPY,QQQ,BTC-USD,ETH-USD
#    one row per symbol: ticker | price | %chg | day range | sparkline

# 4. inside the TUI keystrokes:
#       1 / 5 / G  → 1m / 5m / 15m intraday
#       D / W / M  → daily / weekly / monthly
#       6 / Y / 5  → 6M / 1Y / 5Y
#       s          → toggle summary view
#       o          → options chain for focused symbol
#       p          → toggle pre-/post-market overlay
#       v          → toggle volume bars
#       k          → toggle kagi chart
#       q          → quit

# 5. crypto + FX + index across one screen
tickrs --symbols BTC-USD,ETH-USD,SOL-USD,EURUSD=X,GBPUSD=X,^GSPC,^IXIC,GLD

# 6. headless tape via summary + tmux pane (for a status dashboard)
tmux new-session -d -s ticker 'tickrs --summary --symbols AAPL,NVDA,SPY'
tmux attach -t ticker
```

## Niche It Fills

**A free, no-API-key, terminal-native real-time price tape.**
Every "free" stock dashboard either (a) requires signing up for
an Alpha Vantage / Polygon / Finnhub key with strict free-tier
rate limits, (b) ships only as a web app behind a paywall after
N quotes, or (c) is a heavy desktop GUI. `tickrs` reads Yahoo
Finance's public quote + chart endpoints, the same source many
open-source finance tools use, and renders a TUI in any
terminal — no key, no signup, no quota you have to manage.
For an operator who keeps a watchlist and wants it visible
beside `htop` in a tmux pane, `tickrs` is the lowest-friction
option.

## Why use it

Three things that distinguish it from "open a browser tab":

1. **No account, no API key, no paywall.** Yahoo's public
   endpoints are queried directly; the rate limits are
   generous enough for a 5-second poll across ~30 symbols.
   Compare to Bloomberg Terminal ($25k/yr), TradingView Pro
   (~$15/mo), or any "free" SaaS that throttles to 5 quotes
   per minute on the free tier.
2. **Tmux / SSH-friendly.** Lives in a terminal pane next to
   your editor / shell — no window switching, no Alt-Tab into
   a browser, no "is the browser tab still polling?" doubt.
   On a remote box (a beefy desktop you SSH into from a
   laptop) the TUI re-attaches with `tmux attach` and the
   data picks up where it left off.
3. **Crypto / FX / index / equity in one view.** The same
   binary renders BTC-USD next to AAPL next to ^GSPC next to
   EURUSD=X — anything Yahoo Finance has a quote endpoint for.
   One unified watchlist instead of three separate apps for
   "stocks", "crypto", and "FX".

For an LLM-CLI workflow it is mostly background context — a
ticker pane in tmux, glanced at occasionally — but pairs well
with `aichat 'what just moved NVDA?'` for the explanation
layer when something jumps.

## Vs Already Cataloged

- **Vs [`bottom`](../bottom/) / [`btop`](../btop/) /
  [`s-tui`](../s-tui/):** same "live TUI dashboard" category,
  different content (system stats vs market quotes). They
  share a tmux-friendly footprint and pair well in a status
  board.
- **Vs [`gping`](../gping/) / [`vnstat`](../vnstat/):** also
  live time-series TUIs, but for network latency / bandwidth.
  `tickrs` is the equivalent for market data.
- **Vs a TradingView browser tab:** TradingView has the
  professional-grade charting, indicators, drawing tools, and
  real-time level-2 data; `tickrs` has zero of those — it is
  intentionally a terminal-native price tape, not a charting
  platform. Use TradingView when you actually need to draw
  Fibonacci retracements; use `tickrs` when you want a
  watchlist that lives in tmux next to `vim`.
- **Vs a `cron + curl + jq` script against an API:** rolling
  your own gives full control over rate limit, source, and
  storage but is a project; `tickrs` is one `brew install` and
  a config file. Pick the script when you need historical
  archival; pick `tickrs` for the always-on glance view.

## Caveats

- **Yahoo Finance is the data source — and it is unofficial.**
  Yahoo has changed / removed / re-added these endpoints
  several times over the years (the 2017 API shutdown is the
  canonical incident). When Yahoo changes the shape, `tickrs`
  needs an update — pin to a known-working version and watch
  the issue tracker if your workflow is critical.
- **Quotes are delayed for some exchanges.** US equities are
  effectively real-time on Yahoo; some international exchanges
  are 15-minute delayed. For trading-grade real-time you need
  a paid feed (and a different tool).
- **Not a portfolio tracker.** No position sizes, no cost
  basis, no P/L calculation, no transaction log — `tickrs`
  shows quotes, not your portfolio. Pair with a separate
  spreadsheet / `ledger` / `hledger` for accounting.
- **No charting indicators.** No moving averages, no RSI, no
  MACD, no Bollinger bands — just the raw price line / candle
  / kagi at the chosen interval. For technical analysis use
  TradingView or a Python notebook.
- **Symbol coverage = whatever Yahoo has.** Most US equities,
  ETFs, major indices (`^GSPC`, `^IXIC`, `^DJI`, `^VIX`), top
  cryptos (`-USD` suffix), and FX (`X` suffix) work. Obscure
  OTC tickers or international small caps may be missing or
  have stale quotes.
- **No alerting.** It does not page you when a price crosses
  a threshold; it just renders. Wire `tickrs` into a `cron` +
  `curl` script (or a small Python loop on the same Yahoo
  endpoint) for alerting.
- **Network dependency.** Offline = blank charts. There is no
  on-disk cache of historical bars to fall back on.
