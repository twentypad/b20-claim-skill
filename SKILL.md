---
name: b20-claim-skill
description: Claim TwentyPad creator or platform swap fees from FeeEscrow 0xD43586103c760Bd5e139a2De2655413dE441B150 on Base. Use when the user wants to claim fees, withdraw hook fees, check owed fees, or claim ETH/USDC earned from TFROG or any TwentyPad B20 launch.
tags: [b20, base, fees, twentypad, claim, feeescrow]
version: 1
visibility: public
metadata:
  clawdbot:
    emoji: "💸"
    homepage: "https://github.com/twentypad/b20-instant-launcher"
---

# TwentyPad B20 Claim Fees

Claim accrued swap fees from the TwentyPad FeeEscrow.

Fees are **not** Uniswap LP fees. The launch hook takes a cut of the quote asset (ETH or USDC) and credits `owed[account][asset]` on this escrow. Creators and the platform pull with `claim` / `claimTo`.

Do not use Clanker / Doppler / Bankr default fee-claim flows.

## Contracts (Base, 8453)

| Name | Address |
| --- | --- |
| FeeEscrow | `0xD43586103c760Bd5e139a2De2655413dE441B150` |
| Launch hook | `0x8c0986c564025903B0f1C7c87cBA1760cB4FAAcc` |
| Factory | `0x15a3f3ABb733868d193b511dd5b91f82ebF888A3` |
| ETH asset | `0x0000000000000000000000000000000000000000` |
| USDC asset | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |

Verified ABI (minimum):

```text
owed(address account, address asset) view returns (uint256)
claim(address asset)
claimTo(address to, address asset)
hook() view returns (address)
```

`asset` is a Uniswap v4 `Currency` = an address. Native ETH is the zero address, not WETH.

## Who can claim

`claim` / `claimTo` send **whatever that caller is owed** for `asset`.

- `msg.sender` must be the credited account.
- Creator fees credit the launch `msg.sender` (factory `tokenCreator[token]`).
- Platform fees credit the platform account configured in the hook (not this skill’s job to change).

If Bankr submits from a wallet that is **not** the credited creator, `owed` will be 0 and the tx is a no-op or wastes gas. Always claim from the creator Bankr wallet that launched the token.

TFROG launch creator (example): `0xd4aedc4595196305a60d8ef2dd9b9ba27021b4cb`

## Parse the request

```
@bankrbot claim my twentypad fees
@bankrbot claim TFROG fees
@bankrbot claim TwentyPad ETH fees
@bankrbot check owed fees for TFROG
@bankrbot claim fees to 0x...
@bankrbot claim USDC fees from FeeEscrow
```

| Intent | Action |
| --- | --- |
| check / how much / owed | read only |
| claim / withdraw / collect | `claim(asset)` to the caller |
| claim to ADDRESS | `claimTo(to, asset)` |

If they name a token, use that token’s launch **quote** as `asset` (ETH or USDC). If they say “all fees”, check both ETH and USDC and claim every asset with `owed > 0`.

## Workflow

### 1. Resolve claimant and assets

- Claimant = Bankr wallet that will sign (`/wallet/me` or the known creator).
- If they named a token, read factory `tokenCreator(token)`. Warn if it is not the signing wallet.
- Assets to check: ETH zero address and USDC, unless they specified one.

### 2. Read balances (required before sending a claim)

```text
owed(claimant, 0x0000000000000000000000000000000000000000)
owed(claimant, 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913)
```

If both are 0, reply “nothing to claim” and stop. Do not send a tx.

Also report escrow native balance only as context. Claimable amount is `owed`, not the contract’s ETH balance (that balance is shared across all creditors).

### 3. Confirm

Reply with:

- claimant address
- each asset and `owed` amount (wei / USDC raw + human)
- destination (`msg.sender` or `claimTo` target)
- reminder: 70% of the 1% hook fee is creator; 30% is platform; anti-snipe surplus goes to platform

### 4. Submit

`value` is always `"0"`. Escrow pays out; the caller does not send ETH.

Claim to self:

```text
to:   0xD43586103c760Bd5e139a2De2655413dE441B150
data: claim(asset)
value: 0
chainId: 8453
```

Selector `claim(address)` = `0x1e83409a`

Claim to another address:

```text
data: claimTo(to, asset)
```

Selector `claimTo(address,address)` = `0x34a1ca89`

If both ETH and USDC are owed, send **two** txs (one per asset).

Wait for confirmation.

### 5. After success

Re-read `owed(claimant, asset)` — should be 0.

Reply with:

- tx hash + Basescan
- asset and amount claimed
- new wallet quote balance
- leftover owed on the other asset if any

## Encoding notes

ETH claim calldata example (`claim(address(0))`):

```
0x1e83409a0000000000000000000000000000000000000000000000000000000000000000
```

USDC claim:

```
0x1e83409a000000000000000000000000833589fcd6edb6e08f4c7c32d4f71b54bda02913
```

Do not pass WETH. This escrow treats native ETH as `address(0)` and uses a raw call to pay it.

`credit` is hook-only (`NotHook` if anyone else calls it). Never call `credit` from this skill.

## Errors

| Case | Action |
| --- | --- |
| `owed == 0` | Do not send tx |
| Wrong wallet | Tell user to use the launch creator Bankr account |
| Token is not TwentyPad | `tokenCreator` is zero — refuse |
| `claimTo` to zero address | refuse |
| User asks to claim “LP fees” | Explain these are hook fees in quote asset |

## What this skill does not do

- Launch tokens (`b20-launcher-skill`)
- Buy/sell (`b20-swap-skill`)
- Change fee split or suffix (owner-only on factory)
- Claim for a third party unless that party’s wallet is the signer, or the signer uses `claimTo` **after** the funds are owed to the signer

## Example commands

```
@bankrbot use b20-claim-skill to check my TwentyPad fees
@bankrbot use b20-claim-skill to claim my ETH fees
@bankrbot use b20-claim-skill to claim TFROG fees
@bankrbot use b20-claim-skill to claim fees to 0xYOUR_ADDRESS
```
