# 🔋 BESS Arbitrage Strategy Simulation (GB, Elexon)

This project simulates and benchmarks trading strategies for a utility-scale **Battery Energy Storage System (BESS)** in Great Britain. It pulls real **Elexon System Price** data, builds a simple **Day-Ahead (DA) proxy** price, and runs four strategy “sessions”:

- **A**: rule-based physics simulation (baseline)
- **B**: **linear program** with perfect foresight on DA
- **C**: **DA + BM** co-allocation heuristic with physical safety
- **D**: **CVaR** risk-aware optimisation over **Monte Carlo** price scenarios

The goal is to find **profitable** and **physically realistic** schedules and to show the trade-off between raw profit and downside risk.

---

## Quick view

- **Data**: Elexon System Price, half-hourly, **48** settlement periods per UTC day  
- **Battery**: **50 MW / 100 MWh**, round-trip efficiency about **90%** (`η_ch = η_dis = 0.95`)  
- **Costs**: simple throughput degradation, default **£6/MWh** (tune it)  
- **Outputs**: CSVs, PNG plots, JSON summary (ready for a CV / portfolio)

**Example run (ninety days):**

| Strategy                       | Profit (90 days) | Notes                                                                 |
|-------------------------------|------------------|-----------------------------------------------------------------------|
| A — simple heuristic           | **£84,843**      | p30/p70 triggers, good baseline                                       |
| B — DA perfect-foresight LP    | **£283,253**     | upper bound for DA under constraints, terminal SoC enforced           |
| C — DA+BM co-allocation        | **£269,854**     | zero overlap, total cap respected; close to PF but a bit lower        |
| D — CVaR (scenarios = 150)     | **£38,687** mean | conservative schedule with λ = 0.5 and overlap penalty                |

> Numbers depend on the window, the fee, and solver tolerances. Use the repo to reproduce.

---

## Why this matters

Power prices move a lot during the day. A battery can **charge when cheap** and **discharge when expensive**. Doing this well needs:

- **battery physics** (power, energy, efficiency, SoC limits),
- **market logic** (DA vs BM),
- **good optimisation** (LP),
- **risk control** (CVaR).

This repo shows all of these in a clean, single file.

---

## Market and data (one page)

- **Settlement Periods (SPs)**: **48** half-hours per UTC day (SP=1 at 00:00, SP=48 at 23:30).
- **System Price (BM / imbalance)**: out-turn balancing price; volatile and can be negative.
- **DA proxy** (built from System Price):  
  Rolling **median** over six half-hours, then an **EMA** with span “twelve”, then clip to a DA-like range. In short:
  \[
  \text{DA}_t = \operatorname{clip}\big(\text{EMA}_{\text{span=twelve}}(\operatorname{Med}_{6}(\text{SP}_t)),\,[-20,\,400]\big)
  \]
  This lets us use one public source while still having a smooth DA-style signal.

---

## Battery model (physics)

**SoC update (time step \(\Delta t\) hours):**
\[
s_{t+1} = s_t + \eta_{\text{ch}}\,p^{\text{ch}}_t\,\Delta t - \frac{1}{\eta_{\text{dis}}}\,p^{\text{dis}}_t\,\Delta t
\]

**Constraints**
- Power: \(0 \le p^{\text{ch}}_t \le P_{\max}\), \(0 \le p^{\text{dis}}_t \le P_{\max}\)
- **Total power cap (surrogate for non-overlap)**: \(p^{\text{ch}}_t + p^{\text{dis}}_t \le P_{\max}\)
- Energy: \(s_{\min} \le s_t \le s_{\max}\)
- **Terminal SoC**: \(s_T = s_0\) (no “free” energy from the end)

**Degradation (simple throughput model)**
\[
C^{\text{deg}} = \sum_t c_{\text{deg}}\,(p^{\text{ch}}_t + p^{\text{dis}}_t)\,\Delta t \quad [£]
\]

**Profit**
\[
\text{P\&L} = \sum_t \pi_t\,(p^{\text{dis}}_t - p^{\text{ch}}_t)\,\Delta t - C^{\text{deg}}
\]
where \(\pi_t\) is the price used by the strategy.

**KPIs**
- **Cycles** \(= \dfrac{\text{throughput}}{2E_{\max}}\), where throughput \(=\sum_t (p^{\text{ch}}_t + p^{\text{dis}}_t)\Delta t\)
- **Utilisation** \(= \text{avg}(|\text{power}|) / P_{\max}\)
- **P\&L per MWh moved** (helps sanity-check spreads vs costs)

**Defaults used**
- \(E_{\max} = 100\) MWh, \(P_{\max} = 50\) MW
- \(\eta_{\text{ch}} = \eta_{\text{dis}} = 0.95\), so round-trip about \(0.90\)
- \(\Delta t = 0.5\) h, \(s_0 = 50\), \(s_{\min} = 5\), \(s_{\max} = 95\)
- \(c_{\text{deg}} = 6\) £/MWh (tune as you like)

---

## Four strategy sessions

### Session A — rule-based physics (baseline)

- **Idea**: charge if price < p30, discharge if price > p70, else do nothing.
- **Why**: very clear and fast; good to debug SoC, caps, and units.
- **P\&L**: uses the formula above with the chosen price series.

### Session B — perfect-foresight LP (DA only)

- **Variables**: \(p^{\text{ch}}_t, p^{\text{dis}}_t, s_t\)
- **Maximise**
  \[
  \sum_t \text{DA}_t\,(p^{\text{dis}}_t - p^{\text{ch}}_t)\,\Delta t \;-\; c_{\text{deg}}\sum_t (p^{\text{ch}}_t + p^{\text{dis}}_t)\,\Delta t
  \]
- **Subject to**: power caps, **total power cap** \(p^{\text{ch}}_t + p^{\text{dis}}_t \le P_{\max}\), SoC bounds, **SoC dynamics**, **terminal SoC**
- **Why**: this gives a **convex LP** with a strong global optimum. It is a clean upper bound for DA given the constraints.

> Solvers used by cvxpy: ECOS, OSQP, SCS, Clarabel (fallback as needed).  
> OSQP might report `optimal_inaccurate`; that is common for QP-like forms and is usually fine.

### Session C — DA + BM co-allocation (heuristic, physically safe)

- **Split power** each period: reserve \(a^{\text{DA}}_t \in [0,1]\) for DA, and \(1 - a^{\text{DA}}_t\) for BM.  
  \(a^{\text{DA}}_t\) comes from recent spread volatility \(|\text{BM} - \text{DA}|\) (more vol ⇒ more BM).
- Apply **threshold rules** on DA and BM to propose charge/discharge per market.
- **Safety guards**
  - **No buy & sell in the same period**: if both sides trigger, keep the more valuable side and drop the other
  - **Total power cap across both markets**:
    \[
    (p^{\text{ch}}_{\text{DA}} + p^{\text{ch}}_{\text{BM}}) + (p^{\text{dis}}_{\text{DA}} + p^{\text{dis}}_{\text{BM}}) \le P_{\max}
    \]
- **Why**: easy to explain, fast, and safe. You can later upgrade this to a deterministic LP co-optimisation across both prices.

### Session D — risk-aware optimisation (CVaR via Monte Carlo)

- **Scenarios**: simulate DA and BM price paths with a simple **AR(1)** on log-returns (or shifted logs).  
  Returns:
  \[
  r_t = \mu + \phi r_{t-1} + \varepsilon_t, \quad \varepsilon_t \sim \mathcal{N}(0,\sigma^2)
  \]
  Then exponentiate back and clip to guard rails.
- **One schedule** across all scenarios: same decision vectors for all paths.
- **Loss per scenario**: \(L_s = -\Pi_s\), where \(\Pi_s\) is profit in scenario \(s\).

**CVaR** (Conditional Value at Risk) with level \(\beta\)  
Think of it as the **average of the worst** \((1-\beta)\) fraction of losses. For \(\beta = 0.95\), it is the mean of the worst five percent outcomes.

**Rockafellar–Uryasev form** (convex and solver-friendly):
- Variables: \(\alpha\) (VaR-like) and slacks \(z_s \ge 0\)
- Constraints: \(z_s \ge L_s - \alpha\) for each scenario
- CVaR:
  \[
  \operatorname{CVaR}_\beta = \alpha + \frac{1}{(1-\beta)S} \sum_s z_s
  \]

**Objective (risk vs return)**:
\[
\min\ (1-\lambda)\operatorname{CVaR}_\beta \;-\; \lambda\,\mathbb{E}[\Pi] \;+\; \gamma \sum_t o_t \Delta t
\]
- \(\lambda\) moves the balance between return and downside risk
- \(o_t\) is an **overlap proxy** with bounds \(o_t \le \text{total\_charge}_t\), \(o_t \le \text{total\_discharge}_t\)
- We also enforce a **total power cap** each period

This stays **convex** and works well in cvxpy.

---

## How to run

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
python battery_arbitrage.py
