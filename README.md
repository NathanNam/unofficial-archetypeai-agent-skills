# Unofficial Archetype AI Agent Skills

> **Disclaimer:** This is an unofficial, community-maintained project. It is not affiliated with, endorsed by, or maintained by [Archetype AI](https://www.archetypeai.dev/).

Agent skills for building applications with [Archetype AI's Newton](https://www.archetypeai.dev/) — a real-time sensor intelligence platform that understands physical world data through foundation models.

Inspired by [mongodb/agent-skills](https://github.com/mongodb/agent-skills).

## Skills

| Skill | Description |
|-------|-------------|
| [newton-setup](skills/newton-setup/) | Configure Newton API access, environment setup, and SDK initialization |
| [newton-machine-state](skills/newton-machine-state/) | N-shot classification of sensor data using the Machine State Lens |
| [newton-activity-monitor](skills/newton-activity-monitor/) | Vision-based analysis and Q&A using the Activity Monitor Lens |
| [newton-sensor-streaming](skills/newton-sensor-streaming/) | Real-time sensor data ingestion patterns (BLE, OBD2, serial, etc.) |
| [newton-batch-upload](skills/newton-batch-upload/) | Upload large files (> 255 MB) via multipart presigned URLs |
| [newton-batch-inference](skills/newton-batch-inference/) | Create and monitor asynchronous batch processing jobs |

## Quick Start

### Claude Code

```bash
# Add as global skills (available in all projects)
cp -r skills/* ~/.claude/skills/

# Or add to a specific project
cp -r skills/* your-project/.claude/skills/
```

### Invoke a Skill

```
/newton-setup            # Set up API access
/newton-machine-state    # Build a classification pipeline
/newton-activity-monitor # Analyze visual data
/newton-sensor-streaming # Connect hardware sensors
/newton-batch-upload     # Upload large files (> 255 MB)
/newton-batch-inference  # Run batch processing jobs
```

## Architecture

Newton operates through **Lens Sessions** — persistent inference pipelines that process sensor data in real-time:

```
Real-time:  Sensor Data → Upload File → Create Session → Set Input Stream → SSE Results
                                              ↑
                                      Focus/N-shot Examples

Batch:      Upload Files → Create Batch Job → Poll Status → View Results
```

### Two Core Lenses

| Lens | ID | Input | Output |
|------|----|-------|--------|
| **Machine State** | `lns-1d519091822706e2-bc108andqxf8b4os` | CSV time-series | Classification labels + confidence scores |
| **Activity Monitor** | `lns-fd669361822b07e2-bc608aa3fdf8b4f9` | Video | Natural language text |

- **Machine State Lens** — Classifies time-series sensor data using n-shot learning (e.g., stressed vs. relaxed, normal vs. anomaly). Provide labeled CSV examples and it classifies new data via KNN over sliding windows.
- **Activity Monitor Lens** — Analyzes visual data (charts, dashboards, camera feeds) and answers natural language questions using a 2B-parameter vision model.

## Example Projects

These projects demonstrate the patterns covered by these skills:

- [corsense-hrv](https://github.com/NathanNam/corsense-hrv) — Real-time HRV stress detection using BLE heart rate monitors + Newton Machine State & Activity Monitor
- [obd2-scanner](https://github.com/NathanNam/obd2-scanner) — Browser-based vehicle diagnostics via OBD2/ELM327 + Newton health classification & chat

## API Base URL

```
https://api.u1.archetypeai.app/v0.5
```

## License

Apache-2.0
