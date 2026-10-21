# holder_watch — read-only token and trader monitor

For **Solana, Ethereum, Base, BNB Chain and Arbitrum**, with alerts on Telegram:

- **Trader watch** (every chain): every buy and sell of the wallets you watch, in any token, with the
  token's price, liquidity, market cap and pool age.
- **Built-in traders** (every chain): for each chain you choose at setup, wallets with a good last 30
  days across every token they traded — profitable on several tokens, winning on most, holding for
  hours rather than scalping, not bots, still active. See [Built-in traders](#built-in-traders).
- **Trader discovery** (every chain): any token's most profitable traders, ranked by realized +
  unrealized profit, ready to watch.
- **Token monitor** (Solana): one SPL token's top holders, price and liquidity (configured for
  **CyberLeek / CYBERLEEK**, mint `ApZuxdpzMrbEYTGEzeY9afh5pj9d6qPRJCTgQYiipbKg`). It alerts when one
  of **your** rules fires and tells whether moved tokens were **sold on a DEX** or **sent to another
  wallet**. Optional: the trader watch also runs without it.
- **Major-token monitor** (built in): "Trader 1" (`HWM6KGKSQrJMeSnQLp1XotYshHBaRg1St5VEquaHyW3t`) is
  watched for its trades in SOL, BTC, ETH, JUP, BONK and the other major tokens.
- **Signal log**: every alerted trade is also one JSON line in `signals.jsonl`, a stable record for
  backtests and a future copy-trading tool. See [Signal log](#signal-log).

**What it never does:** ask for, store or use a seed phrase or private key; sign or send a
transaction; trade. It only reads public data and sends messages. Alerts report what happened and
which rule fired. They never tell you to buy or sell. Past results of a wallet are not a prediction.

> **Check the token address.** DEX Screener lists about 30 tokens named "CyberLeek". At least one hides
> a right-to-left Unicode character in its name and reports an implausible $642M of liquidity. The
> monitor only follows addresses in `config.json`, strips such characters from names it displays, and
> the token search in `setup` / `discover` flags look-alikes.

## Quick start

Needs Python 3.10+.

```bash
python -m venv .venv
.venv/bin/pip install -r requirements.txt          # Windows: .venv\Scripts\pip install -r requirements.txt
.venv/bin/python holder_watch.py setup             # Telegram, your chains and keys, built-in traders, a token's traders
.venv/bin/python holder_watch.py --once            # one check (the first one only notes where each wallet starts)
.venv/bin/python holder_watch.py                   # run continuously (Ctrl+C to stop)
```

`setup` walks through four steps and writes `.env` and `config.json` for you (keys are typed hidden
and tested before they are saved; `config.json` is backed up before every change):

1. **Telegram.** Paste the token from @BotFather; it checks the token, finds your chat ID when you
   message the bot (or add it to a group), sends a test message and saves both values.
2. **Chains.** Pick the chains you follow (`1,3,4` or `all`). It asks for the key each one needs
   (below) when it is missing.
3. **Built-in traders.** Watch the chains' built-in traders. A chain without a list yet can have one
   built on the spot from its active traders (several minutes per chain).
4. **A token's traders** (optional). Paste a token address or search by name or symbol (look-alikes
   are flagged; an `0x` address is looked up on every EVM chain), rank its traders by profit, pick who
   to watch (`1-5`, `1,3,7`, `all`), any token or only this one, and the smallest trade to alert in
   USD. It shows the expected API use and offers a Telegram test alert.

Each part also exists on its own: `telegram`, `presets`, `discover`, `traders`, `scorecard` (see
[Command line](#command-line)).

### Accounts and keys (`.env`, git-ignored, never in config.json)

| Variable | Needed for | Free plan |
|---|---|---|
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` | Telegram alerts. `python holder_watch.py telegram` sets both. By hand: message **@BotFather** → `/newbot` → copy the token; send your bot any message, open `https://api.telegram.org/bot<TOKEN>/getUpdates`, and copy `"chat":{"id":…}`. Without them, alerts go to the console and log file only. | free |
| `HELIUS_API_KEY` | **Solana**: the token monitor (the public RPC answers `getTokenLargestAccounts` with HTTP 429 every time), discovery and building the Solana list. The Solana trader watch works on the public RPC, slowly. Used automatically when `rpc_url` is left at the public default. Key: [dashboard.helius.dev](https://dashboard.helius.dev). | 1M credits/month, 10 requests/s |
| *(none)* | **Ethereum, Base, Arbitrum trader watch**: without any key, watched wallets' trades are read through each chain's public RPC (wallet-filtered logs, receipts and balances; checked 2026-10-07). | free |
| `ANKR_API_KEY` | **EVM chains, everything else**: BNB Chain (no public BNB Chain RPC allows the above), finding a token's traders, building built-in lists and scorecards on all four EVM chains (they need a wallet's history: Ankr's token-transfer index). Also a sturdier source for the watch. Key: [ankr.com/rpc](https://www.ankr.com/rpc/) → sign up → the key is the last part of any endpoint URL. | 200M credits/month |
| `ALCHEMY_API_KEY` | Optional instead of Ankr: the same, through Alchemy (`alchemy_getAssetTransfers`). | 30M compute units/month |
| `SOLANA_RPC_URL` | Optional: a full Solana RPC URL from another provider. Overrides `rpc_url`. | |
| `ETHEREUM_RPC_URL`, `BASE_RPC_URL`, `BSC_RPC_URL`, `ARBITRUM_RPC_URL` | Optional: your own node for one EVM chain (receipts, balances). Wallet histories still come from Ankr when its key is set. | |

The per-minute checks (how many transactions each watched wallet has sent) always go to free public
RPCs, so a key's quota is only spent on wallets that actually traded.

Keys never appear in logs, alerts, error messages or state files: every log line and error passes
through a masking filter, and keys inside URLs are masked too. `rpc_url` can reference variables as
`${MY_VAR}`; a literal `api-key=` in `config.json` is rejected at startup.

## Configuration (`config.json`)

Validated at startup. Every problem is listed at once, typos get "did you mean …?", and the program
exits with code 2. Keys starting with `_` are comments.

| Setting | Default | Meaning |
|---|---|---|
| `mint` | `PASTE_MINT_HERE` | Solana token for the token monitor. Leave the placeholder to run the trader watch only. |
| `rpc_url` | public mainnet | Solana RPC endpoint. See keys above. |
| `poll_seconds` | 120 | Time between token-monitor cycles (minimum 15). |
| `top_n` | 10 | How many of the largest non-excluded owners to watch (1–20). |
| `exclude_owners` | `[]` | Wallets to ignore: LP pools, burn addresses, exchanges. |
| `auto_exclude_pools` | `true` | Also ignore wallets recognised as DEX pools or burn addresses (see below). |
| `always_alert_owners` | `[]` | Wallets (e.g. the creator) where **any** outflow alerts, even if they are not top holders. |
| `labels` | `{}` | `{"<address>": "Team wallet"}`: names shown in alerts and `--list`, on any chain. They win over trader labels. |
| `rules.holder_drop_pct` | null | Rule 1: a watched wallet's balance drops by at least this % within the window. |
| `rules.combined_drop_pct` | null | Rule 3: the watched wallets' combined balance drops by at least this % within the window. |
| `rules.window_minutes` | 60 | Window for rules 1 and 3. |
| `rules.stop_price_usd` | null | Rule 4: price ≤ this. |
| `rules.trailing_stop_pct` | null | Rule 5: price ≤ peak × (1 − pct/100). The peak is persisted. |
| `rules.min_liquidity_usd` | null | Rule 6: liquidity of the deepest pool < this, or no trading pair at all. |
| `alert_cooldown_minutes` | see file | Per rule: `holder_drop_pct` 30, `always_alert_owners` 0, `combined_drop_pct` 30, `stop_price_usd` 60, `trailing_stop_pct` 60, `min_liquidity_usd` 60, `monitor_failure` 60 (also used for the trader watch's failure alert). |
| `heartbeat` | 10:00 America/New_York | `{"enabled", "time", "timezone"}`: daily "still alive" message. |
| `max_consecutive_failures` | 5 | Failed cycles (or trader checks) in a row before a "failing" alert. |
| `max_tracked_owners` | 50 | Cap on watched holder wallets, including former top holders. |
| `traders.wallets` | Trader 1 | Watched traders: `{"chain", "address", "label", "tokens", "min_trade_usd", "source_mint", "source_symbol", "pnl_usd", "roi_pct", "added", "preset"}`. `chain` is `solana` (default), `ethereum`, `base`, `bsc` or `arbitrum`. `tokens` and `min_trade_usd` override the settings below for that wallet. `setup`, `presets`, `discover` and `traders add` fill this in; a plain Solana address string also works. |
| `traders.poll_seconds` | 60 | Time between trader checks (minimum 15). See [How often to check](#how-often-to-check). |
| `traders.alert_buys`, `traders.alert_sells` | `true` | Which trades alert. |
| `traders.tokens` | `"any"` | `"any"`: their trades in every token. `"source"`: only the token each wallet was discovered on (wallets added by hand or built in still alert on every token). `"major"`: only [major tokens](#major-token-monitor-built-in). |
| `traders.min_trade_usd` | 10 | Smaller trades don't alert. Paid in the chain's coin: valued at its current price; in stablecoins: at face value; token-for-token swaps: at DEX Screener prices. |
| `traders.major_mints` | `[]` | Token addresses to count as major on top of the built-in lists. |
| `traders.major_min_market_cap_usd` | 500,000,000 | A token not on a list is also major when DEX Screener shows at least this market cap and $1M liquidity. `null`: the lists only. |
| `signal_log` | `signals.jsonl` | One JSON line per alerted trade ([Signal log](#signal-log)). `null` turns it off. |
| `discovery.*` | see file | How `discover` ranks traders; see [Trader discovery](#trader-discovery). |
| `trader_state_file` | `trader_state.json` | Where the trader watch remembers each wallet's place. |

Rule 2 is `always_alert_owners`. Set any rule to `null` to turn it off. The shipped `config.json` checks
the token every 30 seconds. It turns on holder drops (20%), combined drops (10%) and a $100,000
liquidity floor, and leaves the price stops off. It also puts the five largest non-pool wallets as of
2026-10-06 ("Whale 1"–"Whale 5") on any-outflow alerts. They were chosen by size only: all are regular
wallets that bought through the pool, none bot-like. It does the same for "Trader 1", picked for its
past results trading CYBERLEEK: the best overall result among the top 60 holders not already watched.
**These are starting values; set your own.** Restart the monitor after editing.

## Built-in traders

Each chain has a list of wallets that did well over **the last 30 days across every token they
traded**. `setup` offers the lists of the chains you pick; `presets add base bsc` watches them later,
`presets remove base` stops, `presets list` shows them.

**How a list is made** (`presets refresh CHAIN`):

1. **Candidates.** The chain's 8 trending pools of ordinary tokens (then its busiest), and in each the
   latest 300 trades of at least $100 (GeckoTerminal, no key needed; about 2 minutes at its rate limit):
   wallets that appear there at most 5 times, like people do, those in more pools first. Pools of the
   coin itself, major tokens and stablecoins are skipped, and so are pegged pairs — two stablecoins, two
   kinds of BTC, staked ETH against ETH — whose busiest wallets are arbitrage bots. (Measured on
   2026-10-07: Solana's busiest pools by volume were wash-traded — a median trade of $0.05, and their
   most active wallets made 1,000 transactions in 1–4 hours — while each trending pool's $100+ trades
   came from 125–150 different wallets over several hours.)
2. **Pre-screen**, one cheap look each, before any history is read: on Solana, 100 transactions within
   2 hours is a bot; on EVM chains (one free batched call), a wallet that sent over 5,000 transactions
   is a bot or a bundler, and one that never sent any is a smart-contract wallet whose trades others
   send, which the watch couldn't follow. The first 80 that pass are scored.
3. **Scorecard.** Each candidate's last 30 days of trades in **every** token, with the same
   accounting as discovery: average cost in the chain's coin, stablecoins converted, gas included,
   tokens still held valued at today's price.
4. **Picked** only when it:
   - made at least **$500** in total and **10%** on what it spent (a whale's thin margin on one lucky
     token isn't skill), over at least **3 different tokens**, and made money on **most** of them;
   - is **not a scalper**: no more than half of what it sold was held under **60 minutes** (first in,
     first out);
   - is **not bot-like**: signed its trades itself, at most 600 trades, never three or more sells in the
     same block as its buys, and at most 800 token transfers in 30 days;
   - traded in the **last 7 days**.
5. **Ranked** by profit in USD; the top 5 are kept, labelled "Base pro #1" and so on.

The shipped `presets.json` (built 2026-10-07) holds one trader each for Solana, Ethereum, Base and BNB
Chain; Arbitrum had none that passed. Few wallets do: of about 900 candidates scored that day, most
were bots (the pre-screen caught several hundred), many traded fewer than 3 tokens, and one BNB Chain
pick that passed failed again when re-scored hours later, so it was left out. Strict rules keep the
lists short, and they go stale: hence the monthly renewal below.

**Monthly renewal.** `presets renew` rebuilds the list of every chain whose built-in traders you watch
and watches the new list instead: traders that still qualify stay (with your settings for them), new
ones are added, the others are dropped; wallets you added yourself are never touched. A chain where
nobody passes keeps its current traders. It sends a Telegram summary ("Built-in traders renewed: Base:
2 trader(s) now, 1 new, 1 kept, 1 dropped …"). On the server, the installer enables a systemd timer
that runs it on the 1st of each month around 05:00 UTC and restarts the monitor when the watch list
changed (`systemctl list-timers holder-watch-presets.timer`; run it now with `sudo systemctl start
holder-watch-presets.service`; `--no-auto-renew` at install leaves it off). On a PC, schedule
`python holder_watch.py presets renew` monthly yourself (Windows: Task Scheduler) and restart the
monitor afterwards. A renewal of four or five chains takes up to an hour or two and a few million Ankr
credits.

A refreshed list is saved to `presets.local.json`, which wins over the shipped `presets.json` and
survives updates. `--top N`, `--days D` and `--add` (also watch it) adjust a refresh. A wallet without a
list can be checked the same way: `scorecard ADDRESS --chain base` prints its profit, ROI, tokens,
win rate, average holding time, share sold within 60 minutes, best tokens, and whether it passes.

A wallet that qualifies did well recently; that is all it says. A refresh or renewal needs a wallet's
history: an Ankr (or Alchemy) key on EVM chains, Helius on Solana. It costs roughly 1–10M Ankr credits
per EVM chain (80 transfer listings at 700, then receipts and balances at 200 each), or a few thousand
Helius credits on Solana.

## Trader discovery

`python holder_watch.py discover [TOKEN] [--chain CHAIN]` ranks a token's traders by **profit**, in the
chain's coin and in USD. TOKEN is an address or a name/symbol to search; an `0x` address needs
`--chain` when given on the command line.

1. **Candidates.**
   - Solana: the most recent transactions in the token's busiest pools (DEX Screener's list, pools with
     at least 2% of its 24 h volume), up to `scan_transactions` (2000), none older than
     `lookback_hours` (168); the `candidates` (40) wallets that traded the most and **signed** their
     trades (pools and programs never qualify), plus the current top holders.
   - EVM chains: the senders of the latest trades in the token's busiest pools (GeckoTerminal), twice
     `candidates` of them, because many senders are routers and bots that are left out later.
2. **Full history.** For each candidate, every trade in this token from its first buy (Solana: the
   transactions of its token account; EVM: Ankr's or Alchemy's transfer index, then each transaction's receipt
   and the wallet's coin balance around it). Average-cost accounting in the chain's coin: **realized**
   profit on each sell, plus **unrealized** profit on what it still holds at today's price. Fees and
   gas count as costs; stablecoin legs are converted at today's price.
3. **Ranking** by total profit, leaving out — and counting, under "Left out" — wallets that: have more
   than `max_wallet_transactions` (300) transactions on the token or more than `max_trades` (200)
   trades; sold in the same block they bought, twice or more (sandwich and arbitrage bots); are
   **scalpers** (more than `max_scalp_share`, 50%, of what they sold was held under
   `min_hold_minutes`, 60); sold tokens they weren't seen buying (their cost is unknown); bought less
   than `min_buy_usd` ($100); or lost money.

Each row shows profit, ROI, the coin spent, buys/sells, the share it still holds, its average holding
time and its last trade, then realized/unrealized and notes such as **"early: first buy 2 min after the
first pool opened"** (a sniper or insider, not necessarily skill). Sample (test harness, fake addresses):

```
   #  Wallet                                         Profit SOL    Profit $     ROI   Spent SOL  Buys/Sells  Avg hold  Holds  Last trade
   1  CAJbcqWTiFnbVaiFAH9T55G5TnwCwa3g6cBSVP1Bj6dN    +590.0000    +$59,000  +5900%     10.0000       1 / 0         -   100%  2026-08-22
      realized +0.0000, unrealized +590.0000 · early: first buy 2 min after the first pool opened
   2  AcYMaNxNo8sPQNUeWs8a6EW8DwTdzg4LmgWZVgzJtRvo     +12.0000     +$1,200   +240%      5.0000       1 / 1       1 h    40%  2026-09-19
      realized +6.0000, unrealized +6.0000

Profit = realized + unrealized (what it still holds, at today's price), in SOL, fees included; $ at today's SOL price.
Left out: 1 bot-like (sold in the same block it bought); 1 lost money or broke even; 1 sold tokens it wasn't seen buying; 1 bought less than $100.
```

Then pick who to watch (`1-5`, `1,3`, `all`, `none`). Options: `--watch N` saves the top N without
asking (for scripts and servers), `--json` prints the ranking as JSON, and `--scan`, `--hours`,
`--candidates`, `--show` override the `discovery` settings for one run.

**Cost and speed.** Solana: with Helius' `getTransactionsForAddress` (10 credits per 100 transactions)
a default run costs a few hundred to about 1,000 credits and takes under a minute; otherwise it falls
back to `getSignaturesForAddress` + `getTransaction` (1 credit per transaction, a few minutes). EVM
(needs `ANKR_API_KEY` or `ALCHEMY_API_KEY`): a run reads up to 80 wallets' history on the token, a few
million Ankr credits, in a few minutes (Ankr's free plan allows 50 history listings a minute and
GeckoTerminal's public API only 6 requests a minute when measured; both are paced automatically). A
wallet whose transfer list alone shows it is too busy is skipped without reading its transactions.

**Read the ranking with care.** It covers the candidates' history on this one token. A top wallet can be
an insider, a sniper, a lucky gambler, or one of several wallets of the same person; profits can also
come from tokens moved to other wallets (noted when it is over 10% of its buys). Past results are not a
prediction.

## Trader watch

Every `traders.poll_seconds` (60 s), the watched wallets are checked:

- **Solana**: each wallet's transactions since the last check (`getSignaturesForAddress` with `until`,
  then each new transaction).
- **EVM chains**: one batched call per chain to a free public RPC asks how many transactions each
  wallet has sent. Only a wallet whose count went up is read further: the blocks where its tokens moved
  (Ankr's or Alchemy's transfer index with a key; without one, the public RPC's logs filtered by the
  wallet), each transaction's receipt (token transfers, WETH/WBNB wrapping) and its coin balance just
  before and after the block. The balance is what makes it exact: it includes the ETH/BNB a router paid
  out internally after a sale, which no event shows. (Checked live: a real Base sale read through
  Base's public RPC came out as 4,077.4865 BRETT for 0.0085142166 ETH, GeckoTerminal's figures to the
  last digit.) The wallet stays "active" for 5 minutes after its count rises, so a trade an index lists
  late is still found; a block is never reported twice.

Trades are decided from balance changes: **BUY** and **SELL** for the chain's coin (SOL, ETH, BNB,
wrapped or not) or stablecoins, **SWAP** for token-for-token; selling the coin itself for stablecoins
(or buying it with them) is a trade of the coin. Stablecoins are USDC and USDT on every chain plus, per
chain, DAI, USDe, USDS, PYUSD, USD1, FDUSD, RLUSD, USDG, GHO, crvUSD, FRAX, USDbC, USDT0, BUSD, USDf
(addresses checked on DEX Screener, in `watcher/chains.py`), so moving between two of them is never
alerted. A Solana sell whose proceeds went to another wallet is
still a SELL. Transfers, airdrops, wrapping and failed transactions don't alert. All trades found in
one check go into **one message**:

```
Trader watch: 3 new trades · 2026-09-21 14:15 UTC

1. BUY by TEST #1 (ECKU…Cg99) · discovered on TEST: +$1,200, +240%
   Bought 1,250,000 WIF for 3.20 SOL ($320.00) at 14:14 UTC · new position · via Raydium CPMM · tx
   Wif Two (WIF) · price $0.0003000 · liquidity $85,000 · market cap $300,000 · pool opened 3 h ago · chart

2. BUY on Base by Base pro #1 (0xd2cd…1dbe) · built-in trader: +$5,400, +85% over 30 days
   Bought 50,000 DEGEN for 0.25 ETH ($750.00) at 14:14 UTC · tx
   Degen (DEGEN) · price $0.01200 · liquidity $500,000 · market cap $5,000,000 · chart

3. SELL on Base by Base pro #1 (0xd2cd…1dbe) · built-in trader: +$5,400, +85% over 30 days
   Sold 20,000 DEGEN for $240.00 in stablecoins at 14:14 UTC · tx
```

(Test harness output with links removed. Telegram links the wallet, the token, `tx` and `chart` to the
chain's explorer and DEX Screener.)

- **No history replay, no gaps.** A wallet's first check only notes where it is. After that, its place
  only moves forward once the alert is delivered: if Telegram is down, the trades are read again and
  reported next check. Restarts keep the place (`trader_state.json`).
- **Token details** come from DEX Screener (one request for up to 30 tokens per chain per check); a
  token it doesn't list yet is shown as "not listed on DEX Screener (yet)", and a name with hidden
  text-direction characters gets a warning.
- **Health.** `max_consecutive_failures` failed checks in a row send "Trader watch: N checks in a row
  failed", then "recovered". The startup message and the daily heartbeat include the trader watch
  (with the wallets per chain).

### How often to check

The default, **60 seconds**, reports a trade about 1–1.5 minutes after it happens and fits the free
plans comfortably. Estimated use a month (`setup` and `traders` print yours):

| Watched | Helius credits (Solana) | Ankr credits (EVM, with a key) |
|---|---|---|
| Trader 1 only (the shipped config) | ≈ 43K, plus ≈ 350K for the token monitor at 30 s | 0 |
| 5 built-in traders on each of 4 EVM chains | | ≈ 28M of 200M |
| 10 Solana wallets | ≈ 430K of 1M | |
| EVM wallets on Ethereum, Base, Arbitrum without a key | | 0 (public RPCs) |

The EVM checks themselves go to public RPCs and cost nothing; a key is only charged for reading the
trades of a wallet that made some (estimated at about 1.4M Ankr credits per wallet trading ten times a
day). A Solana check costs 1 Helius credit per wallet plus 1 per new transaction, so going to 30
seconds doubles that part. Public RPCs are rate-limited: past about 50 EVM wallets per chain, use 90–120
seconds.

## Major-token monitor (built in)

`config.json` ships with one watched wallet: **Trader 1** (`HWM6KGKSQrJMeSnQLp1XotYshHBaRg1St5VEquaHyW3t`,
picked on 2026-10-06 for its CYBERLEEK results), with `"tokens": "major"`. It reports Trader 1's buys,
sells and swaps in major tokens only, tagged "major-token monitor":

```
1. SELL by Trader 1 (HWM6…yW3t) · major-token monitor
   Sold 50.00 SOL for $5,900 in stablecoins at 14:15 UTC · SOL balance now 50.00 · via Jupiter · tx
```

**Major tokens** are, per chain (every address checked against DEX Screener, lists in
`watcher/known.py` and `watcher/chains.py`):

- Solana: SOL (sold for or bought with stablecoins), cbBTC, WBTC, WETH, JUP, JTO, PYTH, RAY, ORCA, W,
  RENDER, HNT, JLP, BONK, WIF, TRUMP, PENGU, JitoSOL, mSOL, JupSOL, bSOL.
- Ethereum: ETH, WBTC, cbBTC, wstETH, LINK, UNI, AAVE, ONDO, ENA, PEPE, SHIB.
- Base: ETH, cbBTC, cbETH, AERO, VIRTUAL. BNB Chain: BNB, BTCB, ETH, SOL, XRP, DOGE, CAKE. Arbitrum:
  ETH, WBTC, ARB, GMX, LINK, PENDLE.

Plus `traders.major_mints`, plus any token DEX Screener shows past `traders.major_min_market_cap_usd`
($500M) with at least $1M liquidity — so a new big coin counts without a code change, while a token
claiming a huge market cap on a shallow pool doesn't. A swap counts when either side is major.

- Its CYBERLEEK sells keep alerting through `always_alert_owners` (rule 2), so they are not reported
  twice.
- Over the month before 2026-10-07, Trader 1's 17 trades were all in CYBERLEEK and other small tokens,
  so this monitor would have stayed quiet: it speaks up when Trader 1 moves into or out of majors. To get
  all of its trades instead: `python holder_watch.py traders add HWM6KGKSQrJMeSnQLp1XotYshHBaRg1St5VEquaHyW3t --tokens any`.
- Any wallet on any chain can be watched the same way: `traders add <ADDRESS> [--chain base] --label
  "Name" --tokens major [--min-usd 100]`, or set `"tokens": "major"` on its entry. Remove the monitor
  with `traders remove HWM6KGKSQrJMeSnQLp1XotYshHBaRg1St5VEquaHyW3t`.

## Signal log

Each alerted trade is appended to `signals.jsonl` (setting `signal_log`) as one JSON line, written only
after the alert was delivered:

```json
{"v": 1, "time": 1790000030, "seen": 1790000060.0, "chain": "base", "wallet": "0xd2cd…", "label": "Base pro #1",
 "side": "buy", "token": "0x266c…", "symbol": "DEGEN", "amount": 50000.0, "other_token": null,
 "native": 0.25, "usd_paid": 0.0, "value_usd": 750.0, "price_usd": 0.012, "tx": "0x…"}
```

`time` is when the trade was made (block time), `seen` when the monitor alerted it; `side` is `buy`,
`sell` or `swap` (`other_token` is what a swap gave up); `native` is the coin paid or received, gas
excluded; `usd_paid` the stablecoins; `value_usd` the trade's size. The format is versioned (`v`), so
other tools can rely on it: backtesting "what if I had followed this wallet", or a separate,
opt-in copy-trading program. This monitor itself never trades.

## How the token monitor works

Each cycle:

1. **Price and liquidity** come from DEX Screener `GET /token-pairs/v1/solana/{mint}` (300 requests/min,
   checked against their API reference). That endpoint also returns pools where the token is the
   *quote* side, whose price belongs to a different token. The monitor keeps Solana pools with
   `baseToken == mint` and uses the deepest one (currently Raydium CPMM CYBERLEEK/SOL).
2. **Holders**: `getTokenSupply`, `getTokenLargestAccounts` (top 20 token accounts) and
   `getMultipleAccounts` (jsonParsed). Token accounts are mapped to owner wallets and balances are
   summed per owner. SPL Token and Token-2022 both work.
   - Every wallet that was ever watched stays watched after it leaves the top 20, and its token
     accounts are re-read each cycle. A closed account counts as 0, so a complete sell-out alerts.
   - `always_alert_owners` are found with `getTokenAccountsByOwner` even when they are tiny. Wallets
     already in the top 20 are not looked up again each cycle.
   - Every balance drop is double-checked against all of the wallet's token accounts. Tokens moved
     to a new account of the same wallet are not reported as an outflow.
   - **Pools are excluded automatically.** A pool's balance swings with every trade, so a big buy
     would look like a whale selling. Recognised as pools: Raydium pool authorities (Raydium's shared
     CPMM vault holds about 28% of CYBERLEEK supply), every pool address DEX Screener lists, any
     wallet whose account is owned by a DEX program, and the incinerator. `--list` shows why each
     wallet is excluded.
3. **Rules** are evaluated. All hits in a cycle go into **one message** with the current price and
   liquidity.
4. **Holder alerts are classified.** The wallet's latest transactions on its token accounts are
   fetched and each outflow is classified from the balance changes: **SELL** (tokens went into a DEX
   pool; shows the route and what was received), **TRANSFER** (tokens went to another wallet; shows
   which), or burn. Every outflow comes with a Solscan link. Instruction names alone are not trusted:
   on mainnet, Jupiter's `ClaimToken` moved tokens between eight wallets without any sale, and it is
   classified as a transfer.
5. **State** is saved atomically to `state.json` (temp file, fsync, rename). Restarts keep balance
   history, the price peak and cooldowns, and don't resend old alerts.

The trader watch runs in the same process on its own schedule (`traders.poll_seconds`) and keeps its
own `trader_state.json`, so switching the monitored token doesn't reset it. Without a token (`mint`
left as the placeholder) only the trader watch runs, with its own startup message and daily heartbeat.

**One event, one alert.** A drop is measured from the higher of the window start and the last alert
for the same wallet and rule. A sell-off is therefore reported once, and again only if the wallet
sells more. Cooldowns are per wallet for holder rules. For ongoing conditions (price below stop,
low liquidity) the alert repeats as a reminder after each cooldown while the condition lasts. A drop
that happens *during* a cooldown is not lost: it is reported when the cooldown ends, even if the
window has passed by then. "No trading pair" must be seen on two checks in a row, so a single empty
API reply doesn't trigger it.

**Health.** Network calls retry with exponential backoff and jitter on HTTP 429/5xx and on network
errors, and honour `Retry-After`. One failed source doesn't stop the rules that can still run: if
the price feed is down, holder rules still fire. Five failed cycles in a row trigger a "monitor
failing" alert (repeated hourly while it lasts), followed by a "recovered" message. A daily
heartbeat at 10:00 New York time (DST-aware) reports price, liquidity and the top holders' share of
supply, and the trader watch's activity.

### Sample alert (generated by the test harness, abridged, URLs removed)

```
TEST monitor: 3 of your rules fired · 2026-09-21 14:15 UTC
Price $0.001650 · liquidity $420,000 · Raydium CPMM TEST/SOL

1. Your rule holder_drop_pct = 20% within 60 min
   Whale B (8Gpy…mcSN): 40,000,000 → 0 TEST (−100.0%), first seen falling at 14:15 UTC — now holds 0 (sold out)
   Share of supply: 4.00% → 0.00%
   • SELL via Jupiter → Raydium CPMM: −40,000,000 TEST, received 61.25 SOL at 14:15 UTC · tx
   Wallet on Solscan

2. Your rule always_alert_owners (any outflow)
   Creator (DgX9…wqKn): 500,000 → 450,000 TEST (−10.0%), first seen falling at 14:15 UTC
   • TRANSFER −50,000 TEST to EtDh…iV6S at 14:15 UTC · tx

3. Your rule trailing_stop_pct = 30%
   Price $0.001650 is at or below $0.001680: peak $0.002400 (seen 2026-09-21 14:14 UTC) minus 30%
   Now 31.2% below the peak
```

Telegram turns wallet names, "tx" and the pool into links to Solscan and DEX Screener.

## Command line

| Command | What it does |
|---|---|
| `holder_watch.py setup` | Guided setup: Telegram, your chains and their keys, built-in traders, a token's most profitable traders, test alert. |
| `holder_watch.py telegram` | Connect a bot and chat (chat ID detected automatically), send a test message, save both to `.env`. |
| `holder_watch.py presets [list\|add\|remove\|refresh\|renew] [CHAIN…]` | The built-in traders. `add base bsc` watches those lists (`--build`: build a missing one first), `remove base` stops, `refresh base` rebuilds (`--top N`, `--days D`, `--add`), `renew` rebuilds every watched list and watches the new ones (what the monthly timer runs). Chains: `solana ethereum base bsc arbitrum`. |
| `holder_watch.py scorecard ADDRESS [--chain CHAIN] [--days D]` | A wallet's last 30 days across every token, and whether it passes the built-in trader rules. |
| `holder_watch.py discover [TOKEN] [--chain CHAIN]` | Rank a token's traders by profit (TOKEN: address, or a name/symbol to search). `--watch N`, `--json`, `--scan N`, `--hours H`, `--candidates N`, `--show N`. |
| `holder_watch.py traders` | List watched traders and the estimated API use. `traders add ADDRESS [--chain CHAIN] [--label NAME] [--tokens any\|source\|major] [--min-usd USD]` (on a watched wallet it changes those settings), `traders remove ADDRESS [--chain CHAIN]`. |
| `holder_watch.py` | Run the loop: token monitor (when a token is set) and trader watch. Sends a startup message. Stops cleanly on Ctrl+C / SIGTERM. |
| `holder_watch.py --once` | One token-monitor cycle and one trader check, including the heartbeat check. Exit code 1 if either had errors. |
| `holder_watch.py --list` | Print the token's current top holders: address, balance, % of supply, accounts, watched, excluded (and why), label. Read-only; doesn't touch `state.json`. |
| `holder_watch.py --test-alert` | Send a test message. |
| `--config PATH`, `-v` | Use another config file (`.env` is read from its folder); debug logging. |

Commands that change `config.json` keep a backup (`config.json.bak-<date>-<time>`), write it
atomically and put the old one back if the result doesn't validate. A running monitor picks up the
changes after a restart. Logs go to the console and to `logs/holder_watch.log` (5 MB × 5 files, UTC
timestamps).

## Tests

```bash
.venv/bin/pip install -r requirements-dev.txt
.venv/bin/python -m pytest -q
```

There are 228 tests against an in-memory Solana RPC, EVM chains (public nodes, Ankr, Alchemy), DEX
Screener, GeckoTerminal and Telegram:

- **Token monitor**: every rule; excluded and auto-detected pool wallets; a holder selling out and
  closing the account; a holder leaving the top 20; creator-wallet outflows; the combined drop with
  wallets joining and leaving; cooldowns, including a drop held back by one; restarts from
  `state.json`; failure and recovery alerts; the DST-aware heartbeat; retries and backoff.
- **Trader discovery** end to end: profit ranking, a launch-time whale found through the holder list,
  bots, scalpers, small traders, airdropped tokens, Helius bulk history and its fallback; on Base, a
  ranking from GeckoTerminal candidates and wallet histories that leaves out a loser, a scalper and a
  router contract, and a clear message when no transfer index (key) is available.
- **Trader watch** on Solana and EVM chains, for every alert path: baseline, buys, partial and full
  sells, swaps, sells paid to another wallet, sales for stablecoins, the coin sold for dollars, size and
  token filters, the major-token scope, per-wallet settings, delivery failures, restarts, failure
  alerts. On EVM chains also: trades the transfer index lists late, a very busy wallet read over several
  checks, wallets that keep their own transaction counts, the cost of a quiet wallet (one batched
  call to a public node), blocks whose state a node has pruned, which source each chain is read from
  (Ankr, Alchemy, your URL, or keyless public RPCs), Ankr's transfer index, the keyless watch through
  public RPC logs, falling back to the keyed node when public ones are down, and BNB Chain without a key.
- **Built-in traders**: the scorecard (scalpers, bots, too few tokens, small or losing results,
  inactivity, holdings valued at today's price), which pools candidates come from (real pegged and
  staked-ETH pools from Arbitrum), GeckoTerminal's rate limit, building a list, the shipped and
  refreshed lists, the monthly renewal (only built-in wallets swapped, your settings kept, a month with
  no pick, the Telegram summary, the restart flag), and `presets` / `scorecard` / `setup` with them.
- **Setup and commands** end to end: Telegram chat detection, an Ankr key checked on each chosen chain
  (a chain it isn't served for is named), setup without the key (keyless chains kept, BNB Chain
  skipped), token search across chains with look-alikes, an `0x` address with pools on two chains,
  safe `config.json` / `.env` edits, 1.1.0 configs, traders-only runs, and secret masking (Helius,
  Ankr, Alchemy, Telegram) in logs, alerts and state.
- **Real mainnet transactions** in `tests/fixtures/`: the Solana classifier and trade parser on
  CYBERLEEK trades (a Jupiter→Raydium sell, a multi-hop sell paid to another wallet, transfers, a buy, a
  version-1 transaction), and the EVM parser on swaps from Ethereum, Base, BNB Chain and Arbitrum (a Base
  sale whose ETH a router paid out internally matches GeckoTerminal's record exactly; senders whose
  contract traded its own funds are not mistaken for traders).

## Run it 24/7 on Linux (systemd)

**[DEPLOY.md](DEPLOY.md) has the full, step-by-step procedure**, written so an AI agent or a person
can follow it; **[CODEX_DEPLOY.md](CODEX_DEPLOY.md)** is a ready brief to hand an agent such as Codex.
In short:

```bash
python deploy/package.py                                   # on your PC: builds dist/holder-watch-<version>.tar.gz (+ .sha256), without secrets
# copy the archive, its .sha256 and your .env to /tmp on the server, then on the server:
tar -xzf /tmp/holder-watch-1.2.0.tar.gz -C /tmp && cd /tmp/holder-watch-1.2.0
sudo ./deploy/install.sh --env-file /tmp/holder-watch.env --replace-config
```

`deploy/install.sh` creates the unprivileged `holderwatch` user and installs into
`/opt/holder-watch`. It sets up a virtualenv, runs the test suite, validates the config, installs and
starts the `holder-watch` systemd unit and enables the monthly renewal timer
(`holder-watch-presets.timer`; `--no-auto-renew` leaves it off). It then checks that the first check
succeeds and that a Telegram test alert is delivered, and ends with `RESULT: …`. `--chains "base bsc"`
also watches those chains' built-in traders (building a missing list first); `--replace-config` makes
the package's `config.json` the server's (the old one is backed up). Re-running it updates the program
and keeps `.env`, `state.json`, `trader_state.json`, `signals.jsonl`, `presets.local.json`, `logs/`
and, without `--replace-config`, your `config.json`.

The service restarts 30 s after any crash and starts at boot. An invalid `config.json` (exit code 2)
is not restarted in a loop; `journalctl -u holder-watch` shows what to fix. Run only one copy
against the same state files: stop the service before using `--once` by hand. `--list`,
`--test-alert`, `discover`, `scorecard`, `presets` and `traders` are safe while it runs (restart it to
apply a changed watch list).

## Troubleshooting

- **`Holder data unavailable: RPC getTokenLargestAccounts: HTTP 429 … add HELIUS_API_KEY`**: you're on
  the public Solana RPC. Add the key to `.env` (`setup` offers to).
- **`BNB Chain: no RPC for 2 wallet(s); add ANKR_API_KEY to .env`**: BNB Chain wallets need a key (no
  public BNB Chain RPC lists a wallet's transfers). Add it; `setup` tests it before saving.
- **`… a wallet's history needs ANKR_API_KEY`** from `discover`, `presets refresh` or `scorecard` on an
  EVM chain: those read 30 days or more of a wallet's transfers, which only a transfer index (Ankr or
  Alchemy) provides. The trader watch itself works without it on Ethereum, Base and Arbitrum.
- **A public EVM RPC is down or rate-limits**: each chain has backups (`watcher/chains.py`), and with a
  key the keyed node answers instead. Your own node: `BASE_RPC_URL=…` etc. in `.env`.
- **`BNB Chain: the node no longer has the balances of N block(s)`**: the node keeps only recent state.
  Trades in those blocks show their token side only (a sale for BNB is then not alerted); stablecoin
  and WBNB trades are unaffected. It happens after the monitor was stopped for a while.
- **`Telegram delivery failed: HTTP 400 (Bad Request: chat not found)`**: wrong `TELEGRAM_CHAT_ID`, or
  you haven't sent your bot a message yet. `python holder_watch.py telegram` finds the right one.
- **`telegram` says the bot "has a webhook"**: another service receives its messages. Type the chat ID
  when asked, or use a separate bot for alerts.
- **`discover` is slow**: on Solana you're on the public RPC (3 requests/s): add a Helius key, or use
  `--scan 300`. On EVM chains the pace is set by GeckoTerminal's public API (about 6 requests a minute,
  waited out automatically when it says "too many").
- **`presets refresh` picks nobody**: the summary line says why each candidate was left out (scalper,
  bot-like, too few tokens, …). Busy days have more candidates; `--days 14` scores a shorter period.
- **No trader alerts**: `traders` lists who is watched; the first check of a wallet only notes where it
  is; check `traders.min_trade_usd` and `traders.tokens` (a built-in wallet with `"tokens": "major"`
  only reports majors).
- **`unknown time zone 'America/New_York'`** (Windows): `pip install -r requirements.txt` installs `tzdata`.
- **Reset the trailing-stop peak**: stop the monitor, set `"peak": null` in `state.json`, and start it
  again. The next price becomes the new peak.
- **Starting over**: stop it and delete `state.json` (and `trader_state.json`). If you change `mint`,
  the old state is set aside automatically.

## Limits

- Polling, not streaming: a sell is noticed within one `poll_seconds` (30 s in the shipped config),
  a trader's trade within `traders.poll_seconds` (60 s). Several trades between two polls show up as
  their net effect for holder rules; the trader watch reports each one (on EVM chains, trades of one
  token in the same block are combined).
- Holders are tracked from the top 20 token accounts, the RPC's maximum, plus wallets watched
  before. A wallet's balance is the sum of the token accounts the monitor knows about.
- Classification is best effort and always links the transaction. A sale through a pool type the
  monitor doesn't recognise may show as a transfer to that pool.
- Price and liquidity are DEX Screener's figures for the deepest pool.
- Discovery sees each candidate's history on that one token and values holdings at today's price;
  built-in lists see 30 days. Both find wallets that did well; neither predicts anything.
- Trades the wallet doesn't send itself aren't seen: on Solana, limit orders and DCA fills executed by
  a keeper; on EVM chains, gasless and intent-based orders (UniswapX, CoW Swap, 1inch Fusion) filled by
  a solver, unless the wallet also sends a transaction within a few minutes. The Solana watch reads at
  most 25 new transactions per wallet per check.
