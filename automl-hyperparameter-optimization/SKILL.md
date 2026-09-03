---
name: automl-hyperparameter-optimization
description: "Best practices for AutoML and hyperparameter search with Optuna, Ray Tune, and PyCaret, covering search-space design, validation splits, and leakage prevention. Use when tuning model hyperparameters, setting up a pruned or distributed hyperparameter search, designing a nested validation scheme, or evaluating whether an AutoML leaderboard result is production-ready."
---

# AutoML and Hyperparameter Optimization

This skill covers designing sound hyperparameter searches and using AutoML tooling (Optuna, Ray Tune, PyCaret, time-series AutoML libraries) without bypassing problem framing, validation design, or explainability.

## Workflow for Running a Hyperparameter Search

1. **Define the target metric and baseline first** — Pick the metric before selecting tooling, and train a simple baseline (linear model, random forest, or naive time-series forecast) with a fixed, minimal search.
2. **Design the validation scheme** — Use nested cross-validation or a final untouched test split for any model-selection claim; use time-aware splits (never shuffled) for time-series problems.
3. **Fit preprocessing inside the fold** — Fit scalers, encoders, and imputers only on the training portion of each fold to prevent leakage.
4. **Define a structured search space** — Use log-scale ranges for learning rates, regularization strength, and tree counts; keep ranges domain-informed rather than arbitrarily broad.
5. **Choose the right tool** — Optuna or Ray Tune for custom training loops with pruning and distributed trials; PyCaret for a quick low-code comparison on a straightforward tabular problem; a time-series-specific library (AutoTS, Merlion, PyAF) when seasonality and horizon handling need first-class support.
6. **Run with resource limits and pruning** — Set a trial or time budget and use early stopping/pruning so bad trials don't consume the full budget.
7. **Track every run** — Log datasets, splits, metric definitions, random seeds, library versions, and the search space itself to MLflow, Weights & Biases, TensorBoard, or an equivalent tracker.
8. **Report against the baseline** — Compare the selected model to the baseline and at least one non-AutoML alternative before calling it production-ready.

## Experiment Design

- Define the target metric before choosing tooling — the metric shapes the search space and the pruning strategy, not the other way around.
- Use nested validation (an inner loop for hyperparameter selection, an outer loop for performance estimation) or a final untouched test split whenever reporting a model-selection claim.
- Use time-aware splits for time-series problems — never shuffle across time boundaries, since that leaks future information into training.
- Fit all preprocessing (scalers, encoders, imputers, feature selection) only on the training fold, never on validation or test data.
- Always include simple baselines: a linear/logistic model, a random forest, or — for time series — a naive/seasonal-naive forecast. A complex model that doesn't beat the baseline is not worth the operational cost.
- Use early stopping and resource limits (max trials, wall-clock budget) for expensive searches so a runaway search doesn't consume unbounded compute.
- Prefer structured, domain-informed search spaces over arbitrarily broad grids — a learning rate range of `1e-5` to `1e-1` on a log scale is more useful than `0.0001` to `10` on a linear scale.

## Search Space Design

- Keep search spaces explicit and reviewed by someone other than the author — an unreviewed space can silently exclude the true optimum or waste budget on implausible regions.
- Use log-scale sampling for learning rates, regularization coefficients, tree counts, and other scale-sensitive hyperparameters.
- Constrain model complexity (max depth, layer width, number of estimators) to keep training time and memory use realistic for the deployment environment.
- Only include preprocessing choices in the search space when they can be applied per-fold without leakage.
- Never tune on the test set — the test set exists solely to report a final, unbiased estimate once tuning is complete.

## Tooling

### Optuna — custom loops with pruning

Use Optuna for fine-grained control over the training loop, trial pruning, and search algorithms (TPE, CMA-ES).

```python
import optuna
from sklearn.datasets import load_breast_cancer
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import cross_val_score, StratifiedKFold

X, y = load_breast_cancer(return_X_y=True)

def objective(trial: optuna.Trial) -> float:
    params = {
        "n_estimators": trial.suggest_int("n_estimators", 50, 500, log=True),
        "max_depth": trial.suggest_int("max_depth", 2, 10),
        "learning_rate": trial.suggest_float("learning_rate", 1e-3, 3e-1, log=True),
        "subsample": trial.suggest_float("subsample", 0.5, 1.0),
    }
    model = GradientBoostingClassifier(random_state=42, **params)
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    scores = cross_val_score(model, X, y, cv=cv, scoring="roc_auc")

    # Report the running mean for pruning support
    trial.report(scores.mean(), step=0)
    if trial.should_prune():
        raise optuna.TrialPruned()
    return scores.mean()

study = optuna.create_study(
    direction="maximize",
    sampler=optuna.samplers.TPESampler(seed=42),
    pruner=optuna.pruners.MedianPruner(n_warmup_steps=5),
)
study.optimize(objective, n_trials=100, timeout=1800)

print("Best AUROC:", study.best_value)
print("Best params:", study.best_params)
```

- Use `optuna.pruners.MedianPruner` or `HyperbandPruner` to stop unpromising trials early, especially for iterative models (gradient boosting, neural networks).
- Set both `n_trials` and `timeout` so the search always terminates within budget.
- Seed the sampler for reproducibility, and log `study.trials_dataframe()` to your experiment tracker.

### Ray Tune — distributed trials

Use Ray Tune when trials need to run across multiple machines/GPUs, or when integrating pruning schedulers like ASHA with a deep learning training loop.

```python
from ray import tune
from ray.tune.schedulers import ASHAScheduler

def train_fn(config):
    # ... build model/optimizer from config, train for several epochs ...
    for epoch in range(config["max_epochs"]):
        val_loss = train_one_epoch(config)  # user-defined training step
        tune.report({"val_loss": val_loss})

search_space = {
    "lr": tune.loguniform(1e-4, 1e-1),
    "batch_size": tune.choice([32, 64, 128]),
    "max_epochs": 20,
}

tuner = tune.Tuner(
    train_fn,
    param_space=search_space,
    tune_config=tune.TuneConfig(
        metric="val_loss",
        mode="min",
        scheduler=ASHAScheduler(max_t=20, grace_period=3),
        num_samples=50,
    ),
)
results = tuner.fit()
best_result = results.get_best_result()
print(best_result.config, best_result.metrics["val_loss"])
```

### PyCaret — quick low-code comparison

Use PyCaret for a fast first pass on a straightforward tabular problem where the metric and preprocessing needs are simple.

```python
from pycaret.classification import setup, compare_models, tune_model, finalize_model

setup(data=df, target="churn", train_size=0.8, session_id=42)
best_model = compare_models(sort="AUC")
tuned_model = tune_model(best_model, optimize="AUC", n_iter=50)
final_model = finalize_model(tuned_model)
```

Treat PyCaret's leaderboard as a starting point for investigation, not a production decision by itself.

### Time-series AutoML

Use AutoTS, Merlion, PyAF, or another project-approved time-series library when forecast-specific concerns — seasonality detection, horizon handling, backtesting with rolling windows — matter more than raw model variety. These libraries build in time-aware cross-validation by default, which generic tabular AutoML tools do not.

### Experiment tracking and environments

- Store run metadata (metric, params, seed, data version, library versions) in MLflow, Weights & Biases, TensorBoard, or a project-approved tracker — never rely on memory or ad hoc spreadsheets.
- Use `uv` or the project's existing package manager to keep search environments reproducible; pin library versions since sampler/pruner behavior can change across releases.

## Reporting

- Report the selected model, the metric used, a confidence interval or variance estimate, the validation scheme, and the final test result — a single point estimate is not sufficient for a production decision.
- Include the best hyperparameters found and the search budget spent (number of trials, wall-clock time) so the search is reproducible and its cost is visible.
- Compare the chosen model against the baseline and at least one non-AutoML alternative.
- Document operational constraints: inference latency, memory footprint, retraining cost, and explainability requirements — a leaderboard-topping model that violates a latency SLA is not deployable as-is.

## Common Mistakes

- Treating leaderboard rank as proof of production readiness — leaderboard metrics ignore latency, memory, and explainability constraints.
- Mixing train/test data during feature engineering (e.g., computing global statistics like mean/frequency encodings before splitting).
- Running massive searches before validating labels and data quality — a search will happily "optimize" against a buggy target.
- Ignoring class imbalance, calibration, or business cost asymmetry when the optimization metric doesn't reflect them (e.g., optimizing accuracy on a 99:1 class split).
- Deploying an AutoML-selected model without reproducible training code and pinned dependencies — if the winning trial can't be rerun, it can't be maintained.
