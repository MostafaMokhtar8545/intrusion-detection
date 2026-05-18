# Network Intrusion Detection

A complete machine learning pipeline for classifying network traffic using the NSL-KDD dataset. Four models are trained and benchmarked: Random Forest, XGBoost, Isolation Forest, and a Multi-Layer Perceptron (MLP).

---

## Results

| Model | Accuracy | Macro F1 | Train Time (s) | Latency (ms/sample) | Interpretable |
|---|---|---|---|---|---|
| Random Forest | 0.7743 | 0.5214 | ~39 | 0.0082 | Yes |
| XGBoost | 0.7979 | 0.6639 | ~64 | 0.0068 | Yes |
| Isolation Forest* | 0.5519 | 0.5020 | ~1 | 0.0128 | Partial |
| MLP | 0.8041 | 0.5877 | ~161 | 0.0027 | No |

> \* Isolation Forest is a binary detector (normal vs. attack) — not directly comparable to the 5-class models.  
> **Primary metric is Macro F1**, not accuracy. Macro F1 weights all 5 classes equally and is the honest metric for imbalanced classification.

**Winner: XGBoost** — best Macro F1, interpretable, fast inference.

---

## Dataset

**NSL-KDD** — a refined version of the KDD Cup 1999 network intrusion dataset.

- `KDDTrain+.txt` — 125,973 labeled connections
- `KDDTest+.txt` — 22,544 labeled connections
- 41 features per connection (duration, bytes, error rates, service type, etc.)
- 5 classes: `normal`, `dos`, `probe`, `r2l`, `u2r`

Download from: https://www.unb.ca/cic/datasets/nsl.html

Place both `.txt` files in the same directory as the notebook before running.

---

## Pipeline

```
Raw Data → Audit → Preprocessing → Scaling → SMOTE → Feature Selection → Modeling → Benchmark
```

1. **Data Audit** — shape, null counts, class distribution, label mapping
2. **Preprocessing** — drop difficulty column, map 22 attack types to 5 classes, one-hot encode categoricals
3. **Feature Scaling** — StandardScaler fit on train only, applied to both splits
4. **Class Balancing** — SMOTE applied to training set only (336,715 balanced samples)
5. **Feature Selection** — RF-based importance, threshold 0.001, reduces 122 → 48 features
6. **Modeling** — RF, XGBoost, Isolation Forest, MLP with early stopping
7. **Benchmark** — accuracy, Macro F1, confusion matrices, inference latency

---

## Quickstart with `uv`

### 1. Install `uv` (once, globally)

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
```

### 2. Clone or download this project

```bash
cd /path/to/your/projects
# place network_intrusion_detection.ipynb, KDDTrain+.txt, KDDTest+.txt here
```

### 3. Initialize the environment

```bash
uv init .
uv add pandas numpy matplotlib seaborn scikit-learn imbalanced-learn xgboost torch ipykernel
```

This creates a `.venv` folder inside the project directory and installs all dependencies into it. After the first install, subsequent `uv add` calls for the same packages are instant (served from `uv`'s global cache at `~/.cache/uv/`).

### 4. Register the Jupyter kernel

```bash
uv run python -m ipykernel install --user --name=intrusion --display-name "Intrusion Detection"
```

### 5. Run the notebook

**Option A — VS Code (recommended)**

Open `network_intrusion_detection.ipynb` in VS Code. Click **Select Kernel** in the top right → **Python Environments** → select the path ending in `.venv/bin/python` inside your project folder.

> Do not select the `.venv` folder itself — select the `python` binary inside it. Selecting the folder causes VS Code to create a nested environment.

**Option B — Jupyter in browser**

```bash
uv run jupyter notebook
```

Then open `network_intrusion_detection.ipynb` and select kernel **"Intrusion Detection"** from the Kernel menu.

### 6. Run all cells top to bottom

All random states are fixed to `SEED = 42`. Running the notebook from top to bottom will reproduce all results exactly.

---

## Reproducibility

| Component | Fixed seed |
|---|---|
| NumPy | `np.random.seed(42)` |
| PyTorch | `torch.manual_seed(42)` |
| SMOTE | `random_state=42` |
| Random Forest | `random_state=42` |
| XGBoost | `random_state=42` |
| Isolation Forest | `random_state=42` |

MLP weights from the best epoch are saved to `mlp_best_weights.pth` automatically during training. To reload without retraining:

```python
model.load_state_dict(torch.load('mlp_best_weights.pth'))
model.eval()
```

---

## Output Files

After running the notebook, the following files are generated in the project directory:

| File | Description |
|---|---|
| `feature_importance.png` | Top 20 features by RF importance score |
| `mlp_training_history.png` | MLP loss and validation F1 per epoch |
| `benchmark.png` | Bar charts comparing all models |
| `confusion_matrices.png` | Confusion matrices for RF, XGBoost, MLP |
| `mlp_best_weights.pth` | Saved MLP weights from best epoch |

---

## Requirements

All managed by `uv`. No manual `requirements.txt` needed — dependencies are declared in `pyproject.toml` and pinned in `uv.lock`.

| Package | Purpose |
|---|---|
| `pandas`, `numpy` | Data handling |
| `matplotlib`, `seaborn` | Visualization |
| `scikit-learn` | Preprocessing, RF, Isolation Forest, metrics |
| `imbalanced-learn` | SMOTE |
| `xgboost` | XGBoost classifier |
| `torch` | MLP (CPU only — no NVIDIA GPU required) |
| `ipykernel` | Jupyter kernel registration |

---

## Notes

- **No GPU required.** PyTorch runs on CPU. NSL-KDD is small enough that MLP training completes in ~3 minutes on CPU.
- **U2R and R2L recall is low across all models.** This is a known dataset limitation — U2R has only 52 real training samples. This is a finding, not a bug.
- **Isolation Forest metrics are binary.** It detects normal vs. attack only, not the specific attack type. Its row in the benchmark table is included for comparison of paradigms, not as a fair head-to-head.