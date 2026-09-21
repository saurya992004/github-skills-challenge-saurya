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

