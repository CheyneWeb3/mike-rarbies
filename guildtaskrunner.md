# GUILD Activator Bot – Documentation

## 1) Purpose

**Goal:** enforce GUILDCoin’s inactivity decay *proactively* by applying decay to **dormant, non-exempt holders** on a schedule.
This:

* Avoids “surprise haircuts” when a user finally transacts after months.
* Steadily feeds **rewardPool** and **treasury** (per split rules).
* Keeps supply circulating and aligns with the token’s “use it or lose it” concept.

**Non-goals:**

* No user rewards distribution. (Your current contract only accumulates to `rewardPool`.)
* No interest/APY for the parking vault (it’s decay-shield only).

---

## 2) How it works (Operation)

### High-level flow (per run)

1. **Discover holders** from on-chain `Transfer` logs (incremental, with a saved checkpoint).
   → Combine with any previously known addresses.
2. **Filter** to current **non-exempt**, **non-zero** balances.
3. **Determine inactivity:** compute `daysInactive = today - lastActiveDay`.
4. **Select eligible** wallets:

   * `FULL_SWEEP=true` ⇒ everyone non-exempt & non-zero.
   * Else `daysInactive ≥ MIN_INACTIVE_DAYS`.
5. **Apply decay**:

   * Preferred: `batchMarkActive(address[] users)` with **≤150** users per tx.
   * Fallback: per-user `markActive(user)` if batch fn not present.
6. **Persist state** (last scanned block; updated `holders.json`).

### What “apply decay” does

* Calls `_applyDecay(user)` in the token:

  * Burns the burn share.
  * Transfers the rewards share to `rewardPool`.
  * Transfers the treasury share to `treasury`.
  * Sets `lastActiveDay[user] = today`.

**Gas cost:** paid by the bot wallet.

---

## 3) Contract prerequisites

* **Token must expose**:

  * `isTrustedActivator(address) -> bool`
  * `isExemptFromDecay(address) -> bool`
  * `balanceOf(address) -> uint256`
  * `lastActiveDay(address) -> uint64`
  * `paused() -> bool`
  * `markActive(address)` *(required)*
  * `batchMarkActive(address[])` *(optional but recommended; bot falls back if missing)*
* **Bot wallet must be trusted**: owner calls `setActivator(bot, true)`.
* **Exempt system addresses** (LPs, vault, rewardPool, treasury, CEX, etc.) via `setExempt(addr, true)` as applicable.
* **Vault exemption:** after deploying the vault, call `enableDecayExemption()` (by the token owner) so parked tokens never decay.

---

## 4) Security & safety

* **Hot key risk:** the bot’s PRIVATE\_KEY spends gas and can submit txs; secure it:

  * Run on a locked-down VM.
  * Restrict shell access; use least privilege.
  * Consider a smaller “gas only” key with limited funds.
* **Rate limiting:** bot throttles logs scans and txs; tune delays as needed.
* **Gas limits:** batches are capped at 150 by default to avoid out-of-gas.
* **Ethics/UX:** with the bot on, idle balances **shrink without user action** (by design). Communicate this clearly to your community.

---

## 5) Repository layout (suggested)

```
guild-activator-bot/
├─ package.json
├─ .env                    # env vars (do NOT commit)
├─ guild-activator-bot.js  # main script (build holders + activate)
├─ holders.json            # generated list (auto-written)
├─ state.json              # last scanned block checkpoint (auto-written)
└─ logs/                   # pm2 logs (created by pm2)
```

---

## 6) Configuration (.env)

```ini
# Chain & Auth
RPC_URL=https://your-rpc-endpoint
PRIVATE_KEY=0xyour_bot_wallet_private_key
TOKEN_ADDRESS=0xYourGuildToken

# Holder discovery
START_BLOCK=123456         # token deploy block (first Transfer)
CHUNK_BLOCKS=5000          # log scan chunk size (reduce if RPC limits)
STATE_FILE=./state.json    # checkpoint for last scanned block
OUTPUT_HOLDERS_FILE=./holders.json

# Activation
BATCH_SIZE=150             # hard cap <= 150 (safe default)
MIN_INACTIVE_DAYS=1        # run daily: 1; full sweep: 0
PER_TX_DELAY_MS=1500       # pause between batch txs
DRY_RUN=false              # true = simulate (no txs)
FULL_SWEEP=false           # true = ignore inactivity threshold

# Parallelism
SCAN_CONCURRENCY=8         # concurrent balance/exempt checks
```

**Notes**

* Set `START_BLOCK` to your token’s **deployment block** to avoid scanning unrelated history.
* **Full sweep** (one-off): `FULL_SWEEP=true` and `MIN_INACTIVE_DAYS=0`.

---

## 7) Installation (Node.js)

### Prereqs

* Node.js ≥ 18
* A working RPC endpoint for your chain
* Bot wallet with a small gas balance

### Steps

```bash
# 1) create directory & files
mkdir guild-activator-bot && cd $_
# Save the provided package.json and guild-activator-bot.js into this folder
# Create .env with your values (see section 6)

# 2) install deps
npm i

# 3) dry run (no transactions)
npm run dry

# 4) live run (sends txs)
npm start
```

On the **first run**, the bot will:

* Scan logs from `START_BLOCK` → latest in chunks,
* Build `holders.json` (non-exempt, non-zero balances),
* Activate eligible wallets in batches.

Subsequent runs:

* Continue from `state.json.lastScannedBlock` (incremental).

---

## 8) Running with PM2 (daemon + restart on crash)

### Install PM2 globally

```bash
npm i -g pm2
```

### Option A: Start with inline env (quick start)

```bash
# from the project directory:
pm2 start guild-activator-bot.js --name guild-activator \
  --time \
  --update-env
```

**Notes**

* PM2 reads `.env` automatically for Node “module type: module”. If not, you can prefix env vars:

  ```bash
  RPC_URL=... PRIVATE_KEY=... TOKEN_ADDRESS=... pm2 start guild-activator-bot.js --name guild-activator
  ```

### Option B: PM2 ecosystem file (recommended)

Create `ecosystem.config.js`:

```js
module.exports = {
  apps: [
    {
      name: "guild-activator",
      script: "./guild-activator-bot.js",
      node_args: "",
      time: true,
      env: {
        NODE_ENV: "production",
        // You can inline env here or rely on .env
        // RPC_URL: "https://your-rpc",
        // PRIVATE_KEY: "0x....",
        // TOKEN_ADDRESS: "0xYourGuildToken",
        // START_BLOCK: "123456",
        // CHUNK_BLOCKS: "5000",
        // STATE_FILE: "./state.json",
        // OUTPUT_HOLDERS_FILE: "./holders.json",
        // BATCH_SIZE: "150",
        // MIN_INACTIVE_DAYS: "1",
        // PER_TX_DELAY_MS: "1500",
        // DRY_RUN: "false",
        // FULL_SWEEP: "false",
        // SCAN_CONCURRENCY: "8"
      }
    }
  ]
}
```

Start & manage:

```bash
pm2 start ecosystem.config.js
pm2 status
pm2 logs guild-activator
pm2 restart guild-activator
pm2 stop guild-activator
pm2 delete guild-activator
```

### Auto-start on reboot

```bash
pm2 startup
# follow the printed command (requires sudo)
pm2 save
```

---

## 9) Scheduling (PM2 + cron)

You have two patterns:

**A. Permanent service (always on):**
The bot runs once per invocation and exits. To make it run **daily**, use a **PM2 cron restart**:

```bash
# run daily at 02:15 UTC (adjust timezone/UTC expectations)
pm2 start guild-activator-bot.js --name guild-activator --cron "15 2 * * *" --time
pm2 save
```

PM2 will **start** the script at that time daily, and it will exit when finished.

**B. Classic system cron:**

```bash
crontab -e
# add:
15 2 * * * cd /path/to/guild-activator-bot && /usr/bin/node guild-activator-bot.js >> logs/run.log 2>&1
```

---

## 10) Common operations

* **One-off full sweep now** (apply decay to everyone non-exempt, regardless of days):

  ```bash
  FULL_SWEEP=true MIN_INACTIVE_DAYS=0 npm start
  ```

* **Increase safety if RPC is flaky**:

  * Lower `CHUNK_BLOCKS` (e.g., 2000).
  * Lower `SCAN_CONCURRENCY` (e.g., 4).
  * Increase `PER_TX_DELAY_MS` (e.g., 2500–3000).

* **Rotate key**:

  * Stop PM2, replace `PRIVATE_KEY` in `.env`, `pm2 start` again.

* **Reset state (force full rescan)**:

  * Delete `state.json` and `holders.json`, then run again.

---

## 11) Troubleshooting

* **`This wallet is NOT a trusted activator`**
  → From token owner: `setActivator(<bot>, true)`.

* **`Token paused; exiting.`**
  → Unpause the token with `unpause()`.

* **RPC `rate limit exceeded` / `missing trie node` / timeouts**
  → Increase `PER_TX_DELAY_MS`, reduce `SCAN_CONCURRENCY`, reduce `CHUNK_BLOCKS`. Consider a better RPC.

* **Out-of-gas in batch tx**
  → Reduce `BATCH_SIZE` to 100 or 75.

* **Holders list missing addresses**
  → Verify `START_BLOCK` equals the token’s deploy block; ensure the node has log history. Delete `state.json` and re-run to rebuild.

* **Batch not found / revert**
  → Your token may not include `batchMarkActive`. The bot auto-falls back to per-user `markActive`.

* **Balances are not decaying in the vault**
  → Ensure `GUILDStakingVault.enableDecayExemption()` was executed **by the token owner** (the function internally calls `token.setExempt(vault, true)`, which is `onlyOwner` on the token).

---

## 12) Environment variable reference

| Var                   | Meaning                            | Typical          |
| --------------------- | ---------------------------------- | ---------------- |
| `RPC_URL`             | JSON-RPC endpoint                  | `https://…`      |
| `PRIVATE_KEY`         | Bot wallet (hex, 0x…)              | `0xabc…`         |
| `TOKEN_ADDRESS`       | GUILDCoin address                  | `0x…`            |
| `START_BLOCK`         | First block to scan for `Transfer` | deploy block     |
| `CHUNK_BLOCKS`        | Log scan chunk size                | 5000             |
| `STATE_FILE`          | JSON file to save lastScannedBlock | `./state.json`   |
| `OUTPUT_HOLDERS_FILE` | Where holders are saved            | `./holders.json` |
| `BATCH_SIZE`          | Users per batch tx (≤150)          | 150              |
| `MIN_INACTIVE_DAYS`   | Eligibility threshold              | 1                |
| `PER_TX_DELAY_MS`     | Delay between batches (ms)         | 1500             |
| `DRY_RUN`             | `true` = no txs                    | `false`          |
| `FULL_SWEEP`          | `true` = ignore inactivity days    | `false`          |
| `SCAN_CONCURRENCY`    | Parallel RPC calls for scanning    | 6–10             |

---

## 13) Upgrade paths

* **Introduce/enable `batchMarkActive`** in the token (if not already). The bot will auto-use it.
* **Whitelist/blacklist**: add simple filters before batching (e.g., skip multi-sig, team wallets).
* **Reporting**: write CSV/JSON of processed addresses, days inactive, tx hashes after each run.
* **Metrics**: add a small Prometheus exporter (counts, gas, failures).

---

## 14) What this does *not* do (by design)

* It does **not** pay out “rewards” to users. The `rewardPool` address simply **accumulates** tokens.
  If, later, you want a yield flow, deploy a separate distributor contract that consumes `rewardPool`.

---
