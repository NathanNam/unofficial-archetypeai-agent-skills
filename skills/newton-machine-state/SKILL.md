---
name: newton-machine-state
description: >
  Build n-shot classification pipelines using Newton's Machine State Lens.
  Use when the user wants to classify sensor data into categories (e.g.,
  stressed vs. relaxed, normal vs. anomaly, idle vs. active), perform
  anomaly detection on time-series data, or implement n-shot learning
  with CSV sensor data. This skill covers session lifecycle, focus CSV
  uploads, sliding window configuration, and SSE result parsing.
  Do NOT use for vision-based analysis or chart reading (use newton-activity-monitor).
  Do NOT use for initial API setup (use newton-setup).
---

# Newton Machine State Lens

Classify time-series sensor data using n-shot learning. The Machine State Lens uses labeled CSV examples (focus files) to classify new data streams via KNN over sliding windows.

## When to Apply

- User wants to classify sensor data into discrete states
- User asks about anomaly detection or state monitoring
- User wants n-shot classification without training a model
- User is building a real-time monitoring dashboard with status indicators
- User references HRV stress detection or OBD2 vehicle health patterns

## Lens ID

```
lns-1d519091822706e2-bc108andqxf8b4os
```

## Core Concepts

### N-Shot Classification

Provide labeled CSV examples (focus files) — one per class — and Newton classifies new data against those examples. No model training required.

### Sliding Windows

Newton processes CSV data in overlapping windows:
- **window_size**: Number of rows per window (default: 16)
- **step_size**: Rows to advance between windows (default: 8)
- Expected windows = `floor((dataPoints - windowSize) / stepSize) + 1`

## Workflow

### Step 1: Prepare Focus CSVs

Create one CSV per class with labeled sensor data. Each CSV should have:
- Consistent column names matching your data stream
- A representative sample of that class (50-200 rows recommended)
- No header row mismatches between focus files and query data

**Example: HRV Stress Detection**
```
# focus_relaxed.csv
rmssd,sdnn,mean_hr,pnn50,sd1
42.5,45.2,68.3,22.1,30.1
...

# focus_stressed.csv
rmssd,sdnn,mean_hr,pnn50,sd1
18.2,22.1,95.7,8.3,12.9
...
```

**Example: Vehicle Health Monitoring**
```
# focus_normal.csv
rpm,speed,coolant_temp,iat,engine_load,throttle,map
800,0,90,35,18,12,34
...

# focus_attention.csv
rpm,speed,coolant_temp,iat,engine_load,throttle,map
850,0,108,52,45,18,42
...
```

### Step 2: Upload Focus Files (One-Time)

```typescript
async function uploadFile(filePath: string, content: string): Promise<string> {
  const formData = new FormData();
  formData.append("file", new Blob([content], { type: "text/csv" }), filePath);

  const response = await newtonFetch("/files", {
    method: "POST",
    body: formData,
  });
  const data = await response.json();
  return data.file_id;
}
```

Cache the returned `file_id` values — focus files only need to be uploaded once per session.

### Step 3: Create a Session

```typescript
const response = await newtonFetch("/lens/sessions/create", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ lens_id: "lns-1d519091822706e2-bc108andqxf8b4os" }),
});
const { session_id } = await response.json();
```

### Step 4: Configure Session with Focus Files

Send a `session.modify` event with n-shot focus file IDs and CSV config:

```typescript
await newtonFetch("/lens/sessions/events/process", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    session_id,
    events: [{
      kind: "session.modify",
      payload: {
        input_n_shot: [
          { label: "relaxed", file_id: relaxedFileId },
          { label: "stressed", file_id: stressedFileId },
        ],
        csv_configs: {
          window_size: 16,
          step_size: 8,
        },
      },
    }],
  }),
});
```

### Step 5: Connect SSE Consumer (Before Input!)

**Critical:** Always connect the SSE consumer before setting the input stream to avoid missing results.

```typescript
const sseResponse = await newtonFetch(
  `/lens/sessions/consumer/${sessionId}`,
  { headers: { Accept: "text/event-stream" } }
);
```

### Step 6: Set Input Stream

Upload query CSV and set as input:

```typescript
const queryFileId = await uploadFile("query.csv", csvContent);

await newtonFetch("/lens/sessions/events/process", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    session_id: sessionId,
    events: [{
      kind: "input_stream.set",
      payload: {
        streams: [{
          id: "input-0",
          source: { type: "csv_file_reader", file_id: queryFileId },
        }],
      },
    }],
  }),
});
```

### Step 7: Parse SSE Results

Each `inference.result` event contains a classification for one window:

```typescript
// SSE event data shape
{
  "kind": "inference.result",
  "payload": {
    "result": ["stressed", { "stressed": 0.85, "relaxed": 0.15 }]
  }
}
```

Aggregate across windows — majority vote or average scores:

```typescript
function aggregateResults(results: Array<[string, Record<string, number>]>) {
  const totals: Record<string, number> = {};
  for (const [label, scores] of results) {
    for (const [key, score] of Object.entries(scores)) {
      totals[key] = (totals[key] || 0) + score;
    }
  }
  const count = results.length;
  return Object.fromEntries(
    Object.entries(totals).map(([k, v]) => [k, v / count])
  );
}
```

### Step 8: Early SSE Termination (Optimization)

Calculate expected window count and close the stream early instead of waiting for the idle timeout (60-80s):

```typescript
const expectedWindows = Math.floor((dataPoints - windowSize) / stepSize) + 1;
let receivedWindows = 0;

// In SSE parse loop:
if (++receivedWindows >= expectedWindows) {
  reader.cancel(); // Close early
}
```

## Session Lifecycle

```
Upload Focus CSVs (once) → Create Session (once) → [Query Loop: Upload CSV → Set Input → Read SSE Results] → Destroy Session
```

Sessions are reusable — create once, query many times. Only recreate on error.

### Cleanup

```typescript
// Destroy session
await newtonFetch("/lens/sessions/destroy", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ session_id: sessionId }),
});

// Delete uploaded files
await newtonFetch(`/files/delete/${fileId}`, { method: "DELETE" });
```

## NewtonStreamManager Singleton Pattern

For real-time applications, implement a server-side singleton that:

1. Buffers incoming sensor data (max ~300 points)
2. Runs periodic queries every 15 seconds
3. Manages session lifecycle (create on first query, reuse, recreate on error)
4. Pushes results to frontend via SSE

See [references/stream-manager-pattern.md](references/stream-manager-pattern.md) for the full implementation pattern.

## Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `window_size` | 16 | Rows per sliding window |
| `step_size` | 8 | Row stride between windows |
| `MAX_BUFFER_SIZE` | 300 | Max sensor readings to buffer |
| `MIN_DATA_POINTS` | 20-32 | Minimum points before first query |
| `QUERY_INTERVAL` | 15000ms | Time between periodic queries |

## Best Practices

- **Focus CSV quality matters more than quantity** — 50-200 well-labeled rows per class is sufficient
- **Column names must match** between focus files and query data exactly
- **Reuse sessions** — creating sessions has overhead; create once and query repeatedly
- **Always connect SSE before setting input** — otherwise results may be missed
- **Implement early termination** — avoids 60-80s idle wait per query
- **Graceful degradation** — apps should work without Newton; check availability first
