# Spanner Async Load Generator 🚀

This is a Java-based command-line tool designed to generate configurable, asynchronous load on a Google Cloud Spanner database. It utilizes `picocli` for command-line parsing and is built to test database performance under various traffic patterns like steady, stepped, and burst QPS.

**Version 2.0.0** - Now with OpenTelemetry metrics and Dynamic Channel Pool (DCP) support!

### Features

  * **Asynchronous Operations**: Leverages Spanner's async client for high throughput.
  * **Multiple Workloads**: Supports `READ`, `WRITE`, and mixed `READ_WRITE` workloads.
  * **Dynamic Channel Pool (DCP)**: Uses `java-spanner` 6.105.0 with `grpc-gcp-java` for automatic gRPC channel scaling (requires explicit enablement via `enableDynamicChannelPool()`).
  * **OpenTelemetry Metrics**: Exports metrics to Google Cloud Monitoring for performance analysis.
  * **Configurable QPS**:
      * Start with an initial QPS (`--start-qps`).
      * Gradually increase QPS over time (`--step-qps`, `--interval-seconds`).
      * Set a maximum QPS ceiling (`--end-qps`).
      * Simulate a sudden traffic spike with a dedicated **burst mode** (`--burst`).
  * **Initial Data Population**: Can pre-load the target table with a specified number of rows (`--num-load-rows`).

-----

## Building the Project

This project uses Maven. To build the executable JAR, run the following command from the project's root directory:

```bash
mvn clean install
```

This will create a JAR file in the `target/` directory (e.g., `spanner-load-generator-1.0-SNAPSHOT.jar`).

-----

## Command-Line Arguments

The tool is highly configurable via the following command-line options:

### Required Parameters

| Option                 | Alias | Description                                                                                    |
| :--------------------- | :---- | :--------------------------------------------------------------------------------------------- |
| **`--project-id`** | `-p`  | **(Required)** Google Cloud Project ID.                                                        |
| **`--instance-id`** | `-i`  | **(Required)** Spanner Instance ID.                                                            |
| **`--database-id`** | `-d`  | **(Required)** Spanner Database ID.                                                            |

### Workload Configuration

| Option                 | Description                                                                                    | Default           |
| :--------------------- | :--------------------------------------------------------------------------------------------- | :---------------- |
| `--table-name`, `-t`   | Name of the table to operate on.                                                               | `LoadTestTable`   |
| `--workload`           | Type of workload: `READ`, `WRITE`, `READ_WRITE`.                                               | `WRITE`           |
| `--num-load-rows`      | Number of rows to pre-load. Recommended for `READ` workloads.                                  | `0`               |
| `--skip-load-phase`    | If present, skips the initial data loading phase.                                              | `false`           |
| `--max-async-inflight` | Max number of concurrent async Spanner operations in flight.                                   | `1000`            |

### QPS Control

| Option                 | Description                                                                                    | Default           |
| :--------------------- | :--------------------------------------------------------------------------------------------- | :---------------- |
| `--start-qps`          | Initial total operations per second (QPS).                                                     | `10`              |
| `--end-qps`            | Maximum total QPS. If `--burst` is used, this is the target burst QPS. `0` means no limit.      | `0`               |
| `--interval-seconds`   | Interval in seconds to increase QPS in step mode.                                              | `60`              |
| `--step-qps`           | Percentage to increase QPS by at each interval.                                                | `5`               |
| `--burst`              | If present, enables burst mode. After 15 mins, QPS jumps to `--end-qps`.                       | `false`           |

### Threading and Duration

| Option                   | Description                                                                                    | Default           |
| :----------------------- | :--------------------------------------------------------------------------------------------- | :---------------- |
| `--num-threads`          | Number of worker threads initiating async operations.                                          | `10`              |
| `--run-duration-minutes` | Total run duration in minutes. `0` means run indefinitely.                                     | `0`               |

### Dynamic Channel Pool (DCP) Configuration

| Option                 | Description                                                                                    | Default           |
| :--------------------- | :--------------------------------------------------------------------------------------------- | :---------------- |
| `--disable-dcp`        | Disable dynamic channel pooling (uses static channel count).                                   | `false` (DCP on)  |
| `--num-channels`       | Number of gRPC channels when DCP is disabled.                                                  | `4`               |

### OpenTelemetry Configuration

| Option                          | Description                                                                             | Default           |
| :------------------------------ | :-------------------------------------------------------------------------------------- | :---------------- |
| `--enable-otel-metrics`         | Enable OpenTelemetry metrics export to Cloud Monitoring.                                | `true`            |
| `--otel-export-interval-seconds`| OpenTelemetry metrics export interval in seconds.                                       | `60`              |

-----

## Examples

### 1. Benchmarking DCP vs Non-DCP (Step QPS from 50 to 600)

**With DCP Enabled (Default):**
```bash
java -jar target/spanner-load-generator-1.0-SNAPSHOT.jar \
    --project-id my-gcp-project \
    --instance-id my-spanner-instance \
    --database-id my-spanner-database \
    --table-name LoadTestTable \
    --workload WRITE \
    --num-load-rows 10000 \
    --start-qps 50 \
    --step-qps 10 \
    --interval-seconds 60 \
    --end-qps 600 \
    --num-threads 20 \
    --run-duration-minutes 60
```

**With DCP Disabled (Static Channels):**
```bash
java -jar target/spanner-load-generator-1.0-SNAPSHOT.jar \
    --project-id my-gcp-project \
    --instance-id my-spanner-instance \
    --database-id my-spanner-database \
    --table-name LoadTestTable \
    --workload WRITE \
    --num-load-rows 10000 \
    --start-qps 50 \
    --step-qps 10 \
    --interval-seconds 60 \
    --end-qps 600 \
    --num-threads 20 \
    --run-duration-minutes 60 \
    --disable-dcp \
    --num-channels 4
```

**Expected Results:**
- **Without DCP**: As QPS increases beyond ~400, you may observe head-of-line blocking, increased latencies.
- **With DCP**: The channel pool scales dynamically, preventing head-of-line blocking and maintaining good latencies.

### 2. Running Step QPS with a Load Phase

This scenario pre-loads the database with 100,000 rows, then starts a `READ_WRITE` workload at 100 QPS. The QPS will increase by 10% every 90 seconds until it reaches the ceiling of 2000 QPS.

```bash
java -jar target/spanner-load-generator-1.0-SNAPSHOT.jar \
    --project-id my-gcp-project \
    --instance-id my-spanner-instance \
    --database-id my-spanner-database \
    --table-name Users \
    --workload READ_WRITE \
    --num-load-rows 100000 \
    --start-qps 100 \
    --step-qps 10 \
    --interval-seconds 90 \
    --end-qps 2000 \
    --num-threads 16
```

  * The load phase is **active** because `--skip-load-phase` is not present.
  * `--burst` is **not used**, so the tool runs in step mode from the beginning.

### 3. Running Step QPS without a Load Phase

This example runs a `WRITE`-only workload, assuming the table is already populated. It starts at 50 QPS and increases by 5% every 60 seconds, with no upper QPS limit.

```bash
java -jar target/spanner-load-generator-1.0-SNAPSHOT.jar \
    --project-id my-gcp-project \
    --instance-id my-spanner-instance \
    --database-id my-spanner-database \
    --table-name Products \
    --workload WRITE \
    --skip-load-phase \
    --start-qps 50 \
    --step-qps 5 \
    --interval-seconds 60 \
    --num-threads 8
```

  * The `--skip-load-phase` flag disables the initial data load.
  * Since `--end-qps` is at its default of `0`, the QPS will increase indefinitely.

### 4. Running a "Burst" QPS without a Load Phase

This scenario tests how the system handles a sudden traffic spike. It skips the data load, runs at a baseline of 200 QPS for 15 minutes, and then **bursts** to 5000 QPS.

```bash
java -jar target/spanner-load-generator-1.0-SNAPSHOT.jar \
    --project-id my-gcp-project \
    --instance-id my-spanner-instance \
    --database-id my-spanner-database \
    --table-name AuditLogs \
    --workload WRITE \
    --skip-load-phase \
    --start-qps 200 \
    --end-qps 5000 \
    --burst \
    --num-threads 32 \
    --run-duration-minutes 30
```

  * The **`--burst`** flag activates burst mode.
  * **`--end-qps 5000`** is crucial here; it specifies the target QPS for the burst.
  * The test will run at 200 QPS for 15 minutes, then jump to 5000 QPS.

-----

## Viewing Metrics in Cloud Monitoring

When `--enable-otel-metrics` is set (default: true), the load generator exports metrics to Google Cloud Monitoring. You can view these metrics in the Cloud Console:

1. Go to **Cloud Monitoring** > **Metrics Explorer**
2. Search for metrics with prefix `workload.googleapis.com/`
3. Key grpc-gcp metrics to look for:
   - `workload.googleapis.com/grpc_gcp_max_channels`: Maximum number of channels in the pool
   - `workload.googleapis.com/grpc_gcp_max_active_streams_per_channel`: Maximum concurrent RPCs per channel
   - `workload.googleapis.com/grpc_gcp_min_active_streams_per_channel`: Minimum concurrent RPCs per channel
   - `workload.googleapis.com/grpc_gcp_num_channel_connect`: Channel creation events
   - `workload.googleapis.com/grpc_gcp_num_channel_disconnect`: Channel teardown events

These metrics help identify:
- **Head-of-line blocking**: When `max_active_streams_per_channel` is consistently high (approaching 100)
- **DCP effectiveness**: When `max_channels` increases in response to load
- **Load distribution**: Comparing min vs max active streams across channels

-----

## Dependencies

- `google-cloud-spanner` 6.105.0 (with grpc-gcp-java enabled by default)
- `io.opentelemetry:opentelemetry-sdk` 1.40.0
- `com.google.cloud.opentelemetry:exporter-metrics` 0.33.0
- `info.picocli:picocli` 4.7.6
