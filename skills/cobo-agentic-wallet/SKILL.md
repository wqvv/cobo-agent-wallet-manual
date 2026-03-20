---
name: cobo-agentic-wallet-sandbox
description: |
  Cobo Agentic Wallet skill for the **sandbox** environment (`--env sandbox`, API: api-agent-wallet-core.sandbox.cobo.com).
  Use for agent wallet use cases and agent-wallet-related guidance — including Cobo Agentic Wallet operations via the `caw` CLI or Cobo API: wallet onboarding and setup, transfers of stablecoins (USDC, USDT, DAI) and tokens (ETH, WETH, WBTC, SOL, ARB, OP, MATIC), smart contract calls, balance and transaction queries, and policy denial handling.
  Covers DeFi execution through Cobo's MPC/TSS policy layer on EVM chains (Base, Ethereum, Arbitrum, Optimism, Polygon) and Solana (mainnet via Jupiter, devnet): Aave V3 borrow/repay, Uniswap V3 swaps, DCA strategies, grid trading, and prediction market positions.
  Use when: user mentions caw, cobo-agentic-wallet, MPC wallet, TSS node, Cobo Portal; needs smart contract or DeFi protocol access through a policy-enforced agent wallet; or asks about agent wallets, policy-enforced wallets, or how agents manage crypto in general — and the target environment is sandbox.
  NOT for: fiat currency operations — credit card payments, bank wire transfers, or crypto-to-fiat off-ramp flows.
---

# Cobo Agentic Wallet (Sandbox)

Cobo Agentic Wallet is a policy-enforced crypto wallet for AI agents. Owners set spending limits and approve transactions; agents operate within those guardrails. The `caw` CLI is the primary interface.

## Execution Rules (CRITICAL — read before running any command)

1. **No re-execution.** Run each command exactly once. If a command is still running, wait for it to exit — do NOT launch it again in a new session.
2. **No polling loops.** Both `bootstrap-env.sh` and `caw onboard` print progress to stdout as they run and exit on completion. Wait for exit code — do NOT poll, re-run, or check output in a loop.
3. **Expect long waits.** `bootstrap-env.sh` first run: 60–120s (downloads ~8MB caw + ~30MB TSS Node). `caw onboard`: 50–60s. On slow networks or cloud VMs, these may take up to **180s**. Set timeout to **300s** to be safe. These durations are normal — do NOT treat a long wait as a failure.
4. **One command at a time.** Do not start `caw onboard` until `bootstrap-env.sh` has exited. Do not start a transfer until `onboard` has exited.
5. **Report progress to the user BEFORE each long-running command.** The user cannot see command output while it runs. Send a text message to the user before and after every step that takes >5s. Follow this template:
   - Before bootstrap: `"⏳ Downloading wallet tools (caw + TSS Node)... This takes 60–120 seconds on first run."`
   - After bootstrap: `"✅ Tools installed. Now creating your wallet..."`
   - Before onboard: `"⏳ Creating wallet (registering agent → initializing TSS → generating keys)... About 60 seconds."`
   - After onboard: `"✅ Wallet created! Wallet ID: {id}. Checking status..."`
   - After wallet active: `"✅ Your wallet is active and ready to use. Address: {address}"`
   - On error: `"❌ {step} failed: {error}. {hint from stderr if available}"`
   This keeps the user informed during the 2–3 minute setup process instead of leaving them in silence.
6. **On timeout or failure — do NOT retry blindly.** Follow this decision tree:
   ```
   bootstrap-env.sh exited non-zero or timed out?
   ├─ Check: ls -la ~/.cobo-agentic-wallet/bin/caw ~/.cobo-agentic-wallet/cache/tss-node/cobo-tss-node
   ├─ Both exist AND size > 0?  → Bootstrap succeeded. Continue to onboard.
   ├─ One or both missing/empty? → Re-run ONCE: ./scripts/bootstrap-env.sh --env sandbox --force-download
   │   ├─ Second run succeeds?   → Continue to onboard.
   │   └─ Second run fails?      → STOP. Report error + log tail to user. Do NOT retry again.
   └─ Check stderr for "[HINT]" lines — they describe the root cause (DNS, firewall, timeout).
   ```

## Bootstrap

Run the bootstrap script once to install caw and TSS Node:

```bash
./scripts/bootstrap-env.sh --env sandbox
```

caw installs to `~/.cobo-agentic-wallet/bin/caw`; add that dir to PATH.

- **First run**: Downloads caw binary (~8MB) and TSS Node (~30MB) in parallel. Takes 60–120s on fast networks, up to 180s on slow connections. Prints `[3/3] Done` on success.
- **Subsequent runs**: Detects cached binaries, prints `ready`, exits immediately.
- **Run once and wait for exit.** Do NOT re-run, poll, or check status separately. The script handles everything internally.
- **Timeout**: Use `timeout 300s ./scripts/bootstrap-env.sh --env sandbox` or set your exec timeout to 300000ms. The extra margin prevents premature kills on slow cloud networks.
- **Tell the user first.** Before running bootstrap, send: `"⏳ Downloading wallet tools (caw + TSS Node)... This takes 60–120 seconds on first run."` After it exits: `"✅ Tools installed successfully."` or report the error.

**Prompt → action:**

| User prompt | Action |
|-------------|--------|
| `BootStrap a new Cobo Agentic Wallet with token <TOKEN>` | Then `caw onboard --create-wallet --token <TOKEN> --env sandbox` |
| `BootStrap a new Cobo Agentic Wallet with invitation code <CODE>` | Then `caw onboard --create-wallet --invitation-code <CODE> --env sandbox` |
| `BootStrap a new Cobo Agentic Wallet` (no token/code) | Run invite-code flow (see below); once code is available, `caw onboard --create-wallet --invitation-code <CODE> --env sandbox` |

See [Error Handling](./recipes/error-handling.md#onboarding-errors) for bootstrap failures.

## Environment

| Environment | `--env` value | API URL                                          | Web Console |
|-------------|---------------|--------------------------------------------------|-------------|
| Sandbox | `sandbox` | `https://api-agent-wallet-core.sandbox.cobo.com` | https://agenticwallet.sandbox.cobo.com/ |

Set the API URL before any command:

```bash
export AGENT_WALLET_API_URL=https://api-agent-wallet-core.sandbox.cobo.com
```

## Onboarding

All onboard commands take 50–60s. Run once with a **180s timeout** and wait for exit. The command prints progress steps (`[1/5]`…`[5/5]`) to stdout as it runs.

**User communication:** Tell the user what's happening before you run onboard. Example:
> ⏳ Creating your wallet now. This involves 5 steps (agent registration → TSS download → TSS initialization → TSS startup → MPC wallet creation) and takes about 60 seconds. I'll let you know as soon as it's ready.

### Autonomous onboarding (invitation code)

1. After bootstrap-env exits successfully, run:

```bash
export PATH="$HOME/.cobo-agentic-wallet/bin:$PATH"
caw --format table onboard --create-wallet --env sandbox --invitation-code <INVITATION_CODE>
```

~60s: Register → Init TSS → Start TSS → Create wallet. Wallet ready when command exits.

### Invite-code acquisition (when no token/code)

1. Submit waitlist. Get curl from script:

```bash
./scripts/bootstrap-env.sh --env sandbox --print-waitlist-curl
```

Fill in `agent_name`, `agent_description`, `email`, `telegram` and run the printed curl.
2. Ask human to open returned `auth_url` and complete X login.
3. After approval, invite code is sent via X DM.
4. After bootstrap-env exits successfully, run:

```bash
export PATH="$HOME/.cobo-agentic-wallet/bin:$PATH"
caw --format table onboard --create-wallet --env sandbox --invitation-code <INVITATION_CODE>
```

### Supervised onboarding (token provided)

Human initiates from Web Console, provides setup token.

1. After bootstrap-env exits successfully:

```bash
export PATH="$HOME/.cobo-agentic-wallet/bin:$PATH"
caw --format table onboard --create-wallet --env sandbox --token <TOKEN>
```

~60s: Pairing → Init TSS → Start TSS → Create wallet.

Optional post-onboard: `caw profile current` → create address → `onboard self-test` → report summary to user.

---

### Claiming — Transfer Ownership to a Human

When the user wants to claim a wallet (e.g., "我要 claim 这个钱包", "claim the wallet"), use these commands:

```bash
caw profile claim                   # generate a claim link
caw profile claim-info              # check claim status
```

`claim` returns a `claim_link` URL. Share this link with the human — they open it in the Web Console to complete the ownership transfer. Once claimed, the wallet switches to Supervised mode (delegation is created, `--sponsor true` becomes available).

Use `claim-info` to check the current state: `not_found` (no claim initiated), `valid` (pending, waiting for human), `expired`, or `claimed` (transfer complete).

---

### Profile

Each `caw onboard` creates a separate **profile** — an isolated identity with its own credentials, wallet, and TSS Node files. Multiple profiles can coexist on one machine.

- **Default profile**: Most commands automatically use the active profile. Switch it with `caw profile use <agent_id>`.
- **`--profile` flag**: Any command accepts `--profile <agent_id>` to target a specific profile without switching the default. Use this when running multiple agents concurrently.

```bash
# Example: transfer using a non-default profile
caw --profile caw_agent_abc123 tx transfer --to 0x... --token SOLDEV_SOL_USDC --amount 0.0001 --chain SOLDEV_SOL
```

See `caw profile --help` for all profile subcommands (`list`, `current`, `use`, `env`, `archive`, `restore`).

> **ONLY use archive when a previous onboarding has failed and you need to retry.** Do NOT archive before a fresh onboarding — the `onboard` command creates a new profile automatically.

---

## Common Operations

```bash
# Transfer tokens
caw --format json tx transfer --to 0x1234...abcd --token USDC --amount 10 --chain BASE --request-id pay-invoice-1001

# Check wallet balance
caw --format json wallet balance

# List recent transactions
caw --format json tx list --limit 20

# Estimate fee before transfer
caw --format json tx estimate-transfer-fee --to 0x1234...abcd --token USDC --amount 10 --chain BASE

# Contract call
caw --format json tx call --contract 0xContractAddr --calldata 0x... --chain ETH

# Poll a pending approval
caw --format json pending get <operation_id>
```

## Key Notes

- **`--format json`** for programmatic output; `--format table` only when displaying to the user.
- **`--sponsor`**: `true` to have gas fees covered by Cobo Gasless; `false` to pay gas from the wallet's own balance.
- **Gas address** (when not using `--sponsor true`): Keep one fixed address per ecosystem to hold native tokens for fees — one for EVM (ETH), one for Solana (SOL). Before executing any transfer or contract call, check the relevant gas address has sufficient balance:
  ```bash
  caw --format json wallet balance --address <gas-address> --chain <CHAIN>
  ```
  If the balance is low, warn the user and top it up from wherever funds are available before proceeding.
- **No retry loops.** If a `caw` command fails (non-zero exit), read stderr, diagnose the issue, and fix it before running again. Do NOT blindly retry the same command.
- **TSS Node auto-start**: `caw tx transfer` and `caw tx call` automatically check TSS Node status and start it if offline. `caw node stop` checks for pending transactions — use `--force` to skip.
- **wallet_uuid is optional** in most commands — if omitted, the CLI uses the active profile's wallet.
- **StandardResponse format** — API responses are wrapped as `{ success: true, result: <data> }`. Extract from `result` first.
- **Non-zero exit codes** indicate failure — check stdout/stderr before retrying.
- **Show the command**: When reporting `caw` results to the user, always include the full CLI command that was executed, so the user can reproduce or debug independently.
- **Policy denial**: Tell the user what was blocked and why — see [error-handling.md](./recipes/error-handling.md#communicating-denials-to-the-user) for the message template.

## Reference

Read the file that matches the user's task. Do not load files that aren't relevant.

- [Policy Management](./recipes/policy-management.md) — Inspect, test, and troubleshoot policies
- [Error Handling](./recipes/error-handling.md) — Common errors, policy denials, recovery patterns, and user communication
- [Security](./recipes/security.md) — Prompt injection, credential protection, delegation boundaries, incident response

**DeFi recipes** — read the matching file when the user asks about a specific strategy:

| User asks about… | Read |
|---|---|
| Aave, borrow, repay, supply, collateral | [evm-defi-aave.md](./recipes/evm-defi-aave.md) |
| DEX swap, Uniswap, token exchange (EVM) | [evm-defi-dex-swap.md](./recipes/evm-defi-dex-swap.md) |
| DCA, dollar cost average, recurring buy (EVM) | [evm-defi-dca.md](./recipes/evm-defi-dca.md) |
| Grid trading, ladder orders (EVM) | [evm-defi-grid-trading.md](./recipes/evm-defi-grid-trading.md) |
| Solana DEX swap, Jupiter, SOL/USDC | [solana-defi-dex-swap.md](./recipes/solana-defi-dex-swap.md) |
| Solana DCA, recurring SOL purchase | [solana-defi-dca.md](./recipes/solana-defi-dca.md) |
| Solana grid trading | [solana-defi-grid-trading.md](./recipes/solana-defi-grid-trading.md) |
| Prediction market, Drift, Polymarket, long/short | [solana-defi-prediction-market.md](./recipes/solana-defi-prediction-market.md) |
| Policy denial, 403 error, TRANSFER_LIMIT_EXCEEDED | [error-handling.md](./recipes/error-handling.md) |
| Policy setup, dry-run, delegation | [policy-management.md](./recipes/policy-management.md) |
