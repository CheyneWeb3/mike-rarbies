# V2 Arbitrage System 

## What this is

A small system that **hunts and executes token price gaps** between Uniswap-V2-style DEXes on your chain. It has two parts:

1. **An on-chain executor contract** you own.

   * It performs the actual swaps when you tell it to.
   * It checks profit and **reverts if profit is below your threshold**, so you don’t get stuck with a loss.

2. **An off-chain Node service** (a simple server) that:

   * Discovers all pairs on each DEX.
   * Repeatedly reads live reserves.
   * Simulates trades to spot profitable opportunities.
   * Calls your contract to execute when profit clears your threshold.
   * Optionally keeps token **allowances topped up** and **sweeps balances** back to your wallet.

No Flashbots, no Aave, no v3 — **V2 only** .

---

## Why use it

* **Make markets pay you**: When the same token pair is priced differently on two V2 DEXes, you can buy where it’s cheaper and sell where it’s pricier — keeping the spread.
* **PnL in your chosen bases**: You decide which tokens you’re willing to hold (e.g., WETH/USDC/USDT). The system only starts/ends trades in those.
* **Safety rails**: The contract checks you actually finish with **more** of the base token than you started (and at least your minimum profit). If not, it reverts.

---

## The kinds of arbitrage it finds

1. **Cross-DEX, 2-leg**

   * Leg 1: Swap `tokenIn → tokenOut` on DEX A.
   * Leg 2: Swap `tokenOut → tokenIn` on DEX B.
   * Ends with more `tokenIn` (a base token you selected).

2. **Triangular, same DEX (A→B→C→A)**

   * Three hops inside **one router**, same V2 invariant everywhere.
   * Starts and ends in the same **base token** you allow.

You can toggle triangular arbs on/off.

---

## How it works (end-to-end)

1. **Discovery**

   * You provide each router’s **factory address**.
   * The service reads historical `PairCreated` events to learn **every pair** on that DEX.
   * Optional: you can give an **allowlist** of tokens to limit the universe (for speed/safety).
   * Result: a map of all pairs across all listed DEXes.

2. **Polling**

   * The service batches `getReserves()` calls through **Multicall3**, so reads are fast and cheap.
   * It refreshes reserves on a timer (you control the interval).

3. **Simulation**

   * For each shared pair symbol across two DEXes, it simulates both directions of the 2-leg cycle, searching for the **best input size** (binary/heuristic search).
   * For each triangle within one DEX, it simulates `A→B→C→A`.
   * It **ignores anything** that doesn’t start and end in your **BASE\_TOKENS**.
   * It keeps only **positive-profit** opportunities and remembers the best one currently available.

4. **Execution**

   * If the best opportunity’s profit ≥ your **MIN\_PROFIT** setting, the service tells your **executor contract** to run it.
   * The contract performs the swaps with generous on-chain minOut (set loose), but **enforces net profit in the end**. If net profit < your threshold, it **reverts** and you don’t complete the trade.

5. **Funds management (optional but handy)**

   * **Manage allowances**: For selected tokens/routers, top up approvals so the **first trade doesn’t waste gas** doing approvals.

     * Set with `MANAGE_TOKENS`, `MANAGE_ROUTERS`, and a `MIN_ALLOWANCE_WEI` level.
     * If `MANAGE_ROUTERS` is left **blank**, it manages approvals for **all** active routers you configured.
   * **Sweeping**: Periodically move balances sitting on the contract back to your wallet when they exceed your threshold (`SWEEP_MIN_WEI` → `SWEEP_TO`).

---

## Key concepts & knobs (simple meanings)

* **BASE\_TOKENS**
  Tokens you actually want to hold and count profit in. The engine will **only** run cycles that start & end in these.
  *Why:* You avoid ending up with random assets; PnL is clean.

* **TOKENS\_ALLOWLIST** *(optional)*
  A list of tokens to search over. Leave empty to try everything the factories report.
  *Why:* Speed + safety when a chain has thousands of junk pairs.

* **MIN\_PROFIT\_WEI**
  The **profit floor** in your base token’s smallest units. If the final balance increase is below this, the contract **reverts**.
  *Tip:* Set this high enough to cover gas + slippage noise.

* **SCAN\_INTERVAL\_MS**
  How often to refresh reserves and search again.
  *Trade-off:* Faster finds vs. more RPC pressure.

* **GAS\_DEADLINE\_SECS**
  How far into the future the swaps are valid.
  *Shorter:* less chance of price moving; *Longer:* more time to mine.

* **Multicall3**
  A helper contract that lets the engine **read many pairs in one RPC call** (faster reserve polling).

* **Routers & Factories**
  Each DEX has a **factory** (creates pairs) and a **router** (does swaps). You set both per DEX so we can **discover** pairs and **execute** swaps.

* **MANAGE\_TOKENS / MANAGE\_ROUTERS**
  Tell the service which tokens/routers to proactively maintain approvals for. Leave `MANAGE_ROUTERS` blank to cover all configured routers.

---

## What it does **not** do (by design)

* No **Flashbots or private mempool** protection (public mempool only).
* No **v3** math/paths.
* No **flash loans** (you supply capital up front).
* No price prediction; it reacts to **current reserves only**.

---


## Costs, profits, and reality checks

* **Gas**: Each execution is two or three swaps. Your `MIN_PROFIT` should be above typical gas for your chain.
* **Slippage/risk**: The system uses **live reserves** but the market can move between simulation and mining. If profit disappears, the contract reverts and you only pay gas on the reverted attempt.
* **MEV exposure**: Public mempool means arbs can be copied or sandwiched. Keeping `MIN_PROFIT` firm and scanning frequently helps.

---

## When to add the optional allowlist

Add a **token allowlist** if:

* You’re seeing **too many** pairs and the loop gets heavy.
* You want to **exclude** illiquid/honeypot tokens.
* You’re initial-testing and only care about blue chips.

---

## Monitoring & endpoints (plain descriptions)

* **Health check**: returns “ok” so you know the service is alive.
* **Pairs summary**: shows how many pairs were discovered per router.
* **Base tokens**: shows which base tokens are currently enforced.
* **Scan once**: triggers a one-off scan and returns the best 2-leg and triangular candidate it sees right now.

(These are simple endpoints exposed by the service; you can hit them with a browser or curl.)

---

## Troubleshooting

* **“No opportunities found”**

  * Increase scan frequency a bit.
  * Lower `MIN_PROFIT_WEI` temporarily (still above gas).
  * Add more DEX routers/factories if available.
  * Remove an over-restrictive `TOKENS_ALLOWLIST`.

* **“Approvals missing / first trade reverts”**

  * Use the **funds manager** to pre-approve: set `MANAGE_TOKENS`, leave `MANAGE_ROUTERS` empty (all), and set a sensible `MIN_ALLOWANCE_WEI`.

* **“Pairs look wrong / empty”**

  * Verify you matched each router to the **correct factory** (a router normally exposes a `factory()` getter; check they align).
  * Ensure `START_BLOCK` isn’t set **after** the pairs were created.

* **“Weird tokens / tiny pools”**

  * Enable a **token allowlist** for clean assets only.

* **“Too many failed (reverted) txs”**

  * Raise `MIN_PROFIT_WEI`.
  * Shorten `GAS_DEADLINE_SECS`.
  * Consider narrowing to the most liquid pairs.

---

## Security & ops hygiene

* Keep the **private key** in environment variables or a secrets manager.
* Run the server behind a **firewall**; don’t expose it widely.
* Use a reliable **RPC** with good rate limits.
* Start small with capital and scale as you gain telemetry.

---

## Roadmap ideas (when you’re ready)

* **Triangular paths on more routers**, and multi-triangle search pruning.
* **Private relay / bundles** to lower MEV exposure.
* **v3 support** (different math, more routing shapes).
* **Dynamic gas & minProfit** based on network load.
* **PnL dashboard** and per-pair throttles.

---

## Quick glossary

* **Base tokens**: The assets you’re happy to hold; all arbs start and end here.
* **Factory**: DEX contract that **creates pairs** (and emits `PairCreated`).
* **Router**: DEX contract that **executes swaps**.
* **Reserves**: The current balances of token0/token1 inside a pair.
* **Multicall3**: On-chain helper to **batch many read calls** into one.
* **Allowance**: Permission the executor gives a router to spend a token.
* **Sweep**: Move token balances from the executor contract back to your wallet.
* **Profit floor**: Minimum net gain required or the trade reverts.
