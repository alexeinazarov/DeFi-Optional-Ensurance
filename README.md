# DeFi Optional-Ensurance
**Proposal:**  
A mathematically capped loan-to-value ratio — even in the event of an instant ETH price collapse to \$0 before expiry.  
Built on the counter-cyclical nature of put options: as the underlying price falls, the right to sell it at a fixed lower strike gains value.
> **DISCLAIMER**  
> All numerical values of Optional-Ensurance parameters below are illustrative only.  
> Final parameters are subject to rigorous testing, model validation, and ongoing calibration.
#### At today’s implied volatility levels, the indicative cost to ensure **1 ETH** of collateral is approximately **\$67.25 per month**.


### 0. Options 101
An [**option**](https://www.investopedia.com/terms/o/option.asp) is a contract that gives its buyer the **right—but not the obligation**—to buy (*call*) or sell (*put*) an asset at a fixed **strike price** before or at a set expiry date.  

Most crypto options are **European-style**, meaning they **auto-exercise at expiry** if in-the-money.  
On venues like [**Deribit**](https://www.deribit.com/) and [**Lyra**](https://app.lyra.finance/), pricing is sought to follow the [**Black-Scholes model**](https://support.deribit.com/hc/en-us/articles/25944688327069-Inverse-Options), possibly adjusted for [jumps](https://www.fh-vie.ac.at/uploads/WP-081_2013.pdf) or [volatility smiles](https://www.investopedia.com/ask/answers/012015/what-volatility-smile.asp).

The *time-value decay* of a long put position is measured by **θ (theta)**.  Higher implied volatility (**σ**) ⇒ higher θ ⇒ dearer daily *“bleed”* (loss in value).

###  1. Liquidity & availability disclaimer  *(figures as of 20 Jun 2025)*  

| Item | Latest datapoint | Source |
|------|------------------|--------|
| **Total ETH options open-interest (all expiries)** | **\$ 6.8 B** | [Deribit Metrics → ETH OI](https://insights.deribit.com/options-dashboard) |
| **Share in sub-14-day expiries** | **≈ 8 %** (≈ \$ 540 M) | *“Open Interest by Expiry”* panel on same page  |
| **Largest single weekly expiry in June-2025** | **\$ 1.64 B** notional set to expire on 27 Jun 2025 | [FXStreet report](https://www.fxstreet.com/cryptocurrencies/news/ethereum-price-forecast-eth-tackles-2-750-wall-amid-16-billion-options-expiry-on-deribit-202505292130) |
| **Deribit share of global ETH OI** | **≈ 85 %** | [CoinDesk Daybook](https://www.coindesk.com/daybook-us/2025/01/07/crypto-daybook-americas-spxs-cautionary-signal-for-btc)  |

The **two-layer hedge** described below requires at most **\$10 M notional per strike**, which is:

* **< 0.2 %** of the short-dated pool (≤ 14 days)
* **< 0.15 %** of total ETH open interest

Execution at this size is therefore highly unlikely to move markets or drain liquidity.

> At current spot (**\~\$2,555/ETH**), and using the 1.25 ETH notional per loan described below, this translates to capacity for hedging **\~2,500 ETH of loan exposure**.

### 2 · Two-layer “optional-ensurance” structure

| Layer           | Strike (× spot) | Tenor | Notional *(per 0.8-ETH loan)* | Purpose                           |
| --------------- | --------------- | ----- | ----------------------------- | --------------------------------- |
| **Core hedge**  | **0.80 × S**    | 14 d  | **1.00 ETH**                  | Cushion for multi-day bear legs   |
| **Gamma patch** | **0.90 × S**    | 7 d   | **0.25 ETH**                  | First-aid for a quick 10–15 % dip |

----


### 3. Comparing Optional-Ensurance to LLAMMA: What each design is trying to guarantee  

| Mechanism                   | How downside is capped (human-speak)                                                                                                                                           | Can cosequences be *un-done* if price bounces? |
|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| **Optional-Ensurance** (OE) | **Step 1 – Insurance**: the protocol buys puts struck at **80 % of today’s spot** for each borrower.<br> **Step 2 – Top-up**: if the live loan-to-value (LTV) creeps **above 80 %** the keeper **sells part of those puts and buys ETH**, restoring LTV ≤ 78 %. | **No.** Once the keeper has sold the put and used the cash to buy ETH, the upside is gone; if price rebounds the put has already been spent. |
| **Curve LLAMMA**            | ETH is deposited across 400 micro-bands under the spot price. Each tiny price drop swaps a sliver of ETH → crvUSD ([“soft liquidation”](https://resources.curve.finance/crvusd/loan-concepts/#soft-and-de-liquidation)); every uptick swaps it back (“deliquidation”). | **Yes.** ETH is [repurchased automatically](https://resources.curve.finance/crvusd/loan-concepts) on the way back up. |

---
### 4. Deterministic maths ready for audit  
#### DeFi Optional-Ensurance (OE) formal model specification

Let:

- $` C_0 `$ = initial collateral in ETH
- $` C_t `$  collateral (in ETH) held at time $`t`$ 
- $` S_t `$ = ETH spot price at time $`t`$  
- $` P_i(S_t, T_i) `$ = price of the i-th put option at time $`t`$  
- $` n_i `$ = notional (in ETH) of i-th put
- $`V_t = \sum_i n_i P_i(S_t,T_i)`$ = USD value of remaining puts
- $` L `$ = debt principal (fixed at origination, typically $` L = 0.8 \times S_0 `$)
- $` A_0 = C_0\,S_0 + V_0 `$
- $` \mathrm{LTV}_t `$ = Loan-to-value at $`t`$ 
- $\beta = 1+\sum_i \Delta_i$ (portfolio delta) is recalculated each night and after every keeper trade.
- $x_{t} = 1 - \frac{S_t}{S_0}$
```math
\boxed{\mathrm{LTV}_t = \frac{L}{C_t\,S_t + V_t}}
```

#### Keeper intervention logic

```math
\textbf{Trigger:}
\quad
\textbf{IF } \mathrm{LTV}_t > 0.80
```

```math
\textbf{Action:}
\quad
\text{Sell puts in order of nearest expiry}
\quad \Rightarrow \text{obtain Δ\$}
```

```math
\textbf{Rebalance:}
\quad
\text{Use proceeds to market-buy ETH}
\quad \Rightarrow \text{update } C_t
```

```math
\textbf{Stop condition:}
\quad
\text{When }  \mathrm{LTV}_t \le 0.78
```

All steps are gas-efficient and run on existing keeper infra.


### 5. Example for Today’s headline numbers (ETH ≈ \$2,421.47, 20 Jun 2025 at 2035 EDT, 4% drop in 3 hours)
#### Optional-Ensurance during crash
| Quantity                                                    | Symbol / formula                                                  | Value |
|-------------------------------------------------------------|-------------------------------------------------------------------|-------|
| Up-front option cost                                        | $` X_0 = \sum_i n_i P_i `$                                            |Theoretically, using [Black-Scholes formula](https://support.deribit.com/hc/en-us/articles/25944688327069-Inverse-Options) **\$ 16.36** per 0.8-ETH loan *(1 × 14-d 0.8 × S put + 0.25 × 7-d 0.9 × S put, σ₇d ≈ 79 %)*. <br>In practice ([derive.xyz](https://www.derive.xyz/options/eth?expiry=20250704), at 2035 EDT ) **$22.0** per 0.8-ETH loan *(1 × 14-d put K ≈ $2 000, mid-price $17.1 + 0.25 × 7-d put K ≈ $2 200, mid-price $19.6).*|
| PV-adjusted premium (funds full 14-day hedge)¹ | $X_\text{PV} = X_0 + 0.25\,P_{7d}(0.9S_0)\,e^{-r\,7/365}$ | **\$22.0 + 4.85 = \$26.9** |
| Daily theta bleed – informational only                                          | $` Y = -\frac{1}{365}\sum_i n_i \theta_i `$                         |Theoretically, using [Black-Scholes formula](https://support.deribit.com/hc/en-us/articles/25944688327069-Inverse-Options)  \$ 2.73 ≈ 0.141 %/day ≈ **51.5% p.a.**. <br>In practice ([derive.xyz](https://www.derive.xyz/options/eth?expiry=20250704), at 2035 EDT ) **$3.5 ≈ 0.17 % of loan per day ≈ 62.1 % p.a.** *(θ per contract: −$2.37 and −$4.60 ⇒ −$2.37 × 1 + −$4.60 × 0.25)*|
| One-block crash tolerated (with +0.15 ETH extra collateral — ***to be adjusted later***) before rebalance | $` \displaystyle\Delta S^{\star} = -\frac{A_0 - L/0.8}{\beta S_0} `$ | **–14 %** |
| After rebalance worst-case LTV                             | $\displaystyle\max_{x_t\ge1}\mathrm{LTV}(x_t)=\frac{0.80}{1.025}\approx0.781$  | **78 %** worst-case, even if ETH → $0$ before expiry |

<sup>¹ Second 0.25-lot 7-day put must be re-bought on day 7; present-value at 5 % r ≈ \$ 4.85.</sup>

#### Comparing to LLAMMA crash math (λ = 9 %, 80 % start-LTV)
*Hard liquidation fires when loan-health = 0*  
λ = 0.09 (loan\_discount).
→ effective  $` \text{max LTV} = \frac{1}{(1-λ)}=\frac{1}{0.91} ≈109.9 \% `$.  
With $A = 100$ and $N \approx 400$, [loan health](https://resources.curve.finance/crvusd/loan-concepts/#loan-health) reaches zero after approximately **65 to 80** bands are crossed — corresponding to a **32–38 %** price decline — at which point **hard liquidation** is triggered.

