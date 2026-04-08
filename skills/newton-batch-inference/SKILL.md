---
name: newton-batch-inference
description: >
  Create and monitor batch processing jobs on the Archetype AI Newton platform.
  Use when the user wants to run inference on large datasets asynchronously,
  classify sensor data at scale using the Machine State pipeline, or run
  Newton's Nano Inference on uploaded files. This skill covers job creation,
  status polling, and event log retrieval.
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

## Available Pipelines

### Pipeline 1: Machine State Job Pipeline

Classifies time-series sensor data using n-shot examples. Uses Newton to vectorize sensor windows, then a KNN classifier to predict state.

**Pipeline key:** `machine-state-job-pipeline`

**Available model types:**
- `omega_1_3_slb_surface` — surface sensor monitoring (expects 9 sensor channels)
- `omega_1_3_slb_power_drive` — downhole power drive monitoring (expects 9 sensor channels)

**Input ports:**
- `worker.inference` — files to classify
- `worker.n_shots` — labeled example files with `metadata.class`

**Data requirements:**
- CSV files must include a `timestamp` column (Unix time)
- `data_columns` must specify exactly which columns to use as sensor channels
- Models expect 9 sensor channels — using more or fewer causes shape errors
- `window_size` controls how many time steps per classification window
- N-shot files need enough rows to produce windows: `(rows - window_size) / step_size` must be >= `n_neighbors`

```bash
curl -s -X POST "$BASE_URL/jos/jobs" \
  -H "Authorization: Bearer $ATAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-classification-job",
    "pipeline_type": "batch",
    "pipeline_key": "machine-state-job-pipeline",
    "inputs": {
      "worker.inference": [{"file_id": "sensor_data.csv"}],
      "worker.n_shots": [
        {"file_id": "normal_examples.csv", "metadata": {"class": "normal"}},
        {"file_id": "anomaly_examples.csv", "metadata": {"class": "anomaly"}}
      ]
    },
    "parameters": {
      "worker": {
        "parallelism": 1,
        "config": {
          "model_type": "omega_1_3_slb_surface",
          "batch_size": 8,
          "timestamp_column": "timestamp",
          "data_columns": ["col_1", "col_2", "col_3", "col_4", "col_5", "col_6", "col_7", "col_8", "col_9"],
          "reader_config": {"window_size": 64, "step_size": 1},
          "classifier_config": {"n_neighbors": 3, "metric": "euclidean", "weights": "uniform"},
          "flush_every_n_iteration": 1000
        }
      }
    }
  }'
```

### Pipeline 2: Nano Inference Pipeline

General-purpose inference using Newton's language generation on input data files.

**Pipeline key:** `nano-inference-pipeline`

**Input ports:**
- `worker.data` — files to run inference on

```bash
curl -s -X POST "$BASE_URL/jos/jobs" \
  -H "Authorization: Bearer $ATAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-inference-job",
    "pipeline_type": "batch",
    "pipeline_key": "nano-inference-pipeline",
    "inputs": {
      "worker.data": [{"file_id": "my_data.csv"}]
    },
    "parameters": {
      "worker": {
        "parallelism": 1,
        "config": {
          "generation": {
            "do_sample": true,
            "max_new_tokens": 256,
            "repetition_penalty": 1,
            "temperature": 0.7,
            "top_k": 20,
            "top_p": 0.8
          }
        }
      }
    }
  }'
```

## Monitoring Jobs

### Check Status

```bash
curl -s "$BASE_URL/jos/jobs/$JOB_ID" \
  -H "Authorization: Bearer $ATAI_API_KEY"
```

Status lifecycle: `PENDING` → `RUNNING` → `COMPLETED` / `FAILED` / `CANCELLED`

### View Events/Logs

```bash
curl -s "$BASE_URL/jos/jobs/$JOB_ID/events" \
  -H "Authorization: Bearer $ATAI_API_KEY"
```

Returns timestamped events with `level` (INFO/ERROR) and `message`:
```json
{
  "events": [
    {"event_type": "running_job", "level": "INFO", "message": "Running job"},
    {"event_type": "vectorizing_file", "level": "INFO", "message": "Vectorizing file data.csv - Done 0.3 sec"},
    {"event_type": "info", "level": "INFO", "message": "Saving chunk with result for data.csv - Done 0.1 sec"}
  ]
}
```

### List All Jobs

```bash
curl -s "$BASE_URL/jos/jobs?limit=10&offset=0" \
  -H "Authorization: Bearer $ATAI_API_KEY"
```

## API Reference

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v0.5/jos/jobs` | POST | Create a batch job |
| `/v0.5/jos/jobs` | GET | List all jobs |
| `/v0.5/jos/jobs/{job_id}` | GET | Get job status |
| `/v0.5/jos/jobs/{job_id}/events` | GET | Get job events/logs |

## Fine-Tuning (TBD)

Fine-tuning via `POST /v0.5/internal/experiment/runner/jobs` is not yet available in production. It requires JSONL-formatted training data. See [archetype-batch-examples](https://github.com/archetypeai/archetype-batch-examples) for the JSONL converter and dataset schema.

## Best Practices

- **Poll status periodically** — jobs can run for minutes to hours depending on data size
- **Check events on failure** — the events endpoint provides detailed error messages
- **Match model expectations** — `omega_1_3_slb_surface` expects exactly 9 sensor columns
- **Ensure enough n-shot windows** — with `window_size=64` and `step_size=1`, you need at least `n_neighbors + window_size` rows per n-shot file
- **Use `flush_every_n_iteration`** — controls how often intermediate results are saved

## Example Code

See [archetype-batch-examples](https://github.com/archetypeai/archetype-batch-examples) for full Python, shell, and curl implementations.
