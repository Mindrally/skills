---
name: tensorflow-deep-learning
description: "Best practices for building, training, evaluating, and deploying neural networks with TensorFlow and Keras. Use when writing tf.data input pipelines, defining or training a Keras model, configuring callbacks for checkpointing and early stopping, evaluating a trained model, or exporting a model for serving."
---

# TensorFlow and Deep Learning

This skill covers project structure, model development, training, evaluation, and deployment practices for neural networks built with TensorFlow and Keras.

## Workflow for Training and Shipping a Model

1. **Separate concerns into modules** — Keep data loading, model definition, training, evaluation, and serving code in distinct files; treat notebooks as exploratory only.
2. **Build the input pipeline first** — Use `tf.data.Dataset` for scalable, prefetching input processing; validate shapes, dtypes, label ranges, and class balance before writing any model code.
3. **Split before any fitting** — Create train/validation/test splits before fitting normalization statistics or augmentation parameters, to avoid leakage.
4. **Start with a tiny baseline** — Build a small model and run a tiny overfit test (fit on a handful of examples until loss goes to ~0) to prove the training loop works before scaling up.
5. **Train with callbacks** — Wire in `ModelCheckpoint` (save best by validation metric, not final epoch), `EarlyStopping`, a learning-rate schedule, and `TensorBoard` logging.
6. **Evaluate with task-appropriate metrics** — Use a held-out test set only for final reporting, never for tuning.
7. **Export with a explicit signature** — Save with `model.export()` or `tf.saved_model.save()` and define serving input signatures explicitly.
8. **Smoke test the export** — Load the exported model and run inference on sample inputs before deploying.

## Project Structure

- Separate data loading, model definition, training, evaluation, and serving code into distinct modules (e.g., `data.py`, `model.py`, `train.py`, `evaluate.py`, `serve.py`).
- Keep model hyperparameters in typed config objects (dataclasses) or config files, not scattered as literals through training code.
- Store checkpoints, logs, and exported models outside source directories (e.g., under `artifacts/` or a configured output path), so they don't get swept into version control accidentally.
- Keep notebooks exploratory; once an approach is validated, move the repeatable training code into modules that can be run as scripts and covered by tests.

```
project/
  data.py          # tf.data pipeline construction
  model.py         # Keras model definition
  config.py        # typed hyperparameter config
  train.py         # training loop / Keras fit orchestration
  evaluate.py       # metrics computation on held-out data
  serve.py         # export + inference smoke test
  artifacts/       # checkpoints, logs, saved models (gitignored)
```

## Input Pipeline (tf.data)

```python
import tensorflow as tf

def build_dataset(file_pattern: str, batch_size: int, shuffle: bool) -> tf.data.Dataset:
    files = tf.data.Dataset.list_files(file_pattern, shuffle=shuffle)
    ds = files.interleave(
        tf.data.TFRecordDataset,
        cycle_length=tf.data.AUTOTUNE,
        num_parallel_calls=tf.data.AUTOTUNE,
    )
    ds = ds.map(_parse_example, num_parallel_calls=tf.data.AUTOTUNE)
    if shuffle:
        ds = ds.shuffle(buffer_size=10_000)
    ds = ds.batch(batch_size, drop_remainder=False)
    return ds.prefetch(tf.data.AUTOTUNE)

def _parse_example(record: tf.Tensor) -> tuple[tf.Tensor, tf.Tensor]:
    feature_spec = {
        "features": tf.io.FixedLenFeature([32], tf.float32),
        "label": tf.io.FixedLenFeature([], tf.int64),
    }
    parsed = tf.io.parse_single_example(record, feature_spec)
    return parsed["features"], parsed["label"]
```

- Use `num_parallel_calls=tf.data.AUTOTUNE` on `.map()`/`.interleave()` and always end the pipeline with `.prefetch(tf.data.AUTOTUNE)`.
- Shuffle before batching, with a buffer large enough to approximate a full shuffle for the dataset size.
- Validate data shapes, dtypes, label ranges, and class balance on a sample batch before starting a long training run.

## Model Development

- Use Keras layers and models (`tf.keras.Model`, `tf.keras.Sequential`) unless a specific need requires lower-level `tf.GradientTape` control.
- Prefer explicit input shapes and named inputs/outputs — this makes serving signatures and debugging easier.
- Start with a small baseline model and confirm it can overfit a tiny subset of data (loss near zero) before scaling depth, width, or dataset size — this catches pipeline and loss-function bugs early.
- Pin random seeds (`tf.random.set_seed`, `numpy.random.seed`) where reproducibility matters, but document that GPU execution can still be nondeterministic for some ops (e.g., certain cuDNN reductions).
- Only enable mixed precision (`tf.keras.mixed_precision.set_global_policy("mixed_float16")`) after validating numerical stability at full precision — some losses (e.g., ones involving `log` or `exp`) need loss scaling to avoid underflow.

```python
import tensorflow as tf

def build_model(input_dim: int, num_classes: int) -> tf.keras.Model:
    inputs = tf.keras.Input(shape=(input_dim,), name="features")
    x = tf.keras.layers.Dense(128, activation="relu")(inputs)
    x = tf.keras.layers.Dropout(0.3)(x)
    x = tf.keras.layers.Dense(64, activation="relu")(x)
    outputs = tf.keras.layers.Dense(num_classes, activation="softmax", name="predictions")(x)
    return tf.keras.Model(inputs=inputs, outputs=outputs, name="baseline_classifier")
```

## Training

```python
import tensorflow as tf

model = build_model(input_dim=32, num_classes=10)
model.compile(
    optimizer=tf.keras.optimizers.Adam(learning_rate=1e-3),
    loss="sparse_categorical_crossentropy",
    metrics=["accuracy"],
)

callbacks = [
    tf.keras.callbacks.ModelCheckpoint(
        filepath="artifacts/checkpoints/best.keras",
        monitor="val_accuracy",
        save_best_only=True,
    ),
    tf.keras.callbacks.EarlyStopping(
        monitor="val_loss", patience=5, restore_best_weights=True,
    ),
    tf.keras.callbacks.ReduceLROnPlateau(
        monitor="val_loss", factor=0.5, patience=3, min_lr=1e-6,
    ),
    tf.keras.callbacks.TensorBoard(log_dir="artifacts/logs"),
]

train_ds = build_dataset("data/train-*.tfrecord", batch_size=64, shuffle=True)
val_ds = build_dataset("data/val-*.tfrecord", batch_size=64, shuffle=False)

history = model.fit(
    train_ds,
    validation_data=val_ds,
    epochs=50,
    callbacks=callbacks,
)
```

- Split data into train/validation/test before fitting any normalization layer or augmentation parameters — fitting on the full dataset leaks test statistics into training.
- Use the validation set for tuning (architecture, learning rate, regularization); reserve a separate test set purely for final reporting, touched once.
- Track loss curves, metrics, learning rate, and resource utilization (`TensorBoard`, or an experiment tracker) for every run.
- Save the checkpoint with the best validation metric (`save_best_only=True`), not just the weights from the final epoch — the final epoch is not necessarily the best one, especially with `EarlyStopping`.

## Evaluation

- Report task-appropriate metrics: AUROC/F1/calibration for classification, perplexity or BLEU/ROUGE for language tasks, MAE/RMSE for regression.
- Include confusion matrices or per-class error slices for classification tasks — aggregate accuracy hides class-level failures.
- Evaluate on edge cases and known distribution shifts when data allows (e.g., a held-out slice from a different time period or source).
- Compare against non-neural baselines (logistic regression, gradient-boosted trees) when the dataset is small or tabular — a deep model that underperforms a simple baseline is a signal, not a footnote.

```python
test_ds = build_dataset("data/test-*.tfrecord", batch_size=64, shuffle=False)
results = model.evaluate(test_ds, return_dict=True)
print(results)  # {'loss': ..., 'accuracy': ...}
```

## Deployment

- Export models with an explicit input signature so serving infrastructure (TF Serving, Vertex AI, etc.) knows the expected input shape and dtype:

```python
model.export("artifacts/saved_model/v1")

# Or with an explicit serving signature:
@tf.function(input_signature=[tf.TensorSpec(shape=[None, 32], dtype=tf.float32, name="features")])
def serve_fn(features):
    return {"predictions": model(features, training=False)}

tf.saved_model.save(model, "artifacts/saved_model/v1", signatures={"serving_default": serve_fn})
```

- Keep preprocessing identical between training and serving — bake normalization/tokenization into the exported graph (e.g., `tf.keras.layers.Normalization` as part of the model) rather than duplicating logic in a separate serving-side script.
- Add a smoke test that loads the exported `SavedModel` and runs inference on a handful of known sample inputs, asserting outputs are in the expected range/shape, before promoting a model to production.
- Monitor latency, memory, prediction drift, and input schema changes once deployed — a model that was correct at export time can silently degrade as upstream data shifts.

## Common Mistakes

- Tuning architecture before verifying labels and data quality — a data bug will outlast any architecture change.
- Leaking validation or test data through preprocessing or augmentation fit on the full dataset instead of the training split alone.
- Relying on accuracy alone for imbalanced datasets — use precision/recall, F1, or AUROC instead.
- Deploying a notebook-only model with no reproducible training script or pinned dependencies.
- Ignoring batch size, dtype, or device (CPU/GPU) differences between training and inference, which can silently change numerical results (especially under mixed precision).
