# Healthcare Patient Readmission Risk Prediction (DevSecMLOps)

Predicts 30-day hospital readmission from a **synthetic, anonymised** dataset.
> **Disclaimer:** educational/demo use only. Not for real clinical decisions without professional validation.

## Dataset
`data/raw/healthcare_readmission.csv` – 6,000 synthetic patients, 25 input columns + `readmitted_30_days`
(~15% positive = class imbalance), with ~4% missing values in some columns and a few outliers.

## Models and parameters
| Model | Key parameters |
|---|---|
| Logistic Regression | `max_iter=1000`, `class_weight=balanced` (L2, C=1.0, lbfgs) |
| Random Forest | `n_estimators=300`, `max_depth=10`, `min_samples_leaf=5`, `class_weight=balanced` |
| HistGradientBoosting | tuned by RandomizedSearchCV (8 iters, 3-fold, ROC-AUC): learning_rate, max_depth, max_iter, l2_regularization; `class_weight=balanced` |

Pipeline: median imputation → StandardScaler → OneHotEncoder → engineered features
(`age_x_chronic`, `high_glucose_flag`, `utilisation_score`) → model → isotonic probability calibration →
threshold chosen for ≥70% recall. Best model is selected by PR-AUC. Split 80/20 stratified, `random_state=42`.

## Run
```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
python -m src.generate_data      # create the CSV
python -m src.validate           # data validation
python -m src.train              # train, compare, calibrate, track in MLflow
mlflow ui                        # compare runs at http://127.0.0.1:5000
uvicorn src.api:app --reload     # API docs at http://127.0.0.1:8000/docs
pytest -q                        # tests
```

Example: `curl -X POST localhost:8000/predict -H "Content-Type: application/json" -d @data/sample_input.json`

## API
`GET /`, `/health`, `/ready`, `/model-info` · `POST /predict` · `POST /predict/batch` (CSV upload, ≤2 MB).
Set `API_KEY` (see `.env.example`) to require an `x-api-key` header.

## DevSecOps included
Input validation (Pydantic), optional API-key auth (constant-time compare), request IDs, secure HTTP headers,
no stack traces in responses, upload size/type limits, secrets via env vars, non-root Docker user,
CI with tests + Bandit + pip-audit + Trivy (fails on CRITICAL).

## Not included (from the full outline)
Prometheus/Grafana, Evidently drift reports, DVC, Airflow/Prefect, CodeQL, Gitleaks, Checkov and
scheduled retraining are not implemented here – add them as extensions.
