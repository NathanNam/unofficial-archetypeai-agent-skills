# Newton API Reference

## Base URL

```
https://api.u1.archetypeai.app/v0.5
```

## Authentication

All endpoints require:
```
Authorization: Bearer ${ATAI_API_KEY}
```

## Endpoints

### File Management

#### Upload File
```
POST /files
Content-Type: multipart/form-data

Body: file (binary) — CSV or video file
Response: { "file_id": "string" }
```

#### Delete File
```
DELETE /files/delete/{file_id}
Response: 200 OK
```

### Lens Sessions

#### Create Session
```
POST /lens/sessions/create
Content-Type: application/json

Body: { "lens_id": "string" }
Response: { "session_id": "string" }
```

#### Process Events
```
POST /lens/sessions/events/process
Content-Type: application/json

Body: {
  "session_id": "string",
  "events": [
    {
      "kind": "session.modify" | "input_stream.set" | "output_stream.set",
      "payload": { ... }
    }
  ]
}
```

#### SSE Consumer
```
GET /lens/sessions/consumer/{session_id}
Accept: text/event-stream

Response: Server-Sent Events stream
```

#### Destroy Session
```
POST /lens/sessions/destroy
Content-Type: application/json

Body: { "session_id": "string" }
```

## Event Types

### session.modify

Configure a session with focus data and parameters.

**Machine State Lens:**
```json
{
  "kind": "session.modify",
  "payload": {
    "input_n_shot": [
      { "label": "class_a", "file_id": "uploaded_file_id" },
      { "label": "class_b", "file_id": "uploaded_file_id" }
    ],
    "csv_configs": {
      "window_size": 16,
      "step_size": 8
    }
  }
}
```

**Activity Monitor Lens:**
```json
{
  "kind": "session.modify",
  "payload": {
    "focus": "Domain context text describing what the visual shows",
    "instruction": "User's natural language question"
  }
}
```

### input_stream.set

Set the input data source for a session.

**CSV input (Machine State):**
```json
{
  "kind": "input_stream.set",
  "payload": {
    "streams": [{
      "id": "input-0",
      "source": { "type": "csv_file_reader", "file_id": "uploaded_file_id" }
    }]
  }
}
```

**Video input (Activity Monitor):**
```json
{
  "kind": "input_stream.set",
  "payload": {
    "streams": [{
      "id": "input-0",
      "source": { "type": "video_file_reader", "file_id": "uploaded_file_id" }
    }]
  }
}
```

### output_stream.set

Configure where results are delivered.

```json
{
  "kind": "output_stream.set",
  "payload": {
    "streams": [{
      "id": "output-0",
      "sink": { "type": "sse" }
    }]
  }
}
```

## SSE Event Format

### inference.result (Machine State)
```json
{
  "kind": "inference.result",
  "payload": {
    "result": ["label", { "class_a": 0.85, "class_b": 0.15 }]
  }
}
```

### inference.result (Activity Monitor)
```json
{
  "kind": "inference.result",
  "payload": {
    "result": "Natural language response text"
  }
}
```

### sse.stream.end
Indicates the stream has completed. Often arrives 60-80 seconds after last result — use early termination to avoid waiting.

## Known Lens IDs

| Lens | ID | Input | Output |
|------|----|-------|--------|
| Machine State | `lns-1d519091822706e2-bc108andqxf8b4os` | CSV | Classification labels + scores |
| Activity Monitor | `lns-fd669361822b07e2-bc608aa3fdf8b4f9` | Video | Natural language text |
