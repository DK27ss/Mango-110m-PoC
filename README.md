# Mango-110m-PoC

>**Date of attack:** October 11, 2022, 22:08 → 22:53 UTC
>
>**Protocol:** Mango Markets on Solana (program `mv3ekLzLbnVPNxjSKvqBpU3ZeZXPQdEC3bp5MDEBG68`)
>
>**Attacker:** Avraham Eisenberg (publicly self-identified)
>
>**Loss:** ~$117M of user funds drained, ~$115M of bad debt left in the protocol
>
>**Class:** Oracle price manipulation + leveraged perpetual abuse — an *economic* exploit, not a code bug
>

---

## summary

Mango Markets was a Solana-based decentralized exchange offering spot
trading, lending and perpetual futures, the attacker exploited the fact
that the MNGO oracle price feed used in Mango risk engine was effectively
a thin volume-weighted average over Serum, Raydium and Jupiter spot
markets, none of which had deep liquidity for MNGO/USDC.

full chain of actions, all visible on-chain:

1. Funded two wallets with **$10M USDC** from FTX
2. Created two MangoAccounts and deposited 5M USDC into each as collateral
3. Used MangoAccount1 to open a **massive LONG** on MNGO-PERP
   (515,717,245 MNGO @ $0.0382)
4. Used MangoAccount2 to open matching **SHORTs** filling the LONG
   (488,302,109 MNGO @ $0.0382), forming a wash trade
5. Bought MNGO on Serum/Raydium/Jupiter with the unused balance, pushing
   the spot price (and therefore the oracle) **from $0.0382 to ~$0.91**
   (a 24× pump) using less than ~$1.5M
6. The pumped oracle inflated MangoAccount1 unrealized PnL to
   ~$420M+, which Mango risk engine accepted as collateral
7. Borrowed and withdrew **~$117M of real assets** (BTC, USDT, SOL,
   mSOL, USDC, SRM, MNGO) from Mango lending pools
8. Pulled liquidity, oracle collapsed, bad debt of ~$115M was left
   stranded in the protocol

after the drain, Eisenberg drafted a Mango DAO governance proposal
("Repay bad debt") asking the treasury to write off the bad debt
in exchange for him returning a portion of the funds, and voted on
the proposal using the 33M+ MNGO he had just acquired.

---

## Protocol context

Mango Markets had two key design choices that, combined, made the
exploit possible:

## Cross-margined perp accounts

A `MangoAccount` was a single risk container holding

- per-token **deposits** and **borrows** (16 token slots)
- per-market **perpetual positions** (15 perp markets)
- **spot open orders** on Serum

A user borrowing capacity was governed by **cross-margin health**

```
init_health =
    Σ_tokens   ( deposit[i] · deposit_idx[i] · price[i] · init_asset_w[i]
               − borrow[i]  · borrow_idx[i]  · price[i] · init_liab_w[i] )
    + Σ_perps  ( base_position · base_lot_size · price[i] · weight
               + quote_position )
```

A LONG perp mark-to-market value contributed to health *immediately*,
weighted by `init_asset_weight` (0.75 for MNGO-PERP), there was no
"realized vs unrealized" distinction in collateral computation: a
favorable oracle move could turn a $5M deposit into hundreds of
millions of borrowing capacity in seconds.

## oracle for MNGO

MNGO/USD price was sourced from a Pyth aggregator
(`49cnp1ejyvQi3CJw3kKXNCDGnNbWDuZd3UG3Y2zGvQkX`) that tracked the
Serum MNGO/USDC orderbook plus Raydium/Jupiter pools, none of these
venues had more than a few hundred thousand USD of two-sided liquidity
for MNGO at the time.

- no minimum aggregator volume threshold
- no time-weighted average (TWAP) on a long window
- no deviation circuit breaker
- no hard cap on per-asset borrowing

Together, these meant a relatively small market buy could cause a
disproportionately large oracle move that the perp risk engine
trusted blindly.

---

## setup phase (22:07–22:23 UTC)

| Slot          | UTC      | Event                                            |
|---------------|----------|--------------------------------------------------|
| —             | 19:36–19:54 | Account1 (`yUJw9a2P…NqM`) and Account2 (`J44uRJxJ…dJU`) funded with **$10.5M USDC + 2 SOL** from FTX |
| 154,861,607   | 22:07:26 | MangoAccount2 (`CQvKSNnY…KFfX`) created, 1 USDC deposit |
| 154,861,680   | 22:08:07 | MangoAccount1 (`4ND8FVPj…RnjNa`) created, 100 USDC deposit |
| 154,862,798   | 22:18:57 | MangoAccount1 deposit topped up to **5,000,000 USDC** |
| 154,862,829   | 22:19:13 | MangoAccount2 deposit topped up to **4,999,998.95 USDC** |

at this point both accounts hold $5M of USDC collateral, no positions.
Init health for each account ≈ $5M USDC; each can open up to roughly
~$25M of perp notional at the MNGO-PERP `init_asset_weight` of 0.75
(implied leverage 4× when seen as collateral, but Mango's perp
init/liab weights effectively allowed up to 5–7× with this asset).

---

## wash-trade phase (22:23–22:25 UTC)

// LONG order from MangoAccount1

Slot **154,863,297** (22:23:40 UTC) — signature
`2xPS2ZefQ3QvvMpQ1FLLTdAwWARZ42…`

Mango program log

```
Mango: PlacePerpOrder2 client_order_id=1665526968618
bid on book order_id=7065102980230740612530 quantity=515717245 price=382
```

Decoded:

- `quantity = 515,717,245` lots (MNGO-PERP `base_lot_size = 1,000,000` →
  one lot = 1 MNGO with 6 decimals)
- `price = 382` lots (`quote_lot_size = 100`, `base_lot = 1,000,000`
  → ui_price = 382 × 100 / 1,000,000 = **$0.0382 USDC per MNGO**)

notional: 515.7M MNGO × $0.0382 = **~$19.7M LONG**, posted as a resting
**bid** at the current spot price.

// SHORTs from MangoAccount2

Slots 154,863,408 / 154,863,494 / 154,863,524 (22:24:47 → 22:25:51 UTC)
— three `Mango: PlacePerpOrder2` instructions sized so they together
match the LONG bid sitting on the book:

- 261,780,104 + 222,688,514 + 3,833,491 = **488,302,109 MNGO short**

these are aggressively-priced sell orders that immediately match against
MangoAccount1 LONG bid, forming the actual wash trade, both accounts
end up with opposed perp positions at the same entry price ($0.0382),
fully offsetting at the protocol level.

## why two accounts ?

the LONG quantity Eisenberg wanted to acquire was **far** larger than
any organic counterparty would have provided at $0.0382, by being the
counterparty to himself, he

- guaranteed deterministic execution at his entry price
- avoided any slippage on the open
- used his own capital to capitalize both sides — the SHORT account is
  essentially a hedge that protects part of his attack cost (Section 8)

---

## oracle pump (22:26–22:34 UTC)

with the LONG position open and marked at $0.0382, Eisenberg now had to
make Mango risk engine *believe* MNGO had appreciated.

oracle was driven by the spot MNGO/USDC marketplaces, he attacked
all of them in parallel through Jupiter routing.

## First pump (slot 154,863,595, 22:26:30 UTC)

Sig `5o2wku1u3FYwjbAvVuCyWcN6rknLZS…`, Jupiter `Route` instruction.
Decoded Raydium `ray_log` (1 + 7×u64 LE):

- `amount_in  = 80,000,000,000` → **80,000 USDC** in
- `amount_out = 591,163` MNGO out
- effective price ≈ **$0.135 / MNGO** (from $0.0382, +250%)

followed in rapid succession by a chain of Raydium/Jupiter swaps, from
the on-chain logs

| Slot         | USDC in | MNGO out | Effective px |
|--------------|--------:|---------:|-------------:|
| 154,863,595  |  80,000 |  591,163 |       0.1353 |
| 154,864,283  |  50,000 |  210,546 |       0.2375 |
| 154,864,342  |  30,000 |   84,001 |       0.3571 |
| 154,864,405  |  35,000 |   70,585 |       0.4959 |
| 154,864,657  |  45,000 |  107,886 |       0.4171 |
| 154,864,800  |  42,500 |  112,509 |       0.3777 |
| 154,864,863  |  45,000 |  139,441 |       0.3227 |
| 154,865,007  |  70,000 |  144,948 |       0.4829 |

cumulatively about **$400K of USDC** pushed the average oracle price
through $0.50 and ultimately spiked it to **~$0.91** (post-mortem and
later reconstructions), aach swap drained one pool, then Jupiter
re-routed through another, so the oracle aggregator was never able to
"average out" the manipulation.

## Why this works at all

Pyth MNGO publisher set was small and mechanically reflected the
on-chain spot price, Pyth confidence intervals widened but did not
trigger any halt in Mango. Mango v3 read the price unconditionally
with a `valid_interval` of a handful of slots, after ~3 minutes of
sustained manipulation, the on-chain oracle truly believed MNGO was
at >$0.90.

---

## Settle PnL (slot 154,863,886, 22:29:27 UTC)

Sig `SFVdKJpzm2hsK5Kq9LDBD47ZXQu85s…`, three `Mango: SettlePnl`
instructions in a single transaction, settles the unrealized perp PnL
between MangoAccount1 (huge winner) and three counterparties, primarily
MangoAccount2.

after settle, MangoAccount1 quote balance reflects hundreds of millions
of *unrealized* USDC gain marked at the manipulated oracle price, this
is when the cross-margined health collateral becomes spendable.

## First drain (slot 154,863,916, 22:29:44 UTC)

Withdraw `+50,000,000` USDC.

this single withdrawal already exceeds the seed capital (10× the deposit).
Eisenberg keeps going.

## Continued drain (22:36–22:45 UTC)

| Slot         | UTC      | Withdraw                    |
|--------------|----------|-----------------------------|
| 154,864,620  | 22:36:34 | wSOL (≈400,000)             |
| 154,864,714  | 22:37:27 | wSOL (≈361,577)             |
| 154,864,734  | 22:37:38 | mSOL **798,000**            |
| 154,864,760  | 22:37:53 | BTC (sollet) **281.12**     |
| 154,865,091  | 22:41:12 | USDC **2,807,721**          |
| 154,865,125  | 22:41:30 | USDT **3,266,426**          |
| 154,865,481  | 22:45:01 | SRM **2,354,260**           |
| 154,865,505  | 22:45:16 | MNGO **32,409,565**         |

the MNGO withdrawal, 32.4M tokens, is later used to vote on the
"Repay bad debt" proposal.

---

## Economic model — how $5M became $365M of borrow capacity

the core math, applied to the on-chain numbers and reproduced exactly

```
LONG_POSITION  = 515,717,245 MNGO
ENTRY_PRICE    = 0.0382 USDC/MNGO
USDC_DEPOSIT   = 5,000,000 USDC
INIT_ASSET_W   = 0.75    (MNGO-PERP)

uPnL(p)        = (p − 0.0382) · 515,717,245
init_health(p) = USDC_DEPOSIT + uPnL(p) · INIT_ASSET_W
```

| oracle px  | uPnL (USD)        | init health (USD) | × seed |
|-----------:|------------------:|------------------:|------:|
| 0.0382     |               +0  |        5,000,000  |  1.0× |
| 0.10       |       +31,871,326 |       30,497,061  |  6.1× |
| 0.30       |      +135,014,789 |      113,011,091  | 22.6× |
| 0.50       |      +237,642,584 |      195,114,067  | 39.0× |
| **0.91**   |  **+449,602,294** | **+364,681,835**  | **73×** |

At the oracle peak, the protocol was willing to lend out the equivalent
of **~73× the attacker's seed capital** against a single perp position
in an asset with daily volume comfortably below $1M.

The *cost* of moving the oracle from $0.0382 to $0.91 was on the order
of $1–2M of MNGO buys (most of the cost was actually recovered via the
SHORT account, see Section 8). The drained $117M dwarfs that cost by
two orders of magnitude.

---

## The hedge - MangoAccount2 was a feature, not just a counterparty

a subtle point that the post-mortem touched only in passing: Account2
SHORT position naturally **profits** when MNGO crashes back to its real
value. Reproducing in `06_cheatcode_exploit.py`

- Account2 short: 488,302,109 MNGO @ entry $0.0382
- After drain, oracle = $0.91 → Account2 equity = **−$420M** (paper loss)
- Post-crash, oracle = $0.025 → Account2 equity = **+$11,445,588 USDC**

So the crash that creates the bad debt on Account1 simultaneously
generates ~$6.4M of profit on Account2, Account2 collateral is fully
preserved and a small bonus is earned, the attacker recovers from
Account2 *some of the cost of pumping MNGO*.

this is why Eisenberg used **two** accounts rather than one: it's a
self-hedged manipulation, the economic loss on the SHORT side at the
peak is offset by the gain on the LONG side on the way down, both
sides reverse but only the LONG carries the bad-debt liability, and
that liability stays with the protocol, not with him.

---

## Aftermath — the governance troll

about an hour after the drain, Eisenberg created a Mango DAO governance
proposal titled *"Repay bad debt"* offering to return part of the
funds (the MSOL, SOL, and MNGO portions of the haul) in exchange for:

- the Mango treasury covering the remaining bad debt
- all token holders waiving any civil claims against the attacker
- no criminal investigations or fund freezing

the proposal received over **33M MNGO Yes votes**, voted with tokens
he had just withdrawn from the exploit itself. The DAO ultimately
agreed to a modified version, recovering a portion of the assets.

---

## Why this is *not* a code bug

there is no buffer overflow, no missing signer check, no reentrancy.
Every instruction Eisenberg sent was a normal Mango v3 instruction
that any user could call

- `PlacePerpOrder2` — well-formed perp order
- Raydium/Jupiter swaps — normal SPL token swaps
- `SettlePnl` — standard PnL settlement with a willing counterparty
  (himself)
- `Withdraw` — standard withdrawal, blocked only by `init_health < 0`,
  which it never was at the time of the calls

every transaction succeeded at the program-logic level, the protocol
*as written* paid out $117M because its own configured parameters said
the borrower had $360M+ of collateral.

the vulnerability is in the **economic design**: trusting an oracle
backed by thin liquidity, accepting unrealized perp PnL at an
init_asset_weight of 0.75 against unlimited borrowing of any other
listed asset, and not capping per-asset borrows by oracle depth.

---

```
Attacker bottom line:
  • Total seed (from FTX)       :  10,000,000 USD
  • Recovered (drain MA1 + MA2) : 120,108,629 USD
  • Net profit                  : 110,108,629 USD

Mango protocol bottom line:
  • Resulting bad debt          : 110,470,509 USD

Post-mortem cross-check:
  • Post-mortem  : drain ≈ $120M, bad debt ≈ $115M, attacker ≈ +$110M net
  • Our PoC      : drain ≈ $109M, bad debt ≈ $110M
```

---

## References

- On-chain transactions:
  - Account1 setup: `yUJw9a2PyoqKkH47i4yEGf4WXomSHMiK7Lp29Xs2NqM`
  - Account2 setup: `J44uRJxJEDTyPgFG6BtQJ3skhex8FeBbDJWVoh3z9dJU`
  - MangoAccount1: `4ND8FVPjUGGjx9VuGFuJefDWpg3THb58c277hbVRnjNa`
  - MangoAccount2: `CQvKSNnYtPTZfQRQ5jkHq8q2swJyRsdQLcFcj3EmKFfX`
  - First long-order tx: `2xPS2ZefQ3QvvMpQ1FLLTdAwWARZ42BH4VZXJke…`
  - First settle tx: `SFVdKJpzm2hsK5Kq9LDBD47ZXQu85s5UffCYUrzs…`
- Mango v3 program: `mv3ekLzLbnVPNxjSKvqBpU3ZeZXPQdEC3bp5MDEBG68`
- MangoGroup: `98pjRuQjK3qA6gXts96PqZT4Ze5QmnCmt3QYjhbUSPue`
- MangoCache: `EBDRoayCDDUvDgCimta45ajQeXbexv7aKqJubruqpyvu`
- Pyth MNGO/USD oracle: `49cnp1ejyvQi3CJw3kKXNCDGnNbWDuZd3UG3Y2zGvQkX`
- Mango v3 source code: <https://github.com/blockworks-foundation/mango-v3>
- Eisenberg DOJ indictment (S.D.N.Y., 2022-12-26):
  *USA v. Eisenberg*, 22-mj-10337

---

>
>Company : https://blockraider.xyz/
>
>Community : https://discord.gg/Vqqt7jyRr7
>
>Disclosure : https://t.me/blockraider_alerts_bot

<img width="161" height="51" alt="blockraider" src="https://github.com/user-attachments/assets/dc86222a-b5fb-49e8-9a9d-350973b7521d" />
