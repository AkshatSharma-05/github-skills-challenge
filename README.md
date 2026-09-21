# AIOps Pipeline Assessment
This project is a small Python example of monitoring a service and finding
possible problems in its telemetry.

## Service Being Monitored
The example monitors a `payment-service`. The sample data contains:
- response time in milliseconds
- CPU usage percentage
- memory usage percentage
- log level and message
- timestamp
The data is stored in `data/service_data.json`.

## Operational Problem
A payment service can become slow or unhealthy without someone noticing right
away. For example, a request might take too long, the service might use too
much CPU or memory, or its logs might show an error.
This project checks each telemetry record for those warning signs. When it
finds one, it creates an anomaly event that includes the service, timestamp,
and reasons for the alert.

## Purpose of AIOps
AIOps means using software to help monitor and operate applications. In this
assessment, AIOps is represented by the `AnomalyDetector` and the pipeline
around it. The goal is to automatically inspect service data, identify
anomalies, and pass the resulting events through an event flow instead of
requiring a person to inspect every record manually.

This is an educational simulation. The event topics are kept in memory, so it
does not require a real message broker or a running payment service.

## Project Components

- `src/aiops_pipeline.py` loads the data, runs anomaly detection, publishes
	detected events, and consumes them.
- `src/anomaly_detector.py` compares telemetry with response-time, CPU, and
	memory thresholds and checks the log level.
- `src/event_topic.py` provides a simple in-memory event topic.
- `src/event_producer.py` publishes anomaly events to a topic.
- `src/event_consumer.py` reads events from a topic.
- `data/service_data.json` contains the sample `payment-service` telemetry.
- `tests/test_aiops_pipeline.py` tests detection and event flow.
- `src/calculations.py` contains small practice calculation functions, and
	`tests/calculations_test.py` tests them.

## Task 2: Analysis of Logs and Metrics

This analysis is based on the records in `data/service_data.json`. There are
10 observations, and every observation is for the `payment-service`.

### 1. Fields That Represent Metrics
These numeric fields are metrics because they measure the service or its
resources:
- `response_time_ms` measures how long a request took, in milliseconds.
- `cpu_percent` measures CPU usage as a percentage.
- `memory_percent` measures memory usage as a percentage.

The `service` field identifies which service the record belongs to. It is not
a metric.

### 2. Fields That Represent Log Information
These fields represent log information:
- `log_level` shows the importance of the log message, such as `INFO` or
	`ERROR`.
- `message` describes what happened, such as a successful payment or a
	timeout.

### 3. How Timestamps Are Used
The `timestamp` field records when each observation happened. The values use
the format `YYYY-MM-DDTHH:MM:SS`, for example, `2026-09-20T10:00:00`. The records are in one-minute order from 10:00 to 10:09, which makes it possible to see how the service changes over time.

### 4. Observations That Look Normal
The records at 10:00, 10:01, 10:02, 10:03, and 10:04 look normal. They have:
- response times between 120 and 142 milliseconds
- CPU usage between 42% and 48%
- memory usage between 51% and 55%
- an `INFO` log level
- a message saying that the payment request was processed successfully

The records at 10:07, 10:08, and 10:09 also look normal. Their response
times are between 138 and 150 milliseconds, CPU usage is between 47% and 50%,
memory usage is between 55% and 57%, and they have successful `INFO` messages.

### 5. Observations That Look Unusual
The record at 10:05 is unusual. Its response time rises to 610 milliseconds,
and its message says `Payment service timeout`. It also has an `ERROR` log
level.

The record at 10:06 is the most unusual. Its response time is 640 milliseconds, CPU usage is 94%, memory usage is 91%, and its `ERROR` message
says `Database connection timeout`.

Together, the records show a short problem at 10:05 and 10:06. The values
return to the earlier range at 10:07, so the service appears to recover after
those two observations.

## Task 3: Identify Anomalies
The repository includes a detection component in `src/anomaly_detector.py`.
It is configured with default thresholds:
- `response_time_threshold = 500`
- `cpu_threshold = 80`
- `memory_threshold = 80`
The detector checks each record and creates an anomaly when at least one of
these conditions is true:
- response time is greater than 500 ms
- CPU usage is greater than 80%
- memory usage is greater than 80%
- a warning log is present

### Detection Result from the Provided Data
I ran the repository workflow with:

bash
python src/aiops_pipeline.py


The output showed:
- `Records processed: 10`
- `Anomalies detected: 2`
- `Events consumed: 0`
The anomalies were detected at:
- `2026-09-20T10:05:00` — `Payment service timeout`
- `2026-09-20T10:06:00` — `Database connection timeout`

### Relevant Metric and Log Information
For the first anomalous observation, the key values were:
- response time: 610 ms
- CPU: 75%
- memory: 70%
- log level: `ERROR`
- message: `Payment service timeout`

For the second anomalous observation, the key values were:
- response time: 640 ms
- CPU: 94%
- memory: 91%
- log level: `ERROR`
- message: `Database connection timeout`

These observations clearly stand out because the response time is far above
500 ms and system usage is high. They also include error-level log messages,
which are relevant to the service problem.

### Expected Anomaly Missed or Normal Event Incorrectly Flagged?
In this dataset, no normal observation was incorrectly flagged. The normal
records before 10:05 and after 10:06 stayed within the normal operating range.

### One Limitation and Improvement
A limitation is that the detection is based on fixed thresholds and a narrow
log check. That means it can miss important events if the service behaves
slightly differently or uses a different log severity name. A possible
improvement would be to broaden the log handling to include `ERROR` and other
severity levels, or to use threshold values that can be configured by service
team settings.

## Task 4: Verify the AIOps Event Flow
The repository contains a lightweight simulation of an event-streaming system.
The relevant components are:
- `EventProducer` — creates and sends the event
- `EventTopic` — stores messages in memory for the topic
- `EventConsumer` — receives messages from the topic
- `event` / `message` — the payload passed through the system

### Validation Steps
The event flow is implemented in `src/aiops_pipeline.py` and uses the following
logic:
1. load telemetry records from `data/service_data.json`
2. run `AnomalyDetector.detect(record)` on each record
3. if an anomaly is found, publish it through `EventProducer`
4. add the suspicious event to the producer topic
5. consume events using `EventConsumer`

### Verified Runtime Result
I used the repository workflow:
bash
python src/aiops_pipeline.py

This produced:
- `Records processed: 10`
- `Anomalies detected: 2`
- `Events consumed: 0`

This confirms that the anomaly detection portion works, but the event flow
currently does not complete as intended in the provided simulation. In the
code, the anomaly is published to a topic named `service-events`, while the
consumer reads from a separate topic named `anomaly-events`.
Because of that mismatch, the event is created and passed to the producer, but
it never reaches the consumer in the current implementation. This is an
important assessment issue in the flow.

### Role of Each Workflow Component
- `Producer`: sends anomaly events into the topic
- `Topic`: holds messages in memory for the pipeline
- `Consumer`: reads from the topic and processes stored messages
- `Event/message`: the actual information about the anomaly that moves through
  the pipeline

## Task 7: Reproduce and Record the Final Workflow
This project demonstrates an AIOps workflow for monitoring a payment service, identifying abnormal telemetry, and moving anomaly events through a simple event pipeline.
### Final workflow result
The final corrected workflow processed all 10 records, detected 2 anomalies, and consumed 2 events successfully.
### Issues identified and fixed
- Error logs were only checked for `WARNING`, but the dataset used `ERROR` values.
- The producer and consumer were using different topics, so events were never received.
### Limitation / improvement
The approach is rule-based and uses fixed thresholds. It works for this synthetic dataset, but a real deployment should use configurable thresholds and broader log parsing.
### Reproduce this demonstration
From the project root, run:

bash
python -m pytest -q
python src/aiops_pipeline.py

Expected output:
- tests pass
- records processed = 10
- anomalies detected = 2
- events consumed = 2
This is the final working AIOps flow for the repository.
