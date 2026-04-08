# check_rabbitmq

A Nagios/Icinga2-compatible monitoring plugin for RabbitMQ 4.x clusters.

Single file. Zero external dependencies. Uses only Python 3 standard library. Designed for disconnected RHEL 9 environments where pip, CPAN, and package repos are unavailable.

---

## Table of Contents

1. [Features at a Glance](#features-at-a-glance)
2. [Requirements](#requirements)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [Check Modes in Detail](#check-modes-in-detail)
   - [publish_consume](#publish_consume)
   - [queue_exists](#queue_exists)
   - [queue_count](#queue_count)
   - [exchange_count](#exchange_count)
   - [cluster_health](#cluster_health)
6. [Publish Volume Control](#publish-volume-control)
   - [Fixed Message Count](#fixed-message-count)
   - [Duration-Based Publishing](#duration-based-publishing)
   - [Duration Format Reference](#duration-format-reference)
7. [Queue Lifecycle Control](#queue-lifecycle-control)
   - [Default Behavior (Temporary Queue)](#default-behavior-temporary-queue)
   - [Named Persistent Queue](#named-persistent-queue)
   - [No-Delete Flag](#no-delete-flag)
   - [Decision Matrix](#decision-matrix)
8. [TLS and AMQPS](#tls-and-amqps)
9. [Debug Mode](#debug-mode)
10. [Credential Safety](#credential-safety)
11. [Error Messages](#error-messages)
12. [Complete Option Reference](#complete-option-reference)
13. [Output Format and Perfdata](#output-format-and-perfdata)
14. [Exit Codes](#exit-codes)
15. [Icinga2 Integration](#icinga2-integration)
    - [CheckCommand Installation](#checkcommand-installation)
    - [Host Template](#host-template)
    - [Service Examples](#service-examples)
    - [Per-Queue Monitoring with apply-for](#per-queue-monitoring-with-apply-for)
    - [TLS-Enabled Host](#tls-enabled-host)
16. [RabbitMQ Permissions Setup](#rabbitmq-permissions-setup)
17. [Firewall Requirements](#firewall-requirements)
18. [Design Decisions](#design-decisions)
19. [Troubleshooting](#troubleshooting)

---

## Features at a Glance

| Mode | Protocol | What It Does |
|------|----------|-------------|
| `publish_consume` | AMQP 0-9-1 | End-to-end functional test: declare queue, publish N messages (or publish for N seconds), consume all back, verify content integrity, report throughput. |
| `queue_exists` | Management API | Check whether a named queue exists; report its type, state, message count, and consumer count. |
| `queue_count` | Management API | Count all queues (optionally per-vhost) and apply warning/critical thresholds. |
| `exchange_count` | Management API | Count all exchanges (excluding `amq.*` defaults unless told otherwise) and apply thresholds. |
| `cluster_health` | Management API | Comprehensive cluster health: node status, memory/disk alarms, FD/socket usage, maintenance mode, Erlang/RabbitMQ versions, listener protocols. |

All modes produce Nagios-standard perfdata for graphing in Grafana, Graphite, PNP4Nagios, or any compatible tool.

---

## Requirements

- **Python 3.9 or later** — ships with RHEL 9 as `/usr/bin/python3`. Nothing else to install.
- **RabbitMQ Management Plugin** — required for all modes except `publish_consume`. Enable it with `rabbitmq-plugins enable rabbitmq_management`.
- **A monitoring user** — see [RabbitMQ Permissions Setup](#rabbitmq-permissions-setup) below.

The plugin does not use pip, pika, Perl, Java, or any compiled extension.

---

## Installation

```bash
# Copy to the standard Nagios plugins directory
sudo cp check_rabbitmq /usr/lib64/nagios/plugins/check_rabbitmq
sudo chmod 755 /usr/lib64/nagios/plugins/check_rabbitmq
sudo chown root:root /usr/lib64/nagios/plugins/check_rabbitmq

# Verify
/usr/lib64/nagios/plugins/check_rabbitmq --version
/usr/lib64/nagios/plugins/check_rabbitmq --help
```

That's it. One file, zero dependencies.

---

## Quick Start

```bash
# 1. Single-message roundtrip (simplest functional test)
./check_rabbitmq -m publish_consume -H rabbit1.example.com -u monitor -p secret

# 2. Publish 500 messages, consume all back, verify every one
./check_rabbitmq -m publish_consume -H rabbit1.example.com -u monitor -p secret \
    --message-count 500

# 3. Publish continuously for 30 seconds, then consume all
./check_rabbitmq -m publish_consume -H rabbit1.example.com -u monitor -p secret \
    --publish-duration 30s

# 4. Full cluster health overview
./check_rabbitmq -m cluster_health -H rabbit1.example.com -u monitor -p secret

# 5. Check a specific queue exists
./check_rabbitmq -m queue_exists -H rabbit1.example.com -u monitor -p secret \
    --queue-name orders.incoming --vhost /production

# 6. Count queues with thresholds
./check_rabbitmq -m queue_count -H rabbit1.example.com -u monitor -p secret \
    -w 100 -c 500

# 7. Everything above, but with debug tracing
./check_rabbitmq -m cluster_health -H rabbit1.example.com -u monitor -p secret --debug
```

---

## Check Modes in Detail

### publish_consume

Performs a full end-to-end AMQP functional test:

1. Connects to RabbitMQ via AMQP (or AMQPS with `--amqps`).
2. Declares a quorum queue (or reuses an existing one with `--queue-name`).
3. Publishes one or more messages (controlled by `--message-count` or `--publish-duration`).
4. Consumes all messages back and verifies content integrity against a unique run ID.
5. Acknowledges every message.
6. Optionally deletes the queue (see [Queue Lifecycle Control](#queue-lifecycle-control)).
7. Reports timing breakdown and throughput rates.

Each message is tagged with a unique run identifier plus a sequence number. During consumption, the plugin only counts messages that match its own run ID — this means it safely handles existing queues that may already contain application messages without false failures.

Quorum queues are used deliberately because they are the recommended queue type for RabbitMQ 4.x. Testing with quorum queues validates that the Raft consensus mechanism is working across the cluster, which is a stronger health signal than a classic queue test.

**Perfdata emitted:** `connect_time`, `declare_time`, `publish_time`, `consume_time`, `total_time`, `published`, `consumed`, `publish_rate` (msg/s), `consume_rate` (msg/s).

**Thresholds** (`-w`/`-c`) apply to `total_time` in milliseconds.

### queue_exists

Checks whether a specific queue exists in a given vhost and reports its state. Uses the Management API.

Returns CRITICAL if the queue does not exist, WARNING if it exists but its state is not `running`, and OK otherwise.

**Perfdata emitted:** `messages`, `consumers`, `messages_ready`, `messages_unacked`.

**Requires:** `--queue-name`.

### queue_count

Counts the total number of queues, optionally filtered to a single vhost with `--filter-vhost`.

**Thresholds** (`-w`/`-c`) apply to the queue count.

**Perfdata emitted:** `queue_count`.

### exchange_count

Counts user-defined exchanges, excluding the built-in `amq.*` defaults and the nameless default exchange. Pass `--include-defaults` to count those too.

**Thresholds** (`-w`/`-c`) apply to the exchange count.

**Perfdata emitted:** `exchange_count`.

### cluster_health

The most comprehensive mode. Queries the Management API for the full cluster state and evaluates:

- **Node availability** — how many nodes are running vs. total.
- **Memory alarms** — per-node, triggers CRITICAL.
- **Disk free alarms** — per-node, triggers CRITICAL.
- **File descriptor usage** — per-node, triggers WARNING above 90%.
- **Socket descriptor usage** — per-node, triggers WARNING above 90%.
- **Memory usage percentage** — per-node, triggers WARNING above 90%.
- **Disk free headroom** — per-node, triggers WARNING when free space is less than 1.5x the configured limit.
- **Maintenance/draining mode** — per-node, triggers WARNING.
- **Health check endpoints** — `/health/checks/alarms` and `/health/checks/local-alarms`.
- **Cluster metadata** — cluster name, RabbitMQ version, Erlang version, listener protocols.

**Perfdata emitted:** `nodes_running`, `nodes_total`, `queues`, `exchanges`, `connections`, `channels`, `consumers`, `messages`, `messages_ready`, `messages_unacked`, plus per-node `<node>_mem_pct`, `<node>_fd_used`, `<node>_sockets_used`.

---

## Publish Volume Control

The `publish_consume` mode supports three volume strategies. The default is a single message, which is suitable for lightweight health checks. The other two are designed for throughput testing and stress validation.

### Fixed Message Count

```bash
./check_rabbitmq -m publish_consume -H rabbit1 -u monitor -p secret \
    --message-count 500
```

Publishes exactly 500 messages, then consumes all 500 back and verifies each one. Progress is logged to debug output every 100 messages.

### Duration-Based Publishing

```bash
./check_rabbitmq -m publish_consume -H rabbit1 -u monitor -p secret \
    --publish-duration 30s
```

Publishes messages as fast as possible for 30 seconds, then consumes everything back. The total number of messages published depends on your broker's throughput. This is ideal for periodic stress tests.

### Duration Format Reference

The `--publish-duration` flag accepts human-friendly time strings:

| Input | Meaning |
|-------|---------|
| `30` | 30 seconds |
| `30s` | 30 seconds |
| `5m` | 5 minutes |
| `1m30s` | 1 minute 30 seconds |
| `2m` | 2 minutes |
| `0.5s` | 500 milliseconds |

The two volume options (`--message-count` and `--publish-duration`) are **mutually exclusive** — the parser rejects attempts to use both simultaneously.

---

## Queue Lifecycle Control

The plugin supports three strategies for queue management in `publish_consume` mode. The goal is to give you full control over whether queues are created, reused, and/or deleted.

### Default Behavior (Temporary Queue)

```bash
./check_rabbitmq -m publish_consume -H rabbit1 -u monitor -p secret
```

Creates a temporary quorum queue named `icinga2_check_<random>`, runs the test, and deletes the queue afterward. No trace is left.

### Named Persistent Queue

```bash
./check_rabbitmq -m publish_consume -H rabbit1 -u monitor -p secret \
    --queue-name my.health.check.queue
```

When you provide `--queue-name`, the plugin first performs an AMQP passive declare to check if the queue already exists. If it does, the queue is reused and never deleted (it's not ours to delete). If it doesn't exist, the plugin creates it as a quorum queue; this newly created queue will be deleted after the test unless `--no-delete` is also set.

This approach is safe for existing application queues — the consume phase only picks up messages tagged with the current run's unique ID and skips foreign messages.

### No-Delete Flag

```bash
./check_rabbitmq -m publish_consume -H rabbit1 -u monitor -p secret --no-delete
```

When `--no-delete` is set, the queue is never deleted regardless of how it was created. This is useful for:

- Keeping a dedicated monitoring queue across check runs.
- Post-mortem inspection of queue state after a check failure.
- Combining with `--queue-name` and `--publish-duration` for persistent stress-test queues.

### Decision Matrix

| Queue existed before? | `--no-delete` set? | What happens to the queue |
|:---------------------:|:------------------:|:--------------------------|
| Yes | No | Kept (not ours to delete) |
| Yes | Yes | Kept |
| No (created by check) | No | Deleted after test |
| No (created by check) | Yes | Kept |

---

## TLS and AMQPS

The plugin supports TLS for both AMQP connections and Management API HTTP connections.

```bash
# AMQPS with CA certificate verification
./check_rabbitmq -m publish_consume -H rabbit1.example.com \
    -u monitor -p secret --amqps \
    --ssl-ca-cert /etc/pki/tls/certs/rabbitmq-ca.pem

# AMQPS with mutual TLS (client certificate authentication)
./check_rabbitmq -m publish_consume -H rabbit1.example.com \
    -u monitor -p secret --amqps \
    --ssl-ca-cert /etc/pki/tls/certs/ca.pem \
    --ssl-cert /etc/pki/tls/certs/icinga-client.pem \
    --ssl-key /etc/pki/tls/private/icinga-client.key

# Management API over HTTPS
./check_rabbitmq -m cluster_health -H rabbit1.example.com \
    -u monitor -p secret --mgmt-ssl --mgmt-port 15671 \
    --ssl-ca-cert /etc/pki/tls/certs/rabbitmq-ca.pem

# Skip TLS verification (testing only!)
./check_rabbitmq -m publish_consume -H rabbit1.example.com \
    -u monitor -p secret --amqps --ssl-no-verify
```

When `--amqps` is set and the AMQP port has not been explicitly changed from the default 5672, the plugin automatically switches to port 5671.

---

## Debug Mode

Pass `--debug` (or `-d`) to enable detailed tracing on stderr. Debug output never appears on stdout, so it never interferes with Nagios plugin output parsing.

```bash
./check_rabbitmq -m publish_consume -H rabbit1 -u monitor -p secret --debug
```

Every significant step is traced: TCP connection and TLS handshake details, every AMQP frame sent and received with human-readable method names (e.g., `Connection.Tune`, `Queue.Declare-OK`), HTTP requests and responses to the Management API, queue declaration, message publish/consume progress (every 100 messages in bulk mode), threshold evaluation, and final exit code.

Example debug output (abbreviated):

```
[DEBUG 14:23:01] check_rabbitmq v1.2.0 starting
[DEBUG 14:23:01] Mode: publish_consume
[DEBUG 14:23:01] Host: rabbit1.example.com, AMQP port: 5672, Mgmt port: 15672
[DEBUG 14:23:01] Vhost: /, User: monitor, Password: ***
[DEBUG 14:23:01] Message count: 100
[DEBUG 14:23:01] AMQP connecting to amqp://rabbit1.example.com:5672// as user 'monitor'
[DEBUG 14:23:01] Socket created, timeout=10s
[DEBUG 14:23:01] TCP connected to rabbit1.example.com:5672
[DEBUG 14:23:01] Sent AMQP 0-9-1 protocol header
[DEBUG 14:23:01]   << Frame: METHOD ch=0 Connection.Start (422 bytes)
[DEBUG 14:23:01] Received Connection.Start from server
...
[DEBUG 14:23:01] Publishing 100 message(s)
[DEBUG 14:23:02]   Published 100/100 messages
[DEBUG 14:23:02] Publish phase complete: 100 messages
[DEBUG 14:23:02] Waiting 0.3s for quorum replication
[DEBUG 14:23:02] Consuming: expecting 100 messages
[DEBUG 14:23:03]   Consumed 100/100 messages
[DEBUG 14:23:03] All 100 messages verified successfully
[DEBUG 14:23:03] Deleting queue 'icinga2_check_a1b2c3d4e5f6' (created by this check)
[DEBUG 14:23:03] Timings: connect=45.2ms, declare=12.3ms, publish=890.1ms, consume=623.4ms, total=1601.2ms
[DEBUG 14:23:03] Rates: publish=112.3 msg/s, consume=160.5 msg/s
[DEBUG 14:23:03] Exit code: 0 (OK)
```

---

## Credential Safety

The plugin guarantees that credentials never appear in any output — not in stdout (Nagios output), not in stderr (debug output), and not in error messages.

This is enforced at multiple levels:

- The password is registered globally at startup and scrubbed from every string before it reaches any output function.
- Base64-encoded forms of credentials (as used in HTTP Basic auth headers) are also detected and replaced.
- The startup debug log explicitly prints `Password: ***`.
- All exception messages, AMQP server error texts, and HTTP response bodies are passed through the sanitizer before display.
- The `safe()` function is called at every point where error text might contain credential data — AMQP connection errors, Management API errors, and unhandled exception traces.

---

## Error Messages

When RabbitMQ returns an HTTP error, the plugin translates it into a human-readable description instead of just showing a numeric status code. Examples:

| HTTP Status | Plugin Output |
|-------------|--------------|
| 401 | `Management API /overview returned 401 (Unauthorized - authentication failed (check username and permissions))` |
| 403 | `Management API /queues/%2F returned 403 (Forbidden - user does not have sufficient permissions)` |
| 404 | `Queue 'my.queue' does not exist in vhost '/'` |
| 503 | `Management API /nodes returned 503 (Service Unavailable - RabbitMQ is starting up, shutting down, or overloaded)` |

If the response body contains a JSON `reason`, `error`, or `message` field, that detail is appended (with credentials scrubbed).

AMQP-level errors are similarly described with the server's reply code and reply text, e.g.:

```
Channel closed by server: 404 - NOT_FOUND - no queue 'nonexistent' in vhost '/'
```

---

## Complete Option Reference

```
Connection:
  -H, --host HOST             RabbitMQ hostname (default: localhost)
  -u, --username USERNAME     RabbitMQ username (default: guest)
  -p, --password PASSWORD     RabbitMQ password (default: guest)
  --vhost VHOST               AMQP vhost (default: /)
  -t, --timeout TIMEOUT       Connection timeout in seconds (default: 10)

AMQP (publish_consume mode):
  --amqp-port PORT            AMQP port (default: 5672; auto-switches to 5671 with --amqps)
  --amqps                     Use AMQPS (TLS) for the AMQP connection

Management API (all other modes):
  --mgmt-port PORT            Management API port (default: 15672)
  --mgmt-ssl                  Use HTTPS for the Management API

TLS/SSL:
  --ssl-ca-cert FILE          CA certificate for TLS verification
  --ssl-cert FILE             Client certificate (AMQPS mutual TLS)
  --ssl-key FILE              Client private key (AMQPS mutual TLS)
  --ssl-no-verify             Skip TLS certificate verification (insecure!)

Thresholds:
  -w, --warning VALUE         Warning threshold (ms for publish_consume, count for others)
  -c, --critical VALUE        Critical threshold (ms for publish_consume, count for others)

Mode-specific:
  --queue-name NAME           Queue name (required for queue_exists; optional for publish_consume)
  --no-delete                 Do not delete queue after publish_consume test
  --filter-vhost VHOST        Override vhost filter for queue_count/exchange_count
  --include-defaults          Include amq.* default exchanges in exchange_count

Publish volume (publish_consume, mutually exclusive):
  --message-count N           Publish exactly N messages (default: 1)
  --publish-duration D        Publish continuously for duration D (e.g. 30s, 5m, 1m30s)

Debug:
  --debug, -d                 Enable debug output on stderr
```

---

## Output Format and Perfdata

Output follows the Nagios plugin specification:

```
STATUS - Human-readable message | key=value;warn;crit;min;max key2=value2;...
```

### Example: Single-message publish_consume

```
OK - Publish/consume roundtrip in 42.3ms via amqp (queue: icinga2_check_a1b2c3, created, deleted, vhost: /) | connect_time=12.1ms;;;; declare_time=8.4ms;;;; publish_time=3.2ms;;;; consume_time=15.8ms;;;; total_time=42.3ms;;;; published=1;;;; consumed=1;;;; publish_rate=312.5;;;; consume_rate=63.3;;;;
```

### Example: Bulk publish_consume (500 messages)

```
OK - Publish/consume 500 msgs in 4823.1ms (pub: 245 msg/s, con: 312 msg/s) via amqp (queue: icinga2_check_f8e2a1, created, deleted, vhost: /) | connect_time=38.2ms;;;; declare_time=11.5ms;;;; publish_time=2040.8ms;;;; consume_time=1602.3ms;;;; total_time=4823.1ms;;;; published=500;;;; consumed=500;;;; publish_rate=244.9;;;; consume_rate=312.1;;;;
```

### Example: cluster_health

```
OK - Cluster healthy [cluster=rabbit@prod, rabbitmq=4.1.0, erlang=27.0, nodes=3/3 running, protocols=amqp,amqp/ssl,clustering,http,https] | nodes_running=3;;;;3 nodes_total=3;;;; queues=47;;;; exchanges=12;;;; connections=89;;;; channels=156;;;; consumers=42;;;; messages=1203;;;; messages_ready=18;;;; messages_unacked=4;;;; node1_mem_pct=62.3%;90;95;0;100 ...
```

### Example: queue_exists

```
OK - Queue 'orders.incoming' exists and is running (type=quorum, messages=42, consumers=3) | messages=42;;;; consumers=3;;;; messages_ready=40;;;; messages_unacked=2;;;;
```

### Example: queue_count with threshold

```
WARNING - 142 queues found in / (threshold: 100.0) | queue_count=142;100.0;500.0;0;
```

---

## Exit Codes

| Code | State | Meaning |
|------|-------|---------|
| 0 | OK | Check passed |
| 1 | WARNING | Threshold exceeded or non-critical issue (e.g. node in maintenance) |
| 2 | CRITICAL | Check failed, threshold exceeded, connection error, or alarm active |
| 3 | UNKNOWN | Invalid arguments, invalid duration format, or unhandled error |

---

## Icinga2 Integration

The plugin ships with a complete `icinga2_checkcommand.conf` containing the CheckCommand definition, a host template, seven service apply rules, and commented examples for TLS-enabled hosts.

### CheckCommand Installation

```bash
# Copy the CheckCommand definition
sudo cp icinga2_checkcommand.conf \
    /etc/icinga2/zones.d/global-templates/commands/check_rabbitmq.conf

# Validate and reload
sudo icinga2 daemon -C
sudo systemctl reload icinga2
```

### Host Template

```
template Host "rabbitmq-node" {
    vars.rabbitmq_username = "monitor"
    // vars.rabbitmq_password = "set-via-api-or-vault"
    vars.rabbitmq_vhost = "/"
}
```

### Service Examples

The shipped configuration provides seven service definitions, each with recommended intervals:

| Service | Mode | Interval | Opt-in Required |
|---------|------|----------|-----------------|
| `rabbitmq-publish-consume` | Single-message roundtrip | 5m | No (all `rabbitmq` group) |
| `rabbitmq-publish-consume-persistent` | Roundtrip on named queue | 5m | `rabbitmq_persistent_check = true` |
| `rabbitmq-bulk-publish-consume` | 100-message batch test | 15m | `rabbitmq_bulk_check = true` |
| `rabbitmq-stress-test` | 30s continuous publish | 30m | `rabbitmq_stress_check = true` |
| `rabbitmq-cluster-health` | Full cluster health | 2m | No (all `rabbitmq` group) |
| `rabbitmq-queue-count` | Queue count with thresholds | 5m | No (all `rabbitmq` group) |
| `rabbitmq-exchange-count` | Exchange count with thresholds | 10m | No (all `rabbitmq` group) |

The bulk and stress services require explicit opt-in via host vars to prevent accidentally running heavy checks on every node.

### Per-Queue Monitoring with apply-for

The configuration includes an apply-for pattern that automatically creates a `queue_exists` service for every queue listed in a host's `rabbitmq_queues` dictionary:

```
object Host "rabbit1.example.com" {
    import "rabbitmq-node"
    address = "10.0.1.10"
    groups = [ "rabbitmq" ]

    vars.rabbitmq_password = "secret"

    vars.rabbitmq_queues = {
        "orders.incoming" = { rabbitmq_vhost = "/production" }
        "events.notifications" = { rabbitmq_vhost = "/production" }
        "dlx.failed" = {}
    }
}
```

This creates three services automatically: `rabbitmq-queue-orders.incoming`, `rabbitmq-queue-events.notifications`, and `rabbitmq-queue-dlx.failed`.

### TLS-Enabled Host

```
object Host "rabbit-tls.example.com" {
    import "rabbitmq-node"
    address = "10.0.2.10"
    groups = [ "rabbitmq" ]

    vars.rabbitmq_password = "secret"
    vars.rabbitmq_amqps = true
    vars.rabbitmq_amqp_port = 5671
    vars.rabbitmq_mgmt_ssl = true
    vars.rabbitmq_mgmt_port = 15671
    vars.rabbitmq_ssl_ca_cert = "/etc/pki/tls/certs/rabbitmq-ca.pem"
}
```

---

## RabbitMQ Permissions Setup

Create a dedicated monitoring user with minimal permissions:

```bash
# Create the user
rabbitmqctl add_user monitor 'YourSecurePassword'

# Grant the 'monitoring' tag for Management API read-only access
rabbitmqctl set_user_tags monitor monitoring

# Grant AMQP permissions for publish_consume mode
# (scoped to icinga2_check_* queue names only)
rabbitmqctl set_permissions -p / monitor \
    "^icinga2_check_.*$" \
    "^(icinga2_check_.*|amq\\.default)$" \
    "^icinga2_check_.*$"
```

If you use `--queue-name` with a custom name, adjust the configure/write/read regex to match. For example, to also allow a persistent health check queue and a stress-test queue:

```bash
rabbitmqctl set_permissions -p / monitor \
    "^(icinga2_check_.*|icinga2\\.healthcheck|icinga2\\.stress)$" \
    "^(icinga2_check_.*|icinga2\\.healthcheck|icinga2\\.stress|amq\\.default)$" \
    "^(icinga2_check_.*|icinga2\\.healthcheck|icinga2\\.stress)$"
```

If you monitor multiple vhosts, repeat `set_permissions` for each one:

```bash
rabbitmqctl set_permissions -p /production monitor \
    "^icinga2_check_.*$" \
    "^(icinga2_check_.*|amq\\.default)$" \
    "^icinga2_check_.*$"
```

The `monitoring` tag grants read-only access to the Management API. It cannot modify queues, exchanges, bindings, or any other server state through HTTP. The AMQP permissions are scoped so the monitoring user cannot touch application queues.

---

## Firewall Requirements

| Port | Protocol | Needed For |
|------|----------|-----------|
| 5672 | AMQP | `publish_consume` mode |
| 5671 | AMQPS | `publish_consume --amqps` mode |
| 15672 | HTTP | `queue_exists`, `queue_count`, `exchange_count`, `cluster_health` |
| 15671 | HTTPS | Same as above with `--mgmt-ssl` |

---

## Design Decisions

### Why Python stdlib instead of Go, Rust, or C?

Go would produce a single static binary, which is appealing. However, it requires a build machine with Go installed and internet access to fetch module dependencies. On a disconnected RHEL 9 system, Python 3.9 is already present in the base installation. This plugin is a single file you `scp` to the target and it works. No build step, no cross-compilation, no container, no package manager.

### Why raw AMQP 0-9-1 instead of pika?

The `pika` library is not available on disconnected RHEL 9 without pip. This plugin implements the minimal subset of the AMQP 0-9-1 protocol needed for monitoring: connection handshake with PLAIN authentication, channel management, queue declare (active and passive), queue delete, basic publish, basic get, and basic ack. This is roughly 350 lines of protocol handling using Python's `socket` and `struct` modules.

### Why Management API for the non-AMQP modes?

The Management API is the proper way to query cluster-wide state. Getting node health, alarm status, queue counts, and exchange counts over AMQP would require parsing internal diagnostic exchanges or shelling out to `rabbitmqctl`, both of which are fragile. The Management API is stable, versioned, and designed for this use case.

### Why quorum queues?

Classic mirrored queues are deprecated in RabbitMQ 4.x. Quorum queues are the recommended replacement and use a Raft consensus protocol for replication. By creating quorum queues in the health check, the plugin validates that the consensus mechanism is working across the cluster — a meaningfully stronger signal than testing with a classic or transient queue.

### Why messages are tagged with run IDs

When reusing an existing queue (`--queue-name`), the queue may contain messages from application producers or from previous check runs. The plugin tags every published message with a unique run ID and sequence number. During consumption, it only counts messages matching the current run ID. Foreign messages are acknowledged and skipped without triggering a failure. This makes `--queue-name` safe to use on any queue.

---

## Troubleshooting

**"Connection refused" on port 15672**

The Management Plugin is not enabled:
```bash
rabbitmq-plugins enable rabbitmq_management
```

**"401 Unauthorized" from Management API**

The user does not have the `monitoring` or `administrator` tag:
```bash
rabbitmqctl set_user_tags monitor monitoring
```

**"ACCESS_REFUSED" on AMQP connect**

The user does not have permissions on the target vhost:
```bash
rabbitmqctl set_permissions -p /your-vhost monitor \
    "^icinga2_check_.*$" \
    "^(icinga2_check_.*|amq\\.default)$" \
    "^icinga2_check_.*$"
```

**"PRECONDITION_FAILED" when reusing a queue**

The existing queue was declared with different properties (e.g., classic instead of quorum). The passive declare will succeed, but if the queue didn't exist and the plugin tries to create it as quorum while another process creates it as classic, you'll get this error. Ensure the queue type matches or let the plugin manage its own queues.

**Timeout on publish_consume**

- Check firewall rules for port 5672/5671 from the monitoring server to the RabbitMQ node.
- Check that AMQP listeners are bound to the correct network interface (`rabbitmq.conf`: `listeners.tcp.default`).
- Increase timeout with `-t 30`.
- For large `--message-count` or `--publish-duration` values, ensure `--timeout` is generous enough for the connection to survive the full publish/consume cycle. Note that `--timeout` applies to the initial TCP connection and individual socket operations, not to the total runtime of the check.

**SSL handshake failures**

- Verify the CA certificate: `openssl verify -CAfile /path/to/ca.pem /path/to/server.pem`
- Check that the hostname matches the certificate's SAN or CN.
- Use `--ssl-no-verify` temporarily to isolate certificate issues from connectivity issues.
- Check that the correct port is being used (5671 for AMQPS, 15671 for Management HTTPS).

**Missing messages in bulk publish_consume**

If the plugin reports fewer consumed messages than published, this typically indicates:

- A very slow quorum queue (the replication wait is capped at 2 seconds). Try running with `--debug` to see timing details.
- A consumer on the queue that is racing with the plugin's consumer. Use a dedicated `--queue-name` for monitoring to avoid this.
- The queue has a TTL or max-length policy that is discarding messages before the plugin can consume them.

**Debug output is overwhelming**

Debug output includes every AMQP frame. For a 10,000-message test, that's a lot of lines. You can filter it:

```bash
# Only see high-level steps, not individual frames
./check_rabbitmq -m publish_consume -H rabbit1 -u monitor -p secret \
    --message-count 10000 --debug 2>&1 1>/dev/null | grep -v '  <<\|  >>'
```
