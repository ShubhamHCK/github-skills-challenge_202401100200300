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

## Reproduce the Demonstration

From a Python 3 environment, run the following commands from the repository root:

```bash
git clone https://github.com/ShubhamHCK/github-skills-challenge_202401100200300.git
cd github-skills-challenge_202401100200300
python3 -m pip install -r requirements.txt
python3 -m pytest -q
python3 src/aiops_pipeline.py
```

The tests validate the detector, event producer, topic, consumer, and corrected pipeline.
The final command reads `data/service_data.json` and prints the two detected and consumed
anomaly events. No external event broker is required because `EventTopic` is an in-memory
simulation.

## Operational Data Analysis

The supplied `data/service_data.json` contains 10 observations for `payment-service`,
covering 2026-09-20 from 10:00:00 through 10:09:00 at one-minute intervals.

1. **Metrics:** `response_time_ms` is request latency in milliseconds; `cpu_percent` and
	`memory_percent` are resource-utilization percentages. These numeric fields describe
	measurable service and infrastructure behavior.
2. **Log information:** `log_level` identifies the severity or level of the log entry, and
	`message` contains its human-readable event description. The `service` field identifies
	the source of both the metrics and logs.
3. **Timestamps:** `timestamp` records when each observation occurred. The chronological,
	one-minute spacing makes it possible to see the short degradation at 10:05-10:06 and
	the return to normal levels from 10:07 onward.
4. **Normal behavior:** The records from 10:00-10:04 and 10:07-10:09 appear normal. They
	have `INFO` logs reporting successful processing, response times from 120 to 150 ms,
	CPU from 42% to 50%, and memory from 51% to 57%.
5. **Unusual behavior:** The 10:05 record reports a payment-service timeout with an
	`ERROR` log and response time of 610 ms. The 10:06 record is more severe: it reports a
	database connection timeout, response time of 640 ms, CPU at 94%, and memory at 91%.
	These records combine failure-related log messages with degraded latency and, at
	10:06, high resource utilization, so they are the likely anomaly window.

## Anomaly Detection Report

The provided `AnomalyDetector` uses thresholds of 500 ms for response time, 80% for CPU,
and 80% for memory. Running the provided pipeline processes all 10 observations and
produces 2 anomaly events:

- **10:05:00:** flagged for high response time (`610 ms`; threshold `500 ms`). The source
  log is `ERROR` with the message `Payment service timeout`; CPU is 75% and memory is 70%,
  so neither resource threshold is exceeded.
- **10:06:00:** flagged for high response time (`640 ms`), high CPU utilization (`94%`),
  and high memory utilization (`91%`). The source log is `ERROR` with the message
  `Database connection timeout`.

The remaining eight observations are classified as normal: their metrics stay below the
configured thresholds and their logs are `INFO` messages reporting successful processing.
No normal observation was incorrectly flagged based on these rules.

The detector now includes both `ERROR` and `WARNING` as concerning log levels. The two
incident records therefore include their log signal as well as their metric reasons. A
remaining limitation is that detection uses fixed per-record thresholds and does not
correlate trends across a longer time window.

## Event Flow Verification

The event-stream components have these roles:

- **Event/message:** the dictionary produced by `AnomalyDetector`, containing the event
  type, timestamp, service, source record, and detection reasons.
- **Producer:** `EventProducer` accepts a detected event and publishes it to its configured
  topic.
- **Topic:** `EventTopic` is the in-memory stream that stores published messages and makes
  them available to consumers.
- **Consumer:** `EventConsumer` reads messages from its configured topic. In the pipeline,
  the consumed event is the downstream AIOps result that is returned and printed for
  handling or reporting.

The initial workflow execution against `data/service_data.json` produced this result:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 0
```

Investigation identified a topic-wiring defect in `src/aiops_pipeline.py`: it created a
`service-events` topic for the producer and a separate `anomaly-events` topic for the
consumer. The producer was publishing successfully, but the consumer was listening to an
empty topic. The correction was to create one shared `anomaly-events` topic for both
components.

Investigation also identified a detector defect in `src/anomaly_detector.py`: it checked
only for `WARNING`, while the concerning records in this dataset use `ERROR`. The
correction was to recognize both levels and add a log reason to the event.

The corrected workflow was executed again with this result:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2
```

It printed both downstream events:

```text
10:05:00: High response time, Concerning log level detected
10:06:00: High response time, High CPU utilization, High memory utilization,
		  Concerning log level detected
```

Regression validation also found that the modules used script-only imports, causing
`tests/test_aiops_pipeline.py` to fail when importing `src.aiops_pipeline`. The affected
pipeline, producer, and consumer modules now support both package imports and direct
script execution. The focused suite passes with 6 tests, and a package-level pipeline
assertion confirms the two detected events equal the two consumed events.

## End-to-End Pipeline Execution

Task 6 was completed by running `python3 src/aiops_pipeline.py` after the corrections. The
execution verified the complete flow:

`Operational data -> AnomalyDetector -> anomaly event -> EventProducer -> anomaly-events
topic -> EventConsumer -> AIOps output`

The final result was:

```text
Records processed: 10
Anomalies detected: 2
Events consumed: 2
```

The final output successfully reported the detected payment-service issues at 10:05 and
10:06, including high response time, high CPU and memory utilization where applicable,
and the concerning log-level signal. The complete regression suite also passed: `10
passed`.

Your challenge is ready.
Follow the instructions provided for this challenge and complete the required tasks in this repository.

Make sure your work is committed and pushed to your repository before submission.

Good luck!


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

