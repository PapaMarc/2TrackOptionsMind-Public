# Algorithms in Strategy Context

## Draft Summary of AlgoD and AlgoG+

> Conceptual reference with inline math and Python expressions.

---

## 1. The Two Algorithms at a Glance

### AlgoD — "Delta-First" Baseline

AlgoD is the original, simpler selector. It does two things:

#### Stage 1: Filter by Delta Band

> "1st stage remains a Delta Band capture: MSFT CC: 0.05–0.15; Wheel: 0.20–0.30 and stuff outside the band is rejected."

**MSFT CC:**

$$0.05 \leq \Delta \leq 0.15$$

```python
in_band_msft = (0.05 <= delta <= 0.15)
```

**Wheel CSP/CC:**

$$0.20 \leq \Delta \leq 0.30$$

```python
in_band_wheel = (0.20 <= delta <= 0.30)
```

This ensures the strike is in the right "risk zone" for the strategy.

#### Stage 2: Pick the Best Premium Within the Band

**Wheel (CSP → CC)**

For Wheel: maximize premium yield and assignment-adjusted return.

Let:

$$Y = \frac{P_{\text{option}}}{\text{Collateral}}$$

$$R_{\text{adj}} = \text{assignment-adjusted return}$$

$$\alpha, \beta > 0 = \text{policy weights}$$

$$S_{\text{AlgoD}}^{\text{Wheel}} = \alpha Y + \beta R_{\text{adj}}$$

```python
premium_yield = option_premium / collateral
score_wheel_algod = alpha * premium_yield + beta * assignment_adj_return
```

**MSFT holdWrite (Never-Assign CC)**

For MSFT CC: maximize premium, but only if theta is above a minimum floor.

Let $\Theta_{\min}$ be the minimum acceptable theta.

Constraint:

$$\Theta \geq \Theta_{\min}$$

Score:

$$S_{\text{AlgoD}}^{\text{MSFT}} = \begin{cases} P_{\text{CC}}, & \text{if } 0.05 \leq \Delta \leq 0.15 \text{ and } \Theta \geq \Theta_{\min} \\ -\infty, & \text{otherwise} \end{cases}$$

```python
if 0.05 <= delta <= 0.15 and theta >= theta_min:
    score_msft_algod = call_premium
else:
    score_msft_algod = float("-inf")
```

> **Interpretation:** AlgoD is simple — stay in the safe delta band, then pick the strike that pays the most while respecting assignment tolerance.

---

### AlgoG+ — "Greek-Weighted" Enhancement

AlgoG+ keeps the same delta bands, but adds a multi-Greek scoring model to choose the best strike inside the band.

> "Greek-weighted AlgoGplus candidate scoring in Wheel and MSFT selectors with persisted score-breakdown metadata."
> "Greek scoring then applies a scoring matrix."

It uses:

| Greek / Factor             | Role                              |
| -------------------------- | --------------------------------- |
| **Theta**                  | Premium decay                     |
| **Theta efficiency**       | Theta per dollar of collateral    |
| **Vega**                   | Volatility sensitivity            |
| **Gamma**                  | Delta instability                 |
| **Charm**                  | Delta decay over time             |
| **Vanna / Vomma**          | Volatility/delta curvature        |
| **IV Percentile**          | Relative implied volatility level |
| **Liquidity**              | Bid/ask spread quality            |
| **Event-risk suppressors** | Earnings, macro events            |

#### Generic AlgoG+ Score Form

For any candidate, AlgoG+ computes a weighted linear score:

$$S_{\text{AlgoG+}} = \sum_{i} w_i \cdot g_i$$

Where $g_i$ are normalized Greek-derived features.

```python
score_algog = sum(w[i] * g[i] for i in features)
```

---

## 2. How the Algorithms Map to the Two Strategies

### A. Wheel Strategy (CSP → CC)

**Goal:** Maximize premium while accepting controlled assignment risk.

#### AlgoD for Wheel

- Accept only strikes with $0.20 \leq \Delta \leq 0.30$
- Rank by premium yield and assignment-adjusted return

$$S_{\text{AlgoD}}^{\text{Wheel}} = \alpha Y + \beta R_{\text{adj}}$$

```python
score_wheel_algod = alpha * premium_yield + beta * assignment_adj_return
```

> **Simple English:** Pick the strike that pays the most while keeping assignment probability reasonable.

#### AlgoG+ for Wheel

AlgoG+ adds a full scoring model with the following signal directions:

| Factor           | Direction     | Reason                                    |
| ---------------- | ------------- | ----------------------------------------- |
| Theta            | ↑             | More decay = more income                  |
| Theta efficiency | ↑             | More decay per dollar of collateral       |
| Vega             | ↑             | Richer premiums in high IV                |
| Gamma            | moderate      | Predictable assignment behavior           |
| IV Percentile    | ↑             | Sell when IV is elevated                  |
| Vanna / Vomma    | ↑             | Smoother behavior under volatility shifts |
| Delta            | penalized (↓) | Avoid too-high assignment risk            |
| Liquidity        | ↑             | Tighter spreads, safer fills              |

**Formal AlgoG+ Wheel Score**

Let:

- $\Theta$ = theta
- $\Theta_{\text{eff}} = \dfrac{\Theta}{\text{Strike} \cdot 100}$
- $\nu$ = vega
- $\Gamma$ = gamma
- $\text{IVP}$ = IV percentile
- $\text{Vanna}, \text{Vomma}$ = cross-Greeks
- $\text{Liq}$ = liquidity score
- $\text{DeltaRisk} = |\Delta|$ or policy transform

$$S_{\text{AlgoG+}}^{\text{Wheel}} = w_\Theta \Theta + w_{\Theta_{\text{Eff}}} \Theta_{\text{eff}} + w_\nu \nu + w_{\text{IVP}} \text{IVP} + w_\Gamma \Gamma + w_{\text{Vanna}} \text{Vanna} + w_{\text{Vomma}} \text{Vomma} + w_{\text{Liq}} \text{Liq} - w_\Delta \text{DeltaRisk}$$

```python
theta_eff = theta / (strike * 100)

score_wheel_algog = (
    w_theta * theta +
    w_theta_eff * theta_eff +
    w_vega * vega +
    w_ivp * iv_percentile +
    w_gamma * gamma +
    w_vanna * vanna +
    w_vomma * vomma +
    w_liq * liquidity -
    w_delta * abs(delta)
)
```

> **Simple English:** Inside the 0.20–0.30 delta band, AlgoG+ picks the strike that gives the best combination of premium, volatility advantage, and predictable assignment behavior.

---

### B. MSFT holdWrite (Never-Assign Covered Calls)

**Goal:** Generate income without ever letting shares get called away.

#### AlgoD for MSFT

- Accept only $0.05 \leq \Delta \leq 0.15$
- Require $\Theta \geq \Theta_{\min}$

$$S_{\text{AlgoD}}^{\text{MSFT}} = P_{\text{CC}}$$

```python
score_msft_algod = call_premium if (0.05 <= delta <= 0.15 and theta >= theta_min) else -inf
```

> **Simple English:** Sell only very safe calls, and only when the premium is worth it.

#### AlgoG+ for MSFT

AlgoG+ adds assignment-risk suppression:

| Factor        | Direction | Reason                                                 |
| ------------- | --------- | ------------------------------------------------------ |
| Charm         | ↑         | Stable delta path over time                            |
| Gamma         | ↓         | Avoid sudden delta spikes                              |
| Vanna         | ↓         | Avoid volatility-driven delta jumps                    |
| Vega          | ↓         | Avoid selling calls before events or high-IV reversals |
| IV Percentile | ↓         | Avoid selling when IV is too low                       |
| Event-risk    | ↓         | Earnings, macro events                                 |
| Liquidity     | ↑         | Safer fills                                            |

**Formal AlgoG+ MSFT Score**

Let $\text{Charm}$, $\Gamma$, $\text{Vanna}$, $\text{Vega}$, $\text{IVP}$, $\text{EventRisk}$, $\text{Liq}$ as defined above.

$$S_{\text{AlgoG+}}^{\text{MSFT}} = w_\Theta \Theta + w_{\text{Charm}} \text{Charm} - w_\Gamma \Gamma - w_{\text{Vanna}} \text{Vanna} - w_{\text{Vega}} \text{Vega} - w_{\text{IVP}} \text{IVP} - w_{\text{Event}} \text{EventRisk} + w_{\text{Liq}} \text{Liq}$$

```python
score_msft_algog = (
    w_theta * theta +
    w_charm * charm -
    w_gamma * gamma -
    w_vanna * vanna -
    w_vega * vega -
    w_ivp * iv_percentile -
    w_event * event_risk +
    w_liq * liquidity
)
```

> **Simple English:** Inside the 0.05–0.15 delta band, AlgoG+ picks the strike that is least likely to drift ITM, even if the stock moves or volatility changes.

---

## 3. How They Balance Assignment Risk vs. Premium Optimization

### Wheel

Assignment is acceptable, so the algorithm balances:

- **Premium maximization** — theta, vega, IVP
- **Predictable assignment behavior** — gamma, delta penalty

AlgoD does this crudely (premium vs. delta).  
AlgoG+ does it precisely (premium vs. multi-Greek stability).

> **Plain English:** Wheel wants the best rent for the least drama.

### MSFT holdWrite

Assignment is **NOT** acceptable, so the algorithm prioritizes:

- **Delta stability** — charm, gamma, vanna
- **Volatility suppression** — vega, event-risk
- **Premium only if safe** — theta floor

> **Plain English:** MSFT wants safe rent without ever risking losing the house.

---

## 4. The Core Philosophical Difference

|                        | AlgoD                          | AlgoG+                                             |
| ---------------------- | ------------------------------ | -------------------------------------------------- |
| **Principle**          | Delta defines the strike       | Delta defines the region; Greeks define the winner |
| **Selection method**   | Best premium inside delta band | Best Greek-weighted score inside delta band        |
| **Complexity**         | Simple 2-factor score          | Multi-Greek weighted linear score                  |
| **Surprise tolerance** | Higher                         | Lower                                              |

**AlgoD** = _"Delta defines the strike."_  
Pick a delta band, then choose the best premium inside it.

**AlgoG+** = _"Delta defines the region; Greeks define the winner."_  
Use delta to set the safe zone, then use Greeks to choose the strike that is:

- most stable
- most predictable
- most profitable
- least likely to surprise you
