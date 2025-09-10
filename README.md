# 🔋 BESS Arbitrage Strategy Simulation

This project simulates and benchmarks multiple trading strategies for a utility-scale Battery Energy Storage System (BESS) operating in the UK electricity market. It uses real market data fetched from the Elexon API to model profitability and explores the trade-offs between simple heuristics, perfect-foresight optimization, and advanced risk management.

The core objective is to answer the question: **"What is the most profitable and realistic way to operate a large battery in a volatile energy market?"**



---
## The Problem: Grid Volatility & The BESS Opportunity

Modern electricity grids experience significant price volatility. Factors like the intermittent nature of renewable energy (wind/solar), demand fluctuations, and generator outages cause electricity prices to swing dramatically throughout the day.

Battery Energy Storage Systems (BESS) are critical assets that can stabilize the grid and profit from this volatility. The fundamental business case is simple **arbitrage**:
1.  **Charge** the battery when electricity is cheap (low demand, high renewable generation).
2.  **Discharge** (sell) the electricity back to the grid when it is expensive (high demand, low generation).

While the concept is simple, designing a strategy to maximize profit while respecting the battery's physical limitations (degradation, efficiency, power limits) is a complex optimization problem.

---
## The Strategies: A Hierarchy of Models

This project implements four distinct strategies, each representing a different level of complexity and commercial sophistication. This allows for a robust benchmark of performance.

### Strategy A: The Simple Heuristic
-   **Logic:** This is a basic, rule-based model that follows a simple trigger mechanism. It calculates the 30th and 70th price percentiles for the entire period and issues buy/sell signals accordingly.
-   **Concept:** It represents a "naive" or baseline strategy that reacts to prices without any forecasting or advanced planning.
-   **Decision Making:** `IF price < 30th_percentile THEN charge`, `IF price > 70th_percentile THEN discharge`.

### Strategy B: The Perfect-Foresight LP Model
-   **Logic:** This model uses **Linear Programming (LP)** to calculate the absolute maximum possible profit over the entire 90-day period. It is given "perfect foresight"—complete knowledge of all future prices.
-   **Concept:** It serves as a crucial, idealized benchmark. While unachievable in reality, it defines the theoretical maximum profit (`PnL_max`) available in the market, allowing us to measure the efficiency of other, more realistic strategies.
-   **Decision Making:** Solves a constrained optimization problem to find the optimal charge/discharge schedule for every 30-minute period.

### Strategy C: The Co-allocation Heuristic
-   **Logic:** This is a more advanced rule-based model that operates in two markets simultaneously: the Day-Ahead (DA) market and the more volatile Balancing Mechanism (BM). It dynamically allocates the battery's power capacity between the two markets based on recent volatility.
-   **Concept:** It models a more commercially savvy heuristic that attempts to capture value from multiple revenue streams.
-   **Decision Making:** Uses price percentile triggers for both DA and BM markets, with physically realistic constraints to prevent simultaneous charging/discharging.

### Strategy D: The CVaR Risk-Averse Optimizer
-   **Logic:** This is the most sophisticated model. It uses **Conditional Value at Risk (CVaR)**, a common risk metric in finance, to find a trading strategy that not only generates profit but also limits exposure to significant financial losses (downside risk).
-   **Concept:** It moves beyond simple profit maximization to model the decision-making of a risk-averse asset operator or trading firm.
-   **Decision Making:** Solves a complex optimization problem across hundreds of simulated price scenarios to find a schedule that offers the best risk-adjusted return.

---
## Results & Analysis (90-Day Simulation)

The strategies were simulated over a 90-day period using a 100 MWh / 50 MW battery model with realistic degradation costs and efficiency.

| Strategy                               | 90-Day Profit (£) | Key Insight                                         |
| -------------------------------------- | ----------------- | --------------------------------------------------- |
| A: Simple Heuristic                    | £84,843           | Provides a baseline performance.                    |
| B: Perfect-Foresight LP                | **£283,253** | The theoretical maximum profit (the "perfect" score). |
| C: Co-allocation Heuristic             | £269,853          | High performance, but limited by simple rules.      |
| D: CVaR Risk-Averse Optimizer          | £38,687           | Low profit due to an overly cautious risk penalty.    |

### 💡 Key Insights from the Results:

1.  **Optimization Creates Massive Value:** The perfect-foresight LP model (Strategy B) generated **234% more profit** than the simple heuristic (Strategy A). This powerfully quantifies the financial value of using a mathematical optimization approach over basic reactive rules.

2.  **Optimal Strategy > Better Data:** The LP model (B), which only used one price series, still outperformed the more complex co-allocation heuristic (C) which used two. This demonstrates that the *quality of the decision-making algorithm (optimization) can be more important than simply having access to more data.*

3.  **Model Objectives are Critical:** The CVaR model's (Strategy D) unexpectedly low profit was a direct result of an overly severe penalty placed on a specific action in its objective function. This forced the model into an extremely conservative state, teaching a valuable lesson on how sensitive optimizers are to their defined goals. It highlights the importance of careful model tuning.

---
