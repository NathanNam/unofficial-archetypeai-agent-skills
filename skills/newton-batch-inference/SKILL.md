---
name: newton-batch-inference
description: >
  Create and monitor batch processing jobs on the Archetype AI Newton platform.
  Use when the user wants to run inference on large datasets asynchronously
  or classify sensor data at scale using the Machine State Classification
  pipeline. This skill covers job creation, status polling, event log retrieval,
  output download, and config optimization.
  Do NOT use for real-time streaming inference (use newton-machine-state or newton-activity-monitor).
  Do NOT use for file uploads (use newton-batch-upload).
---

# Newton Batch Inference

Create and monitor asynchronous batch processing jobs on the Newton platform. Process large datasets (millions of rows) without maintaining a real-time session.

## When to Apply

- User wants to classify a large CSV dataset at scale
- User asks about batch processing or asynchronous inference
- User wants to run Newton on a file without a WebSocket session
- User needs to process multiple files through a pipeline
- User wants to optimize pipeline config for best accuracy

## Machine State Classification Pipeline

Classifies time-series sensor data using n-shot examples. Uses Newton to vectorize sensor windows, then a KNN classifier to predict state.

**Pipeline key:** `machine-state-classification`

**Available model types:**
- `omega_1_3_surface` — surface sensor monitoring (9 channels)
- `omega_1_3_power_drive` — downhole power drive monitoring (9 channels)

**Input ports:**
- `worker.inference` — CSV files to classify
- `worker.n_shots` — labeled CSV example files with `metadata.class`

**Data requirements:**
- CSV files must include a timestamp column (Unix epoch)
- `data_columns` must specify exactly 9 sensor columns — more or fewer causes shape errors
- `window_size` controls time steps per window (e.g., 16, 32, 64, 128) — value of 1 causes tensor errors
- N-shot files need enough rows to produce windows: `(rows - window_size) / step_size >= n_neighbors`
- N-shot labels should use ACTC rig mode codes for consistent ground truth, not sensor heuristics

**Recommended drilling sensor columns:**
```
DATE_TIME, BPOS, DBTM, FLWI, HDTH, HKLD, ROP, RPM, SPPA, WOB
```

```bash
curl -s -X POST "$BASE_URL/batch/jobs" \
  -H "Authorization: Bearer $ATAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "drilling-classification",
    "pipeline_type": "batch",
    "pipeline_key": "machine-state-classification",
    "inputs": {
      "worker.inference": [{"file_id": "volve_inference.csv"}],
      "worker.n_shots": [
        {"file_id": "volve_drilling.csv", "metadata": {"class": "drilling"}},
        {"file_id": "volve_not_drilling.csv", "metadata": {"class": "not_drilling"}}
      ]
    },
    "parameters": {
      "worker": {
        "parallelism": 1,
        "config": {
          "model_type": "omega_1_3_surface",
          "batch_size": 8,
          "reader_config": {
            "data_columns": ["BPOS","DBTM","FLWI","HDTH","HKLD","ROP","RPM","SPPA","WOB"],
            "timestamp_column": "DATE_TIME",
            "window_size": 64,
            "step_size": 1
          },
          "classifier_config": {"n_neighbors": 5, "metric": "euclidean", "weights": "uniform"},
          "flush_every_n_iteration": 1000
        }
      }
    }
  }'
```

## Monitoring Jobs

### Check Status

```bash
curl -s "$BASE_URL/batch/jobs/$JOB_ID" \
  -H "Authorization: Bearer $ATAI_API_KEY"
```

Status lifecycle: `PENDING` → `RUNNING` → `COMPLETED` / `FAILED` / `CANCELLED`

### View Events/Logs

```bash
curl -s "$BASE_URL/batch/jobs/$JOB_ID/events" \
  -H "Authorization: Bearer $ATAI_API_KEY"
```

### Download Outputs

```bash
# List output artifacts (paginated, presigned S3 URLs, 1-hour expiry)
curl -s "$BASE_URL/batch/jobs/$JOB_ID/outputs?limit=50&offset=0" \
  -H "Authorization: Bearer $ATAI_API_KEY"

# Download artifact (no auth needed — signature is in the URL)
curl -s -o output.csv "$PRESIGNED_URL"
```

Machine State output format:
```csv
DATE_TIME,Prediction
1232522644,drilling
1190749684,not_drilling
```

### List All Jobs

```bash
curl -s "$BASE_URL/batch/jobs?limit=10&offset=0" \
  -H "Authorization: Bearer $ATAI_API_KEY"
```

## Config Optimization

The Machine State pipeline has several tunable hyperparameters. Use grid search with a small test dataset (200 rows) to find the best config before running on the full dataset:

| Parameter | Values to try | Description |
|-----------|--------------|-------------|
| `window_size` | 16, 32, 64, 128 | Time steps per classification window |
| `n_neighbors` | 3, 5, 7, 11 | KNN neighbors |
| `metric` | euclidean, cosine, manhattan | Distance metric |
| `weights` | uniform, distance | KNN weight function |

See `optimize_config.py` in [archetypeai-batch-examples-volve](https://github.com/archetypeai/archetypeai-batch-examples-volve) for an automated grid search script.

## API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v0.5/batch/jobs` | POST | Create a batch job |
| `/v0.5/batch/jobs` | GET | List all jobs |
| `/v0.5/batch/jobs/{job_id}` | GET | Get job status |
| `/v0.5/batch/jobs/{job_id}/events` | GET | Get job events/logs |
| `/v0.5/batch/jobs/{job_id}/outputs` | GET | List output artifacts (paginated, presigned URLs) |

## Fine-Tuning (TBD)

Fine-tuning via `POST /v0.5/internal/experiment/runner/jobs` is not yet available. It requires JSONL-formatted training data. See [archetypeai-batch-examples-volve](https://github.com/archetypeai/archetypeai-batch-examples-volve) for the JSONL converter.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `shape '[-1, 6912]' is invalid for input of size N` | Wrong number of `data_columns` | Must be exactly 9 columns |
| `n_neighbors must be <= number of samples in index` | N-shot files too small for `window_size` | Increase n-shot rows or decrease `window_size` |
| `could not parse as dtype i64` | Mixed int/float values in CSV | Ensure all sensor values are formatted as floats |
| `Failed to resolve file inputs` | File not uploaded | Upload file first via Files API |
| `Pipeline 'X' has no active versions` | Pipeline not deployed in this environment | Verify the pipeline_key is deployed (Machine State = `machine-state-classification`) |

## Best Practices

- **Start with quick tests** — use 200-row samples to verify pipeline works before full runs
- **Use ACTC labels for n-shots** — rig control system labels are more reliable than sensor heuristics
- **Optimize config first** — grid search over hyperparameters with small data before committing to a multi-hour full run
- **Poll status periodically** — jobs can run for minutes to hours depending on data size
- **Check events on failure** — the events endpoint provides detailed error messages
- **Match model expectations** — `omega_1_3_surface` expects exactly 9 sensor columns
- **Ensure enough n-shot windows** — `(rows - window_size) / step_size >= n_neighbors`

## Example Code

See [archetypeai-batch-examples-volve](https://github.com/archetypeai/archetypeai-batch-examples-volve) for full Python, shell, and curl implementations including data preparation, config optimization, and evaluation scripts.
