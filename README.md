# conservation-regime

**Conservation ratio regime detection, anomaly analysis, and spectral forecasting for time-varying graphs — pure Rust, zero dependencies.**

## The Big Idea: CR Dropping IS the Early Warning

Every graph has a spectral fingerprint. The **conservation ratio** (CR) is λ₂/λ_max of the normalized Laplacian — a single number that tells you how "coherent" the graph is.

- CR ≈ 1: the graph is fully connected, information flows everywhere, everything agrees
- CR ≈ 0: the graph is falling apart, clusters are disconnecting, coherence is collapsing
- **CR dropping** is the early warning signal. Before things break, they slow down.

This is the graph-theoretic version of **critical slowing down** — a phenomenon from statistical physics where systems near a phase transition respond more slowly to perturbations. The spectral gap narrows. The second eigenvalue drops. CR catches it.

```
Graph health over time:

CR
1.0 ┤ ═════════════╗
    │               ╚══╗
0.8 ┤                   ╚══╗
    │                       ╚══╖
0.6 ┤                          ╚══─ ···
    │                         ↑ regime change
0.4 ┤                    detected here
    │
0.0 ┤
    └────────────────────────────────────→ time
         healthy    degrading   critical
```

## The Conservation Ratio

CR = λ₂(L_norm) / λ_max(L_norm) where L_norm = D^{-1/2} L D^{-1/2} is the normalized Laplacian.

```text
λ₂ = algebraic connectivity → how hard it is to disconnect the graph
λ_max = total spectral energy → maximum "stretch" of the Laplacian
CR = λ₂ / λ_max → normalized measure of structural coherence
```

**Properties:**
- CR = 1 for complete graphs (maximum connectivity)
- CR = 0 for disconnected graphs
- CR is scale-invariant (adding vertices with the same structure preserves CR)
- CR is related to the Cheeger constant by Cheeger's inequality: CR/2 ≤ h ≤ √(2·CR)

## Regime Detection

A **regime** is a time interval where CR is statistically stable. Regime changes happen when CR shifts beyond what noise would explain.

The detection algorithm:
1. Compute CR for the graph at each time step → time series
2. Track a rolling mean and variance of CR
3. Flag a regime change when CR deviates by > k standard deviations from the rolling mean
4. Use QR-based eigenvalue computation for numerical stability

```rust
// Conceptual usage (based on the crate's design):
use conservation_regime::*;

// Build a time-varying graph sequence
let mut sequence = GraphSequence::new();
for t in 0..1000 {
    let g = build_graph_at_time(t);
    sequence.push(g);
}

// Detect regime changes
let regimes = sequence.detect_regimes(DetectionConfig {
    window: 50,              // rolling window size
    threshold: 2.0,          // standard deviations for detection
    min_regime_length: 10,   // minimum steps for a valid regime
});

for regime in &regimes {
    println!("Regime [{}, {}): CR = {:.4} ± {:.4}",
        regime.start, regime.end, regime.mean_cr, regime.std_cr);
}

// Early warning: is CR trending down?
let warning = sequence.early_warning(EarlyWarningConfig {
    lookback: 100,
    trend_threshold: -0.001,  // CR decreasing by more than this per step
    variance_inflation: 1.5,  // variance increasing (critical slowing down)
});
```

## The QR Eigenvalue Engine

Under the hood, regime detection needs fast, stable eigenvalue computation. The library uses QR decomposition with shifts — the gold standard for dense eigenvalue problems:

1. **QR iteration:** Factor A = QR, then form A' = RQ. Repeat until triangular.
2. **Wilkinson shifts:** Choose shifts near the eigenvalues for cubic convergence.
3. **Deflation:** Once a subdiagonal element is small enough, extract the eigenvalue and work on the smaller problem.

For the conservation ratio specifically, we only need λ₂ and λ_max, not the full spectrum. The library exploits this with partial eigenvalue extraction — compute only what's needed.

## Early Warning Signals

The library implements three spectral early warning signals inspired by critical slowing down theory:

### 1. Trend Detection
CR decreasing over time → structural degradation. Fit a linear model to recent CR values and flag negative trends exceeding a threshold.

### 2. Variance Inflation
Near a bifurcation, the system's variance increases (critical slowing down → slower recovery from perturbations → larger fluctuations). Track the rolling variance of CR and flag when it increases.

### 3. Autocorrelation Increase
Also from critical slowing down: the system becomes more correlated with its own recent past. Compute lag-1 autocorrelation of CR and flag increases.

```
Early warning signals before a crash:

         ┌─── trend (CR decreasing)
         │    ┌─── variance (fluctuations growing)
         │    │    ┌─── autocorrelation (memory increasing)
         ▼    ▼    ▼
    ──────────────── crash ─────────────
    All three light up BEFORE the crash.
```

## Examples

### Financial Networks

```rust
// Correlation network of stock returns
let mut sequence = GraphSequence::new();
for day in trading_days {
    let mut g = Graph::new(n_stocks);
    for i in 0..n_stocks {
        for j in (i+1)..n_stocks {
            let corr = compute_correlation(returns[i], returns[j], window);
            if corr > threshold {
                g.add_edge(i, j, corr);
            }
        }
    }
    sequence.push(g);
}

// Market regime detection
let regimes = sequence.detect_regimes(DetectionConfig::default());
// Regime 1: bull market (high CR, stocks move together)
// Regime 2: sector rotation (medium CR, some clusters)
// Regime 3: crisis (low CR, correlations break down paradoxically)
```

The paradox: during a crisis, all correlations go to 1 (everything crashes together), *but* the graph structure changes — the correlation network becomes dominated by a few strong links rather than many moderate ones. CR captures this structural shift.

### Ecological Networks

```rust
// Species interaction network over time
let mut sequence = GraphSequence::new();
for year in survey_years {
    let g = build_food_web(year);
    sequence.push(g);
}

// Ecosystem regime shifts
let warning = sequence.early_warning(EarlyWarningConfig {
    lookback: 10,  // 10-year window
    trend_threshold: -0.005,
    variance_inflation: 1.3,
});

if warning.is_active {
    println!("⚠ Ecosystem may be approaching a tipping point");
    println!("   CR trend: {:.4}/year", warning.trend);
    println!("   Variance ratio: {:.2}x baseline", warning.variance_ratio);
}
```

Ecosystem collapse follows the same critical-slowing-down pattern: recovery from perturbations slows, variance increases, autocorrelation increases. CR captures all of this through the spectral properties of the interaction network.

## Connection to the SuperInstance Ecosystem

This crate is built on top of:

- **[spectral-graph-core]** — Jacobi eigenvalues, graph construction, conservation ratio computation, Fiedler vectors
- **[sheaf-cohomology]** — when the graph is the backbone of a sheaf, CR on the sheaf Laplacian measures *data* coherence, not just structural coherence

The chain: **spectral-graph-core** gives you the eigenvalues → **conservation-regime** watches them change over time → **sheaf-cohomology** tells you *why* they're changing (local-to-global obstruction).

## API Summary

| Type | Description |
|------|-------------|
| `GraphSequence` | Time-ordered sequence of graphs |
| `Regime` | A time interval with stable CR statistics |
| `DetectionConfig` | Parameters for regime detection |
| `EarlyWarningConfig` | Parameters for early warning signals |
| `EarlyWarning` | Result of early warning analysis |

| Method | Description |
|--------|-------------|
| `detect_regimes` | Segment the time series into CR-stable regimes |
| `early_warning` | Compute trend, variance, and autocorrelation signals |
| `cr_series` | Get the raw CR time series |

## Honest Limitations

- **No streaming.** The current design requires the full graph sequence up front. For real-time monitoring, you'd need to adapt the rolling window logic.
- **Dense eigenvalues.** Uses Jacobi (from spectral-graph-core) which is O(n³) per graph. For large graphs (>500 vertices), this becomes expensive. A Lanczos-based partial eigensolver would help.
- **Simple regime model.** Regime boundaries are detected via threshold crossings, not hidden Markov models or Bayesian changepoint detection. More sophisticated statistical models would improve accuracy.
- **No weighted CR variants.** The conservation ratio uses the standard normalized Laplacian. Weighted variants (using edge importance) are not yet supported.
- **No confidence intervals.** The regime detection gives point estimates for CR but no uncertainty quantification.

## When to Use This

- **Financial markets** — detect regime changes in correlation structure
- **Ecological monitoring** — early warning of ecosystem tipping points
- **Infrastructure networks** — power grid, transportation, communication network health
- **Social network analysis** — detect community structure shifts
- **Any time-varying graph** where structural coherence matters

## Installation

```toml
[dependencies]
conservation-regime = "0.1"
```

## License

MIT

[spectral-graph-core]: https://github.com/SuperInstance/spectral-graph-core
[sheaf-cohomology]: https://github.com/SuperInstance/sheaf-cohomology

## Ecosystem Integration

- Detects and classifies conservation law regimes using spectral graph analysis
- Uses `spectral-graph-core` for eigenvalue-based regime signatures
- Integrates with `conservation-protocol` for regime-aware constraint broadcasting
- Can trigger regime transitions that propagate across the agent fleet
- Part of the conservation-law stack alongside `conservation-protocol` and `emergent-coupling`

