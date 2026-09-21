# AIOps Assessment: Payment Service Monitoring

## Part 1: AIOps Scenario

The service in this example is a `payment-service`. It processes payment requests. I am using the data to see if the service is working normally or having problems.

The problem is that slow requests, high resource use, or error messages can show that payments are failing. The aim is to find this early.

AIOps is used to check the service data, find unusual records, and send the findings as events. In this assessment it is a small Python workflow, not a real production monitoring system.

### Main Components

- **Data:** [`data/service_data.json`](data/service_data.json) contains the payment-service records.
- **Detector:** [`src/anomaly_detector.py`](src/anomaly_detector.py) checks the metrics and log level and creates an anomaly event.
- **Producer:** [`src/event_producer.py`](src/event_producer.py) sends an anomaly event to a topic.
- **Topic:** [`src/event_topic.py`](src/event_topic.py) is a small in-memory place where events are stored.
- **Consumer:** [`src/event_consumer.py`](src/event_consumer.py) reads the events from the topic.
- **Pipeline:** [`src/aiops_pipeline.py`](src/aiops_pipeline.py) runs all the steps and prints the final result.
- [`src/calculations.py`](src/calculations.py) is a separate utility example and is not used by the AIOps flow.

## Part 2: Operational Data Analysis

I inspected the 10 records in [`data/service_data.json`](data/service_data.json).

1. **Metrics:** `response_time_ms` is the request time. `cpu_percent` is CPU use and `memory_percent` is memory use.
2. **Logs:** `log_level` is the log type, such as `INFO` or `ERROR`. `message` explains what happened. `service` says which service sent the record.
3. **Timestamps:** `timestamp` shows when each record happened. The records are one minute apart, from 10:00 to 10:09 on 2026-09-20. This makes the problem easy to follow in time order.
4. **Normal records:** 10:00 to 10:04 and 10:07 to 10:09 look normal. They have successful `INFO` messages, response times from 120 to 150 ms, CPU from 42% to 50%, and memory from 51% to 57%.
5. **Unusual records:** At 10:05 the response time was 610 ms and the log said payment service timeout. At 10:06 the response time was 640 ms, CPU was 94%, memory was 91%, and the log said database connection timeout. The values return to normal at 10:07, so this looks like a short problem.

## Part 3: Anomaly Detection and Event Streaming Analysis

I used the provided [`src/anomaly_detector.py`](src/anomaly_detector.py). It checks all 10 records. It flags a record when response time is over 500 ms, CPU is over 80%, memory is over 80%, or the log level is `ERROR`.

### Detection Report

The two detected anomalies are:

- **2026-09-20T10:05:00, `payment-service`:** response time was 610 ms, compared with the 500 ms threshold. The source log was `ERROR` with the message `Payment service timeout`. The detector reasons were `High response time` and `Error log detected`.
- **2026-09-20T10:06:00, `payment-service`:** response time was 640 ms, CPU was 94%, and memory was 91%. All three values exceeded their configured thresholds. The source log was `ERROR` with the message `Database connection timeout`. The detector reasons were `High response time`, `High CPU utilization`, `High memory utilization`, and `Error log detected`.

The five normal records before the problem and the three records after it were not flagged. The two problem records were flagged. I did not see a normal record being incorrectly flagged.

### Detection Review

The detector found both the bad metrics and the `ERROR` logs. This was the expected result.

Each event includes the time, service, original record, and reasons. This explains why it was flagged. The producer and consumer use the same `anomaly-events` topic, so the events appear in the final output.

One limitation is that the detector uses fixed thresholds. Different services may need different threshold values.

## Task 5: Troubleshoot and Correct the Workflow

I found two problems that stopped the full workflow from working correctly.

1. **Anomaly detector problem**
	- Component: [`src/anomaly_detector.py`](src/anomaly_detector.py)
	- Cause: the data uses `ERROR`, but the detector was checking for `WARNING`.
	- Correction: I changed the check to `ERROR`.
	- Verification: I ran the pipeline again. The two error messages were included as `Error log detected` reasons.

2. **Event topic problem**
	- Component: [`src/aiops_pipeline.py`](src/aiops_pipeline.py)
	- Cause: the producer used `service-events`, but the consumer used a separate `anomaly-events` topic. The consumer therefore received nothing.
	- Correction: I made the producer and consumer use the same `anomaly-events` topic.
	- Verification: I ran the pipeline again and the number of consumed events changed from 0 to 2.

The corrections keep the existing detector, producer, topic, consumer, and pipeline components. The final run processed 10 records, detected 2 anomalies, and consumed 2 events.

## Task 4: Verify the AIOps Event Flow

I ran the provided pipeline after connecting the producer and consumer to the same `anomaly-events` topic. The roles are:

- **Event:** an anomaly made by the detector. It contains the timestamp, service, type, reasons, and original record.
- **Producer:** [`src/event_producer.py`](src/event_producer.py) sends the event to the topic.
- **Topic:** [`src/event_topic.py`](src/event_topic.py) stores the event in memory.
- **Consumer:** [`src/event_consumer.py`](src/event_consumer.py) gets the event from the topic.
- **AIOps component:** [`src/aiops_pipeline.py`](src/aiops_pipeline.py) runs the whole process and prints the result.

The execution result was:

- 10 operational records processed.
- 2 anomaly events created by the detector.
- The producer published both events to `anomaly-events`.
- The consumer received both events from `anomaly-events`.
- The final AIOps output displayed both payment-service problems, including the response time, CPU, memory, and `ERROR` log reasons.

So the event flow was: operational data -> detector -> event -> producer -> topic -> consumer -> AIOps output.

## Task 6: End-to-End Pipeline Result

I executed the full pipeline with:

```bash
python src/aiops_pipeline.py
```

The output showed:

- 10 records were processed from the operational data file.
- 2 unusual records were detected.
- An anomaly event was made for each unusual record.
- The producer published the events to the `anomaly-events` topic.
- The consumer received 2 events from the topic.
- The AIOps pipeline processed the received events and printed their details.
- The final output showed the payment service timeout and database connection timeout, with the reasons for each problem.

This confirms the complete path: operational data -> anomaly detection -> event -> producer -> topic -> consumer -> AIOps result.

## Task 8: Validation Result

I ran the validation that is provided in the repository:

```bash
python -m pytest
python src/aiops_pipeline.py
```

The checks passed. There were 8 passing tests. The full workflow also completed successfully:

- 10 operational records were processed.
- 2 anomalies were detected.
- 2 anomaly events were generated and moved through the topic.
- 2 events were consumed and processed.
- The final output showed the payment timeout and database connection timeout.

This confirms that the data processing, anomaly detection, event pipeline, consumer, and final AIOps result are working.

## How to Reproduce the Demonstration

From the repository root, another user can do these steps:

1. Install the packages with `pip install -r requirements.txt`.
2. Run the full workflow with:

	```bash
	python src/aiops_pipeline.py
	```

3. Check that the output says 10 records processed, 2 anomalies detected, and 2 events consumed.
4. Run the tests with:

	```bash
	python -m pytest
	```

The workflow reads [`data/service_data.json`](data/service_data.json) and prints the detected anomaly events.

## Assessment Structure

The original application components and directory structure are preserved. The assessment focuses on understanding how telemetry moves from operational data through detection, event production, event topics, event consumption, and final AIOps reporting.


---

&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

