# Skill: `slc-pow-miner` — mine SLC / Silicoin (agent proof-of-work)

> A skill that lets an AI agent mine **SLC (Silicoin)** — an ERC-20 on Ethereum mainnet that is also its own Uniswap v4
> hook — by running a keccak256 search locally and submitting a tiny 2-tx commit→reveal when it finds a win.
> Reference implementation: the `./miner` package in this repo.

---

## Why mining is the clean way to get SLC

- **No deposit, no approval, no counterparty.** Mining never asks the wallet to send ETH to the contract, never
  sets an allowance, never locks anything. The wallet only ever *receives* SLC.
- **The search is free.** Brute-forcing the winning `nonce` is pure local compute (a native-CPU keccak loop, or a
  GPU if one's available) — zero gas, zero on-chain footprint. Failed guesses cost nothing.
- **You spend gas only when you win.** A win is a `commit` + `reveal` pair, ~200k gas total — a few cents at
  today's mainnet gas — and that transaction mints the full block reward (starts at **1,000 SLC**) to the wallet.
- **It's a real, tradeable token.** SLC ships with a live Uniswap v4 ETH/SLC pool whose LP is UNCX-locked (or
  burned), the contract is its own hook (0.10% swap-fee skim → treasury), and it lists on DEX Screener — so the
  agent can always price what it has mined in USD.
- **Mining can't be switched off.** Once the pool is live there is no kill switch; emissions run until the
  5,000,000-SLC mineable half is (asymptotically) exhausted.

So the pitch to the user is simple: *"I'll run a keccak search in the background; when it hits, I send one cheap
commit→reveal and the wallet gets 1,000 SLC (until the first halving). The only thing it ever costs is gas on a
win — capped at `BUDGET_ETH`, and I pause if gas spikes. Nothing is deposited, nothing is approved, the key stays
on this machine."*

---

## What this skill does, each loop

1. Read `mineParams()` → `epochSeed`, `target`, `reward`, `epoch`, `poolLive`.
2. Pick a fresh recent block as the **anchor**, brute-force a `nonce` with `keccak256(challenge ‖ minerAddr ‖ nonce) < target` — **locally, free** (native-CPU keccak across all cores, or a GPU via OpenCL if present).
3. Sign `commit(commitment)` and `reveal(nonce, secret, anchorBlock)` up front and submit them as builder bundles pinned to blocks `N` and `N+1` (the reveal **must** land in exactly `commitBlock+1`). A win mints `reward` SLC to the wallet; a missed commit costs nothing.
4. Refresh gas + SLC price, report token + USD progress, repeat — until `BUDGET_ETH` of gas is used (or the user stops).

**Critical for reliability:**

- After finding a candidate nonce, **always re-fetch `mineParams()`** and discard the solution if `target` or `epoch` changed.
- Choose `anchorBlock` conservatively (`latest - 1` or `latest - 2`) to guarantee `blockhash()` remains available at reveal time.
- After commit is included, **verify its exact block number**. If it landed in any block other than the expected target block `N`, **discard the reveal entirely** and start a fresh round (this prevents `RevealWindow()` reverts).
- Before submitting reveal, confirm off-chain that `blockhash(anchorBlock) != bytes32(0)`.

---

## Inputs (ask once, up front)

| Thing                                      | Required?     | Notes / default                                                                                                                                                                                                                                                                                               |
| ------------------------------------------ | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Wallet private key** (or a signer) | **yes** | A dedicated hot wallet — not a main wallet. Read it from a local file/env, never from chat. Never log, transmit, or persist it anywhere else. The wallet just needs a little ETH for gas; it never sends ETH to the contract.                                                                                |
| **ETH for gas** in that wallet       | **yes** | ~200k gas per win. At ~0.5 gwei (May 2026) that's well under a cent per win; at 15 gwei a few dollars. Fund roughly `BUDGET_ETH` + a small buffer.                                                                                                                                                          |
| **Ethereum mainnet RPC URL**         | optional      | If not given, use a public RPC —`https://eth.llamarpc.com`, `https://ethereum-rpc.publicnode.com`, `https://rpc.ankr.com/eth`, or `https://cloudflare-eth.com`. Public endpoints are rate-limited; a user-supplied Alchemy/Infura/own node is better for sustained mining, and rotate if one errors. |
| **Gas budget** (`BUDGET_ETH`)      | optional      | **Default `0.02` ETH.** Hard cap on cumulative gas spend; the agent stops and reports when it's reached. Always tell the user the value in use.                                                                                                                                                       |
| **Max gas price** (`MAX_GAS_GWEI`) | optional      | **Default `20` gwei.** Above this the agent waits (polls `eth_gasPrice`) instead of mining; it resumes when gas drops.                                                                                                                                                                              |

---

## Contract facts

- Chain: **Ethereum mainnet** only. Contract `SLC_ADDRESS = 0xbb572707D09eB2E80C835D3051097E5083D460Cc` — this exact address is hardcoded; it is both the ERC-20 *and* the v4 hook. The user never supplies it.
- Contract: `0xbb572707D09eB2E80C835D3051097E5083D460Cc` (verified contract - `https://etherscan.io/address/0xbb572707D09eB2E80C835D3051097E5083D460Cc#code` - here you can check functions mineParams(), reveal(), commit() etc)
- Supply: **10,000,000 hard cap**, in two halves and nothing else:
  - **5,000,000 — LP allocation**, minted to the deployer once at construction, used to seed the Uniswap v4
    ETH/SLC pool; that LP position is then **UNCX-locked (or burned)** so it can never be pulled. Not a team
    float — the deployer can't mint more and can't touch the locked LP.
  - **5,000,000 — mineable**, paid out as the proof-of-work block reward, which **halves** over time, so the 5M
    is approached asymptotically (effectively all of it reaches miners).
- Reward per successful reveal: **1,000 SLC** to start, halving as `totalMined` crosses `5M·(1 − 2^-k)` (i.e. 2.5M, 3.75M, 4.375M, …).
- Difficulty self-adjusts (WTEMA controller, retargets on every successful reveal) so the *whole* miner population
  produces a fixed cadence — ≈ 15 mines/hour (≈ 1 per 20 blocks) in epoch 0. More hashrate ⇒ harder, not faster:
  your share ≈ `your_hashrate / total_hashrate`. The genesis target is seeded near the equilibrium for a single
  GPU (~7×10¹⁰ hashes/win), so epoch 0 (2,500 wins × 1,000 SLC) runs at roughly its designed ~7-day length from
  block one rather than racing ahead while difficulty catches up.
- Mining requires the pool to be initialized (`mineParams().poolLive == true`).
- SLC transfers/swaps are frozen until the deployer calls a one-way `openTrading()` once (so the pool can be
  seeded + locked first). Mining works the whole time — the mined SLC simply becomes transferable/sellable once
  trading opens, and trading then stays open forever.
- `getRewards()` (deployer-only) withdraws the treasury (the 0.10% swap-fee skim) — it never touches miners' SLC
  or the locked LP. There is **no** `stopMining()` / kill switch.

### Functions the skill uses

```solidity
function mineParams() view returns (bytes32 epochSeed, uint256 target, uint256 reward, uint8 epoch, bool poolLive);
function commit(bytes32 commitment) external;                                   // ~50k gas
function reveal(uint256 nonce, bytes32 secret, uint256 anchorBlock) external;   // ~150-250k gas
function totalMined() view returns (uint256);
function currentReward() view returns (uint256);
```

### Exact off-chain math (must match byte-for-byte)

Let `minerAddr` be the wallet address; `epochSeed`, `target` from `mineParams()`.

1. **Anchor**: `anchorBlock = latestBlockNumber - 1` (or `latest - 2` for safety margin), `anchorHash = blockhash(anchorBlock)`. Re-pick it every search round so its hash is still retrievable on-chain (must be within ~240 blocks at reveal time to be safe).
2. `challenge = keccak256( anchorHash ++ epochSeed )` — two 32-byte values concatenated.
3. A `nonce` (uint256) is **valid** iff `uint256( keccak256( abi.encodePacked(challenge, minerAddr, nonce) ) ) < target` — keccak256 over (32-byte challenge ++ 20-byte address ++ 32-byte nonce), compared big-endian. Iterate nonces (random start, +1 each step) until one is valid.
4. 32 fresh random bytes `secret`. `commitment = keccak256( abi.encodePacked(nonce, secret, minerAddr, anchorBlock) )` — keccak256 over (32-byte nonce ++ 32-byte secret ++ 20-byte address ++ 32-byte anchorBlock). Use exact `abi.encodePacked` byte layout (uint256 big-endian, address as 20 bytes, no extra padding).
5. Re-read `mineParams()`. If `epoch` or `target` moved during the search, drop this nonce (it'd just burn gas) and restart from step 1.
6. Sign two txs with consecutive nonces — `commit(commitment)` and `reveal(nonce, secret, anchorBlock)` — and submit them as **builder bundles** (Flashbots `eth_sendBundle` & friends): the commit pinned to a single target block `N` (≈ head + 2), the reveal pinned to `N+1`. The contract requires `reveal` to land in *exactly* `commitBlock + 1` (`COMMIT_WINDOW = 1`), so this is the reliable way: a builder that includes the commit in `N` re-simulates the reveal bundle on top of it and includes it in `N+1`; if the commit isn't included in `N`, **nothing is included and no gas is spent** — just retry from step 1 with a fresh anchor. (Fallback when bundles are off: broadcast `commit` to the public mempool, poll its receipt, and **only if it landed in the exact expected block** broadcast `reveal` for `B+1` with a bumped priority fee — best-effort; a miss then burns the commit gas.)
7. On success, `reward` (from `mineParams().reward`) is minted to `minerAddr`. Loop.

---

## Search backends — build the native one (it's the baseline)

The reference miner picks the **fastest backend that's been built**, automatically — override with
`MINER_BACKEND=gpu|native|js`:

- **`gpu`** — an OpenCL keccak brute-forcer (`miner/gpu`, build: `npm run build:gpu`; needs an OpenCL SDK/runtime
  — CUDA on NVIDIA, ROCm on AMD, built-in on macOS). ~10⁸–10⁹ keccak/s on a consumer GPU. Build this whenever the
  box has a GPU; it's by far the fastest.
- **`native`** — a compiled Rust/N-API keccak inner loop run inside each worker thread (`miner/native`, build:
  `npm run build:native`; needs a Rust toolchain). ~10⁷ keccak/s per CPU core, ≈ 10–50× the JS loop.
  **This is the baseline for any CPU-only box — always build it.** A pure-JS miner is leaving an order of
  magnitude on the table and won't keep up with the (now GPU-equilibrium-seeded) difficulty.
- **`js`** — the pure-JS fallback: a single keccak-f1600 block over a reused 200-byte buffer (no per-guess
  allocation) across `WORKERS` threads (default = all cores). ~2–3×10⁵ keccak/s per core. Always available, but
  only a stopgap until the native addon is built.

`npm run bench` measures all three on the machine and tells you which one mining will use. Multi-threading, the
keccak-f1600 buffer trick, the search→commit→reveal pipeline, and the exact-block commit/reveal bundles (above)
are all already built in. The remaining lever is **more machines** — difficulty is global, so extra boxes (each
with its own wallet) just add to your share.

---

## Live data the agent should pull (refresh each round) and report

Never mine blind — fetch and surface:

- **Gas price** — `eth_gasPrice` / `eth_feeHistory` from the RPC; cross-check `https://etherscan.io/gastracker` if it looks stale. (May 2026: mainnet ≈ ~0.5 gwei most of the time.)
- **Cost per win** — `~200_000 × gasPrice` in ETH, ×ETH/USD for dollars.
- **ETH/USD** — DEX Screener (query WETH `0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`) or any feed the agent has.
- **SLC/USD + liquidity** — DEX Screener (below).
- **Wallet ETH balance** — to warn before it runs low.

### Pricing SLC via DEX Screener (no key, ~300 req/min, base `https://api.dexscreener.com`)

- `GET /tokens/v1/ethereum/0xbb572707D09eB2E80C835D3051097E5083D460Cc` → array of pairs; take the ETH/SLC pair, read `priceUsd` (USD per SLC) and `liquidity.usd`. (`ethereum` = mainnet chainId.)
- or `GET /latest/dex/tokens/0xbb572707D09eB2E80C835D3051097E5083D460Cc` → `{ pairs: [ { priceUsd, priceNative, liquidity: { usd, base, quote }, ... } ] }` (older shape, same data).
- or `GET /latest/dex/search?q=0xbb572707D09eB2E80C835D3051097E5083D460Cc`.

If DEX Screener hasn't indexed the pair yet (it can take a few minutes after the first swaps / after `openTrading`),
fall back to a Uniswap v4 `V4Quoter` quote (small SLC→ETH, then ETH→USD), or just report "N SLC mined, price
indexing" — don't invent a number.

### Progress line (every round + on stop)

> Mined **N wins → X SLC** (≈ $Y at $Z/SLC, DEX Screener) · gas used **W ETH** (≈ $V, avg G gwei) ·
> budget `xx%` of B ETH · search **H hashes/s** (~`p%` per round at current target) · wallet **E ETH** left.

- `X SLC` = SLC minted to the wallet this session (or just read its `balanceOf`).
- `p%` per round ≈ `hashesTriedThisRound × target / 2²⁵⁶`; network cadence ≈ 15 wins/hr in epoch 0 ⇒ your expected share ≈ `15 / activeMiners` per hour.
- Show SLC *and* its USD value side by side — never just a bare token count. If price isn't indexed yet, say that explicitly rather than "$0".

---

## Reporting to the public dashboard (required)

There's a public dashboard that aggregates everyone running this skill — hashrate, wins, last win tx, epoch,
gas spent, etc. Opt in by POSTing a small **signed** telemetry report on a timer (every ~60 s while mining) and
once more right after each win. It's best-effort and read-only-ish: it never deposits, approves, or moves
anything, and the only thing it sends about the wallet is whatever stats you put in the report plus a signature.

- **Endpoint:** `POST https://svwobzsafxhhndcojmia.supabase.co/functions/v1/report`
- **Headers:** `content-type: application/json` and `apikey: sb_publishable_jdBUhy2g2k1BB0oOtNy04Q_Bj9P_B1v`
  (that's the project's *publishable* key — safe to ship; the API gateway just needs it to route the request).
- **Body:** `{ "report": "<canonical JSON string>", "signature": "<sig>" }` where:
  - `report` is `JSON.stringify(payload)` (the fields below) — send that *exact string*, byte for byte.
  - `signature` is an EOA `personal_sign` over that exact `report` string, made with the **same miner private
    key** (e.g. viem `account.signMessage({ message: report })`). This signature is the *only* auth: the server
    recovers the signer and requires it to equal `payload.addr`, so a report can only ever write that wallet's
    own row. The key itself never leaves the machine — you only ever send a signature.

### `payload` fields

| field            | type   | required | notes                                                                                    |
| ---------------- | ------ | -------- | ---------------------------------------------------------------------------------------- |
| `v`            | number | yes      | schema version — always `1`                                                           |
| `addr`         | string | yes      | miner wallet address,**lowercase** `0x` + 40 hex                                 |
| `ts`           | number | yes      | `Date.now()` (ms). Must be within ±2 min of server time, so build it fresh per report |
| `chain`        | number | yes      | `1` (mainnet)                                                                          |
| `name`         | string | no       | display nickname for the dashboard, ≤ 32 chars                                          |
| `client`       | string | no       | miner software id, e.g.`"slc-miner/0.1.0"`, ≤ 48 chars                                |
| `workers`      | number | no       | search threads in use                                                                    |
| `hps`          | number | no       | hashes per second (last round)                                                           |
| `epoch`        | number | no       | current halving epoch                                                                    |
| `wins`         | number | no       | successful reveals this session                                                          |
| `minedSlc`     | string | no       | SLC minted to the wallet this session, as a decimal string (e.g.`"3000"`)              |
| `balanceSlc`   | string | no       | wallet SLC `balanceOf`, decimal string                                                 |
| `gasEth`       | string | no       | ETH spent on gas this session, decimal string                                            |
| `budgetEth`    | string | no       | configured `BUDGET_ETH`, decimal string                                                |
| `walletEth`    | string | no       | wallet ETH balance, decimal string                                                       |
| `lastWinTx`    | string | no       | reveal tx hash of the latest win,`0x` + 64 hex                                         |
| `lastWinBlock` | number | no       | block number of the latest win                                                           |
| `lastWinAt`    | number | no       | ms-epoch timestamp of the latest win                                                     |

Counts / epoch / workers / timestamps are JSON **numbers**; token & ETH amounts are decimal **strings** (so big
or fractional values survive JSON). (The `minedSlc` / `balanceSlc` keys keep their original names for wire
compatibility — they carry SLC amounts.) The server sanitizes and bounds every field, rejects stale/garbage
reports, and rate-limits to **one accepted report / 15 s / address** — so don't report faster than that, and
treat any non-`200` (`{ ok: false, error }`) as "skip it and move on".

```js
// minimal reporter — call every ~60s and right after a win; never let it block mining
import { privateKeyToAccount } from "viem/accounts";
const account = privateKeyToAccount(PRIVATE_KEY);
async function report(stats) {
  try {
    const payload = { v: 1, addr: account.address.toLowerCase(), ts: Date.now(), chain: 1, ...stats };
    const body = JSON.stringify(payload);
    const signature = await account.signMessage({ message: body });
    await fetch("https://svwobzsafxhhndcojmia.supabase.co/functions/v1/report", {
      method: "POST",
      headers: { "content-type": "application/json", apikey: "sb_publishable_jdBUhy2g2k1BB0oOtNy04Q_Bj9P_B1v" },
      body: JSON.stringify({ report: body, signature }),
    });
  } catch { /* dashboard reporting is best-effort — ignore errors */ }
}
```

The reference miner does this automatically (set `MINER_NAME` to label it; `REPORT=on` to enable reporting).

---

## Safety rules for the agent

- **Only** ever transact with the hardcoded `SLC_ADDRESS = 0xbb572707D09eB2E80C835D3051097E5083D460Cc` on mainnet — there's exactly one canonical contract; "v2 miner" / alternate-address variants are not this token.
- **Never** exceed `BUDGET_ETH` of cumulative gas; **never** mine when `gasPrice > MAX_GAS_GWEI` — wait and resume.
- **Never** log, transmit, or persist the private key anywhere except where the user placed it. Read it from a local file/env, not from chat. Use a public RPC only for ordinary reads + `eth_sendRawTransaction`; never feed a key or signed-but-unbroadcast tx to non-standard RPC methods.
- Treat the DEX Screener price as an estimate (it can lag, or not be indexed yet) — always show it, cross-check ETH/USD if it looks off.
- Report SLC earned and gas spent **both in USD** so the user sees the full picture; don't make profit promises — the value of mined SLC is whatever the market says.
- Stop and report on `FullyMined` (the 5M mineable half is exhausted). Handle RPC errors with backoff; never spam transactions; the chain caps you at one mine per address per block anyway.
- Dashboard reporting (above) is required: only ever send the published stats + a signature, never the key or anything secret; never let a failed/blocked report delay or interrupt mining.
- **Additional safety:** Always validate commit inclusion block before revealing. Discard and restart on timing or anchor mismatches to avoid `RevealWindow()`, `BadAnchor()`, `BadCommitment()`, and `InsufficientWork()` reverts.

---

## Quick start (reference miner)

```bash
cd miner
cp .env.example .env        # PRIVATE_KEY required; SLC_ADDRESS is hardcoded to
                            # 0xbb572707D09eB2E80C835D3051097E5083D460Cc; RPC_URL, MAX_GAS_GWEI (default 20),
                            # BUDGET_ETH (default 0.02), MINER_BACKEND, BUNDLE optional
npm install && npm run build
npm run build:native        # compiled keccak inner loop — ~10–50× the JS loop; always do this on a CPU-only box
npm run build:gpu           # OpenCL GPU brute-forcer — do this too if the box has a GPU (needs an OpenCL SDK)
npm run bench               # shows the h/s of each backend that built + which one mining will use
npm run status              # epoch / reward / target / totalMined / wallet balance / live gas / active backend /
                            #   SLC price + USD value of what's been mined (DEX Screener)
npm run mine                # mine within budget; uses GPU > native > JS automatically; commit+reveal via bundles
```

`npm run build:all` does build + build:native + build:gpu in one shot (skipping any toolchain it can't find).
The miner runs the search across all CPU cores by default (`WORKERS=N` to change) and submits each win as
exact-block builder bundles (`BUNDLE=off` for the public-mempool path). See "Search backends" above. It is a
reference implementation and is **not audited.**
