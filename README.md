# AIOps Assessment: Payment Service Monitoring

## Scenario

This assessment monitors a `payment-service` that processes payment requests. Each operational record contains service timing, CPU and memory utilization, log level, and a message describing the request outcome.

The operational problem is identifying service degradation early. Elevated response times, CPU or memory utilization, and relevant log signals can indicate payment timeouts or database connectivity problems that require investigation.

AIOps is used here to inspect operational telemetry, detect anomalous records, and turn those findings into events that can be consumed by downstream processing. The workflow is intentionally small and local so the assessment can focus on the component responsibilities and event flow.

## Component Map

- **Operational data:** [`data/service_data.json`](data/service_data.json) provides the payment-service telemetry records used by the workflow.
- **Metrics and logs:** Each data record contains `response_time_ms`, `cpu_percent`, `memory_percent`, `log_level`, and `message` fields. The records represent normal activity and a short period of payment-service degradation.
- **Anomaly detection:** [`src/anomaly_detector.py`](src/anomaly_detector.py) compares response time, CPU, and memory values with thresholds and creates an anomaly event when one or more checks match.
- **Event production:** [`src/event_producer.py`](src/event_producer.py) publishes detected anomaly events to a topic.
- **Event topics:** [`src/event_topic.py`](src/event_topic.py) provides the in-memory topic used to store and retrieve events.
- **Event consumption:** [`src/event_consumer.py`](src/event_consumer.py) reads the events from a topic for downstream handling.
- **Final AIOps processing:** [`src/aiops_pipeline.py`](src/aiops_pipeline.py) loads the operational data, runs detection, publishes detected events, reads from the configured consumer topic, and reports records processed, anomalies detected, and events consumed. The current assessment wiring uses separate `service-events` and `anomaly-events` topics, so the direct workflow detects 2 events but consumes 0 from the consumer topic.
- **Supporting utility:** [`src/calculations.py`](src/calculations.py) contains standalone circle-area and Fibonacci examples; it is not part of the AIOps workflow.

## Part 2: Operational Data Analysis

The analysis below is based on the 10 records in [`data/service_data.json`](data/service_data.json).

1. **Metrics:** `response_time_ms` measures request latency, `cpu_percent` measures CPU utilization, and `memory_percent` measures memory utilization. These numeric fields describe the service's operational performance and resource use.
2. **Log information:** `log_level` identifies the severity or category of the record (`INFO` or `ERROR`), and `message` contains the related human-readable event description. `service` identifies the emitting service, while `timestamp` identifies when the record was produced.
3. **Timestamp use:** Timestamps use ISO-like date-time values and progress in one-minute intervals from `2026-09-20T10:00:00` through `2026-09-20T10:09:00`. This ordering provides the timeline needed to see the short degradation and the subsequent recovery.
4. **Normal behaviour:** Records from 10:00 through 10:04 and from 10:07 through 10:09 appear normal. They have `INFO` logs, successful-processing messages, response times between 120 and 150 ms, CPU between 42% and 50%, and memory between 51% and 57%.
5. **Unusual behaviour:** The 10:05 record reports a 610 ms response time, an `ERROR` log, and a payment-service timeout. The 10:06 record reports a 640 ms response time, CPU at 94%, memory at 91%, an `ERROR` log, and a database connection timeout. These two consecutive records represent the incident window; the return to normal-looking values at 10:07 suggests recovery.

## Part 3: Anomaly Detection and Event Streaming Analysis

The provided [`src/anomaly_detector.py`](src/anomaly_detector.py) was used without replacing its threshold-based architecture. It processes all 10 records and flags a record when response time exceeds 500 ms, CPU exceeds 80%, or memory exceeds 80%.

### Detection Report

The two detected anomalies are:

- **2026-09-20T10:05:00, `payment-service`:** response time was 610 ms, compared with the 500 ms threshold. The source log was `ERROR` with the message `Payment service timeout`. The detector reason was `High response time`.
- **2026-09-20T10:06:00, `payment-service`:** response time was 640 ms, CPU was 94%, and memory was 91%. All three values exceeded their configured thresholds. The source log was `ERROR` with the message `Database connection timeout`. The detector reasons were `High response time`, `High CPU utilization`, and `High memory utilization`.

The result distinguishes the normal observations from the anomalous observations: the five normal records before the incident and three normal records after it produced no anomaly events, while the two incident records produced events. No normal event was incorrectly flagged based on the supplied data.

### Detection Review

The metric anomalies were detected as expected. The concerning `ERROR` log events were present in the source records, but the detector did not add a log-related reason because its log check currently looks for `WARNING` rather than the dataset's `ERROR` level. This is an expected anomaly signal that was missed by the detector's log rule, although both records were still flagged by their metrics.

The direct detector events include timestamps, service names, source records, and reasons, which makes them readable and explains why each record was flagged. The final pipeline output currently reports zero consumed events because [`src/aiops_pipeline.py`](src/aiops_pipeline.py) publishes to `service-events` while its consumer reads `anomaly-events`; this prevents those detected events from appearing in the final consumed-event listing.

One limitation is that the detector relies on fixed point-in-time thresholds and does not correlate log severity or message content with the metrics. A useful improvement would be to treat `ERROR` records as explicit evidence and combine them with threshold breaches while preserving the existing detector and event architecture.

## Running the Workflow

From the repository root:

```bash
python src/aiops_pipeline.py
```

The workflow reads [`data/service_data.json`](data/service_data.json) and prints the detected anomaly events. The provided tests can be run with:

```bash
pytest
```

## Assessment Structure

The original application components and directory structure are preserved. The assessment focuses on understanding how telemetry moves from operational data through detection, event production, event topics, event consumption, and final AIOps reporting.


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

