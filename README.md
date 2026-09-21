# GitHub Challenge

<img src="https://octodex.github.com/images/Professortocat_v2.png" align="right" height="200px" />

## AIOps Assessment

This repository monitors a simulated `payment-service`. The sample telemetry contains
response times, CPU and memory utilization, log levels, messages, and timestamps for
individual service observations.

The operational problem is detecting service degradation early. Slow requests, resource
pressure, and warning or error-level logs can indicate incidents such as timeouts or
database connection problems, but reviewing each telemetry record manually is slow and
inconsistent.

AIOps is used here to combine these operational signals, identify anomalous records using
defined thresholds and log information, and turn the findings into events that can be
processed by downstream systems. This assessment focuses on the detection and event-flow
parts of that scenario rather than a production monitoring platform.

## Components

- `data/service_data.json` contains the sample `payment-service` telemetry.
- `src/anomaly_detector.py` checks each record for high response time, CPU or memory
	utilization, and relevant log levels, then creates an anomaly event when needed.
- `src/event_topic.py` provides the in-memory event-stream simulation used by the workflow.
- `src/event_producer.py` publishes detected anomaly events to a topic.
- `src/event_consumer.py` reads the published events for downstream handling.
- `src/aiops_pipeline.py` loads the data, coordinates detection and event processing, and
	reports records processed, anomalies detected, and events consumed.
- `src/calculations.py` contains standalone calculation examples included in the assessment.
- `tests/` contains tests for the detector, event flow, pipeline behavior, and calculations.

## Run the workflow

From the repository root, run:

```bash
python3 src/aiops_pipeline.py
```

The command reads the sample telemetry, detects anomalies, publishes them to the in-memory
topic, and prints the consumed events.

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

