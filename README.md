# Vol Street: A Quantitative Research Platform for Volatility Alpha

![Project Status: Active Development](https://img.shields.io/badge/status-active%20development-green)
![Language: Python](https://img.shields.io/badge/python-3.9+-blue.svg)
![Core Tech: XGBoost | Optuna | Genetic Algorithm](https://img.shields.io/badge/core%20tech-XGBoost%20|%20Optuna%20|%20GA-orange)
![Infrastructure: AWS | Lambda Labs](https://img.shields.io/badge/infrastructure-AWS%20|%20Lambda%20Labs-blueviolet)

**Note:** The source code for this project is private due to its proprietary nature. This repository serves as a showcase to document the architecture, methodology, and results of the research platform.

Vol Street is a cloud-native quantitative research and trading platform engineered to extract alpha from complex equity volatility derivatives. It integrates a sophisticated, end-to-end machine learning pipeline that uses a Genetic Algorithm for automated feature discovery and a walk-forward validation framework to build robust predictive models.

---

## Visual Showcase

The platform's output is a "confidence score" (0-100) indicating the model's conviction in a directional move over a defined forward period. The following are representative visualizations generated by the platform's analysis tools.

<p align="center">
  <em>Genetic Algorithm: Walk-Forward Optimization of Out-of-Sample Returns</em><br>
  <img src="images/ga_fitness_evolution_equity_curve.jpg" width="700" alt="Genetic Algorithm Fitness Evolution on Out-of-Sample Returns">
</p>

<p align="center" style="max-width: 700px; margin: auto; text-align: left;">
This plot showcases the research engine's core strength: optimizing on <strong>truly out-of-sample (OOS) returns.</strong> Each grey line is the equity curve of a unique individual (a feature set) being evaluated through its own <strong>full walk-forward backtest.</strong> The underlying strategy—a 5-day long position in VIXY (ProShares VIX Short-Term Futures ETF) triggered at >=60% model confidence—is always tested on unseen data. The visible improvement across generations demonstrates a robust process for discovering alpha, not just a curve-fit result, culminating in the optimal feature set highlighted in red.
</p>

<p align="center">
  <em>SHAP Summary Plot identifying top predictive features for model explainability.</em><br>
  <img src="images/feature_shap_summary.png" width="600" alt="SHAP Feature Importance Plot">
</p>

<p align="center">
  <em>3D Visualization of Genetic Algorithm Fitness Landscape</em><br>
  <img src="images/UMAP_preview.png" width="700" alt="3D UMAP Visualization of GA Fitness Landscape">
</p>
<p align="center" style="max-width: 700px; margin: auto; text-align: left;">
Each point in this 3D landscape represents a unique feature set (an individual), colored by its generation. Its height on the Y-axis reflects its fitness score (a blend of Sortino and Calmar ratios). The visualization demonstrates the evolutionary process, as later generations (brighter yellow) trend higher, discovering more robust and profitable feature combinations.
</p>

---

## Research Philosophy & End-to-End Pipeline

The platform is built on a foundation of hypothesis-driven research and rigorous, automated validation to mitigate overfitting and discover genuine alpha.

### 1. Foundational Data Engineering
A comprehensive data pipeline first acquires and processes terabytes of historical data from multiple sources (IVolatility, CBOE, YFinance). It constructs a rich feature set including:
- Implied volatility surfaces and term structures (slope, curvature, skew).
- VIX futures contango/backwardation and roll yield.
- Backtested performance data from dozens of systematic options strategies (strangles, condors, etc.), turning strategy P&L itself into a feature.

### 2. Genetic Algorithm for Automated Feature Discovery
Instead of manual feature selection, a Genetic Algorithm (`xgb_genetic`) systematically evolves populations of feature sets over hundreds of generations. Each individual (a feature subset) is evaluated via a full walk-forward backtest, optimizing for a multi-objective fitness function that balances performance (e.g., Sharpe/Sortino Ratio) and model parsimony.

### 3. Robust Walk-Forward Validation
The core of the platform is a sophisticated walk-forward engine (`xgb_ga_wf.py`) that simulates real-world performance by:
- **Rolling GA Windows:** Periodically re-running the Genetic Algorithm on expanding windows of historical data to discover the optimal feature set for the current regime.
- **Out-of-Sample Testing:** Using the feature set discovered by the GA to train and optimize an XGBoost model, which is then tested on a subsequent, unseen "out-of-sample" period.
- **Hyperparameter Optimization:** Leveraging Optuna within each OOS period to find the best XGBoost hyperparameters, optimizing for a weighted objective of profit, risk-adjusted return, and directional accuracy.

### 4. Explainable AI (XAI) for Model Insight
To ensure models are not black boxes, SHAP (SHapley Additive exPlanations) analysis is integrated throughout the pipeline. This allows for deep inspection of feature contributions, validation of model behavior, and generation of new research hypotheses.

---

## System Architecture

The project is architected as a modular pipeline, separating data preprocessing, feature engineering, backtesting, GA optimization, and model validation into distinct, orchestrated components.

```mermaid
graph TD
    subgraph Data Layer
        A1[IVolatility API]
        A2[CBOE VIX Data]
        A3[YFinance OHLCV]
    end

    subgraph Preprocessing & Feature Engineering
        B[Data Pipelines]
        C{Options Backtesting Framework}
        D["Consolidated Feature Dataset (.parquet)"]
    end
    
    subgraph Machine Learning Core
        E[Walk-Forward Orchestrator]
        F{Genetic Algorithm}
        G["XGBoost Classifier w/ Optuna"]
        H[Out-of-Sample Predictions]
    end
    
    subgraph Analysis & Output
        I[Combined Equity Curve]
        J["Performance Metrics & SHAP Analysis"]
        K[Final Confidence Score]
    end

    A1 --> B
    A2 --> B
    A3 --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> F
    G --> H
    H --> I
    H --> J
    I --> K
    J --> K
```

## Technical Deep Dive: Key Code Components

### Backtesting Framework & Data Pipeline

The features used by the ML model are not just raw market data; many are generated by a comprehensive options backtesting framework that simulates systematic strategies. This engine provides a rich, realistic source of alpha signals with the following core components:

* **Data Orchestration:** An orchestrator manages five distinct data processing modules for IV surfaces, VIX term structure, CBOE indices, and OHLCV data, creating a unified and processed dataset for the backtester.

* **Multi-Strategy Simulation:** The core engine handles complex, multi-leg options strategies (e.g., straddles, strangles, iron condors, butterflies). It supports dynamic entry/exit logic, IV-based trade filtering, and automated handling of corporate actions like stock splits.

* **Intelligent Option Selection:** Implements logic to select specific options contracts based on quantitative criteria, such as finding the closest option to a target delta (e.g., 0.30) or identifying at-the-money (ATM) straddles, while avoiding strike conflicts in multi-leg structures.

* **Comprehensive Metrics & Greeks Generation:** The framework generates a rich feature set beyond simple P&L for each simulated strategy:
    * **Standard Metrics:** Sharpe Ratio, Sortino Ratio, CAGR, Max Drawdown.
    * **Greeks:** Portfolio-level Delta, Gamma, Theta, and Vega are tracked daily, providing critical inputs for volatility-based modeling.
    * **Custom Risk Metrics:** Includes a custom `VAPQ` (Volatility-Adjusted Performance Quotient) metric to provide a normalized risk/return score tailored for options strategies.

### Machine Learning & Optimization

- **Walk-Forward & GA Orchestrator:** The master script for the entire walk-forward validation process. It generates rolling time windows, invokes the Genetic Algorithm for each window, and then runs the out-of-sample XGBoost backtest on the resulting best features.

- **Genetic Algorithm Feature Optimizer:** The core Genetic Algorithm engine. It manages populations of feature sets, parallelizes fitness evaluations using Dask, and applies genetic operators (crossover, mutation) to evolve optimal solutions.

- **Core Classification Engine:** The main entry point for running a single, complete XGBoost backtest. It is called repeatedly by the GA to evaluate individuals and by the walk-forward orchestrator to test the final feature sets.

- **Training & Prediction Core:** The computational heart of the backtester. It handles the detailed walk-forward logic (rolling/expanding windows), feature scaling, model training, prediction, and signal generation based on confidence thresholds.

- **Hyperparameter Optimization (Optuna):** Implements the hyperparameter search using Optuna. It defines a complex objective function that can be weighted to optimize for Sortino ratio, profit, and other financial metrics.

## Key Technologies

| Category | Technologies |
|---|---|
| **Core Language & Libraries** | Python, Pandas, NumPy, SciPy |
| **Machine Learning** | XGBoost, Scikit-learn, Optuna, SHAP |
| **Genetic Algorithm & Parallelism**| DEAP, Dask, Joblib |
| **GPU Acceleration** | RAPIDS (cuDF, cuML), PyTorch (for device management) |
| **Backend & API** | FastAPI, Uvicorn (for potential future API) |
| **Infrastructure & Deployment**| AWS (S3, EC2), Lambda Labs (for GPU compute), Heroku |
| **Data Format** | Parquet, CSV, JSON |
| **Visualization** | Matplotlib, Plotly, Seaborn |