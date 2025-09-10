# Battery Energy Storage Arbitrage Simulation

A suite of Python models for simulating and optimizing battery energy storage arbitrage strategies in the UK electricity market, using data from the Elexon API.

## Key Features
- **Data Ingestion:** Fetches real UK System Price data from the Elexon API with a robust fallback to synthetic data.
- **Strategy Benchmarking:** Compares four different trading strategies with increasing complexity:
  1.  **Heuristic Model:** A simple, rule-based physics simulation.
  2.  **Perfect-Foresight LP:** An idealized Linear Programming model (`cvxpy`) to establish a theoretical maximum profit.
  3.  **DA+BM Co-allocation:** A heuristic that trades across two different markets.
  4.  **CVaR Risk Optimization:** A sophisticated model that optimizes for profit while managing downside risk.
- **Physical Realism:** The models account for key physical and financial constraints like battery degradation, efficiency losses, power limits, and state-of-charge.

## Concepts Demonstrated
- **Quantitative Finance:** Arbitrage, Risk Management (CVaR), Price Volatility.
- **Operations Research:** Linear Programming, Constrained Optimization.
- **Energy Markets:** UK Day-Ahead (DA) and Balancing Mechanism (BM) dynamics.
- **Software Engineering:** API data handling, object-oriented principles, data visualization.

## How to Run
1.  Ensure you have Python 3 installed.
2.  Install the required libraries:
    ```bash
    pip install pandas numpy requests matplotlib cvxpy
    ```
3.  Run the main script from your terminal:
    ```bash
    python arbitrage_simulation.py
    ```
4.  The script will generate analysis reports, figures, and a summary in a `reports/` directory.
