---
title: Java Client Documentation
description:   "Dive into QuestDB using the Java ingestion client for high-performance,
insert-only operations. Unlock peak time series data ingestion and analysis
efficiency."
---

import Tabs from "@theme/Tabs"
import TabItem from "@theme/TabItem"
import CodeBlock from "@theme/CodeBlock"
import InterpolateReleaseData from "../../src/components/InterpolateReleaseData"
import { RemoteRepoExample } from "@theme/RemoteRepoExample"

The QuestDB Java client is baked right into the QuestDB binary.

It requires no additional configuration steps.

The client provides the following benefits:

- **Automatic table creation**: No need to define your schema upfront.
- **Concurrent schema changes**: Seamlessly handle multiple data streams with
  on-the-fly schema modifications
- **Optimized batching**: Use strong defaults or curate the size of your batches
- **Health checks and feedback**: Ensure your system's integrity with built-in
  health monitoring
- **Automatic write retries**: Reuse connections and retry after interruptions

## Quick start

Add a QuestDB as a dependency in your project's build configuration file.

<Tabs
  defaultValue="maven"
  values={[
    { label: "Maven", value: "maven" },
    { label: "Gradle", value: "gradle" },
  ]}
>
  <TabItem value="maven">
    <InterpolateReleaseData
      renderText={(release) => (
        <CodeBlock className="language-xml">
          {`<dependency>
  <groupId>org.questdb</groupId>
  <artifactId>questdb</artifactId>
  <version>${release.name}</version>
</dependency>`}
        </CodeBlock>
      )}
    />
  </TabItem>
  <TabItem value="gradle">
    <InterpolateReleaseData
      renderText={(release) => (
        <CodeBlock className="language-text">
          {`compile group: 'org.questdb', name: 'questdb', version: '${release.name}'`}
        </CodeBlock>
      )}
    />
  </TabItem>
</Tabs>

The code below creates an instance of a client configured to use HTTP transport
to connect to a QuestDB server running on localhost on port 9000. It then sends
two rows, each containing one symbol and two floating-point values. The client
requests the server to assign a timestamp to each row based on the server's
wall-clock time.

<RemoteRepoExample name="ilp-http" lang="java" header={false} />

The configuration for the client is specified using a configuration string. This
string follows the format:

```
<protocol>::<key>=<value>;<key>=<value>;...;
```

The valid transport protocols are:

- `http`: ILP/HTTP
- `https`: ILP/HTTP with TLS encryption
- `tcp`: ILP/TCP
- `tcps`: ILP/TCP with TLS encryption

A [transport protocol](#transport-selection) and the key `addr=host:port` are
required. The key `addr` defines the hostname and port of the QuestDB server. If
the port is not specified, it defaults to 9000 for HTTP(s) transports and 9009
for TCP(s) transports. For a complete list of options, refer to the
[Configuration Options](#configuration-options) section.

## Example with TLS and Authentication enabled

This sample configures a client to use HTTP transport with TLS enabled for a
connection to a QuestDB server. It also instructs the client to authenticate
using HTTP Basic Authentication.

<RemoteRepoExample name="ilp-http-auth" lang="java" header={false} />

## Client instantiation

There are three ways to create a client instance:

1. **From a configuration string.** This is the most common way to create a
   client instance. It describes the entire client configuration in a single
   string. See [Configuration options](#configuration-options) for all available
   options. It allows sharing the same configuration across clients in different
   languages.
   ```java
   try (Sender sender = Sender.fromConfig("http::addr=localhost:9000;auto_flush_rows=5000;retry_timeout=10000;")) {
       // ...
   }
   ```
2. **From an environment variable.** The `QDB_CLIENT_CONF` environment variable
   is used to set the configuration string. Moving configuration parameters to
   an environment variable allows you to avoid hard-coding sensitive information
   such as tokens and password in your code.
   ```bash
   export QDB_CLIENT_CONF="http::addr=localhost:9000;auto_flush_rows=5000;retry_timeout=10000;"
   ```
   ```java
   try (Sender sender = Sender.fromEnv()) {
       // ...
   }
   ```
3. **Using the Java builder API.** This provides type-safe configuration.
   ```java
   try (Sender sender = Sender.builder(Sender.Transport.HTTP)
           .address("localhost:9000")
           .autoFlushRows(5000)
           .retryTimeoutMillis(10000)
           .build()) {
       // ...
   }
   ```

## General usage pattern

1. Create a client instance via `Sender.fromConfig()`.
2. Use `table(CharSequence)` to select a table for inserting a new row.
3. Use `symbol(CharSequence, CharSequence)` to add all symbols. You must add
   symbols before adding other column type.
4. Use the following options to add all the remaining columns:

   - `stringColumn(CharSequence, CharSequence)`
   - `longColumn(CharSequence, long)`
   - `doubleColumn(CharSequence, double)`
   - `boolColumn(CharSequence, boolean)`
   - `timestampColumn(CharSequence, Instant)`, or
     `timestampColumn(CharSequence, long, ChronoUnit)`

5. Use `at(Instant)` or `at(long timestamp, ChronoUnit unit) ` or `atNow()` to
   set a designated timestamp.
6. Optionally: You can use `flush()` to send locally buffered data into a
   server.
7. Go to the step no. 2 to start a new row.
8. Use `close()` to dispose the Sender after you no longer need it.

## Transport selection

Client supports the following transport options:

- HTTP (default port 9000)
- TCP (default port 9009)

The HTTP transport is recommended for most use cases. It provides feedback on
errors, automatically retries failed requests, and is easier to configure. The
TCP transport is kept for compatibility with older QuestDB versions. It has
limited error feedback, no automatic retries, and requires manual handling of
connection failures. However, while HTTP is recommended, TCP has a lower
overhead than HTTP and may be useful in high-throughput scenarios in
high-latency networks.

## Flushing

Client accumulates data into an internal buffer. Flushing the buffer sends the
data to the server over the network and clears the buffer.

Flushing can be done explicitly or automatically.

### Explicit flushing

An explicit flush can be done by calling the `flush()` method.

```java
 try (Sender sender = Sender.fromConfig("http::addr=localhost:9000;")) {
    sender.table("weather_sensor")
            .symbol("id", "toronto1")
            .doubleColumn("temperature", 23.5)
            .doubleColumn("humidity", 0.49)
            .atNow();
    sender.flush();
    sender.table("weather_sensor")
            .symbol("id", "dubai2")
            .doubleColumn("temperature", 41.2)
            .doubleColumn("humidity", 0.34)
            .atNow();
    sender.flush();
}
```

### Automatic flushing

To avoid accumulating very large buffers, the client will - by default - flush
the buffer automatically.

HTTP auto-flushing is triggered when appending a row to the internal buffer and
the buffer either:

- Reaches 75,000 rows
- Hasn't been flushed for 1 second.

Both parameters control batching and can be customized. Larger batches can
improve throughput, but can increase lag between data ingestion and visibility
in a target table. Smaller batches can reduce this lag, but can also reduce
throughput.

A configuration string example that auto-flushes every 10 rows or every 10
seconds, whichever comes first:

`http::addr=localhost:9000;auto_flush_rows=10;auto_flush_interval=10000;`

An example with auto-flushing disabled:

`http::addr=localhost:9000;auto_flush=off;`

TCP auto-flushing is triggered when appending a row to the internal sender
buffer and the buffer is full.

Auto-flushing is also triggered when the client is being closed. Be aware that
retrying of failed requests is disabled when flushing on close.

## Error handling

The HTTP transport supports automatic retries for failed requests deemed
recoverable. Recoverable errors include network errors, some server errors, and
timeouts, while non-recoverable errors encompass invalid data, authentication
errors, and other client-side errors.

Retrying is particularly beneficial during network issues or when the server is
temporarily unavailable. The retrying behavior can be configured through the
`retry_timeout` configuration option or via the builder API with
`retryTimeoutMillis(long timeoutMillis)`. The client continues to retry
recoverable errors until they either succeed or the specified timeout is
reached. Upon reaching the timeout, the client ceases retry attempts and throws
`LineSenderException`.

When utilizing the HTTP transport, the client can be reused after receiving an
error. Conversely, a client using TCP transport should be discarded after an
error, necessitating the creation of a new client.

Retrying is disabled for failed requests when executing a flush upon closure.

The TCP transport lacks support for error propagation from the server. In such
cases, the server merely closes the connection upon encountering an error, which
manifests as a `LineSenderException` on the client side. Consequently, the
client receives no additional error information from the server. This limitation
significantly contributes to the preference for HTTP transport over TCP
transport.

### Exactly-once delivery vs at-least-once delivery

The retrying behavior of the HTTP transport can lead to some data being sent to
the server more than once.

**Example**: Client sends a batch to the server, the server receives the batch,
processes it, but fails to send a response back to the client due to a network
error. The client will retry sending the batch to the server. This means the
server will receive the batch again and process it again. This can lead to
duplicated rows in the server.

The are two ways to mitigate this issue:

- Use [QuestDB deduplication feature](/docs/concept/deduplication/) to remove
  duplicated rows. QuestDB server can detect and remove duplicated rows
  automatically, resulting in exactly-once processing. This is recommended when
  using the HTTP transport with retrying enabled.
- Disable retrying by setting `retry_timeout` to 0. This will make the client
  send the batch only once, failed requests will not be retried and the client
  will receive an error. This effectively turns the client into an at-most-once
  delivery.

## Designated timestamp considerations

The concept of [designated timestamp](/docs/concept/designated-timestamp/) is
important when ingesting data into QuestDB.

There are two ways to assign a designated timestamp to a row:

1. User-assigned timestamp: The client assigns a specific timestamp to the row.

   ```java
   java.time.Instant timestamp = Instant.now(); // or any other timestamp
   sender.table("weather_sensor")
           .symbol("id", "toronto1")
           .doubleColumn("temperature", 23.5)
           .doubleColumn("humidity", 0.49)
           .at(timestamp);
   ```

   The `Instant` class is part of the `java.time` package and is used to
   represent a specific moment in time. The `sender.at()` method can accept a
   long timestamp representing the elapsed time since the beginning of the
   [Unix epoch](https://en.wikipedia.org/wiki/Unix_time), as well as a
   `ChronoUnit` to specify the time unit. This approach is useful in
   high-throughput scenarios where instantiating an `Instant` object for each
   row is not feasible due to performance considerations.

2. Server-assigned timestamp: The server automatically assigns a timestamp to
   the row based on the server's wall-clock time. Example:
   ```java
   sender.table("weather_sensor")
           .symbol("id", "toronto1")
           .doubleColumn("temperature", 23.5)
           .doubleColumn("humidity", 0.49)
           .atNow();
   ```

We recommended to use User-assigned timestamps when ingesting data into QuestDB.
Using Server-assigned hinder the ability to deduplicate rows which is
[important for exactly-once processing](#exactly-once-delivery-vs-at-least-once-delivery).

:::note

QuestDB works best when rows are ingested in chronological order. This means
rows with older timestamps are ingested before rows with newer timestamps.

:::

## Table and column auto-creation

When sending data to a table that does not exist, the server will create the
table automatically. This also applies to columns that do not exist. The server
will use the first row of data to determine the column types.

If the table already exists, the server will validate that the columns match the
existing table. If the columns do not match, the server will return a
non-recoverable error which is propagated to the client as a
`LineSenderException`.

If you're using QuestDB Enterprise, you must grant further permissions to the
authenticated user:

```sql
CREATE SERVICE ACCOUNT ingest_user; -- creates a service account to be used by a client
GRANT ilp, create table TO ingest_user; -- grants permissions to ingest data and create tables
GRANT add column, insert ON all tables TO ingest_user; -- grants permissions to add columns and insert data to all tables
--  OR
GRANT add column, insert ON table1, table2 TO ingest_user; -- grants permissions to add columns and insert data to specific tables
```

Read more setup details in the
[Enterprise quickstart](/docs/guides/enterprise-quick-start/#4-ingest-data-influxdb-line-protocol)
and the [role-based access control](/docs/operations/rbac/) guides.

## Configuration options

Client can be configured either by using a configuration string as shown in the
examples above, or by using the builder API.

The builder API is available via the `Sender.builder(Transport transport)`
method.

When using the configuration string, the following options are available:

### HTTP transport authentication

- `username` : Username for HTTP basic authentication.
- `password` : Password for HTTP basic authentication.
- `token` : Bearer token for HTTP authentication.

### TCP transport authentication

- `username`: Username for TCP authentication.
- `token`: Token for TCP authentication.

### TLS encryption

TLS in enabled by selecting the `https` or `tcps` protocol. The following
options are available:

- `tls_roots` : Path to a Java keystore file containing trusted root
  certificates. Defaults to the system default trust store.
- `tls_roots_password` : Password for the keystore file. It's always required
  when `tls_roots` is set.
- `tls_verify` : Whether to verify the server's certificate. This should only be
  used for testing as a last resort and never used in production as it makes the
  connection vulnerable to man-in-the-middle attacks. Options are `on` or
  `unsafe_off`. Defaults to `on`.

### Auto-flushing

- `auto_flush` : Global switch for the auto-flushing behavior. Options are `on`
  or `off`. Defaults to `on`.
- `auto_flush_rows` : The number of rows that will trigger a flush. This option
  is supported for HTTP transport only. Defaults to 75,000.
- `auto_flush_interval` : The time in milliseconds that will trigger a flush.
  Defaults to 1000. This option is support for HTTP transport only.

The TCP transport for a client automatically flushes when its buffer is full.
The TCP transport utilizes a fixed-size buffer, and its maximum size is the same
as the initial size. Thus, the option `init_buf_size` (see below) effectively
controls the auto-flushing behavior of the TCP transport.

### Buffer

- `init_buf_size` : The initial size of the buffer in bytes. Default: 65536
  (64KiB)
- `max_buf_size` : The maximum size of the buffer in bytes. Default: 104857600
  (100MiB) This option is support for HTTP transport only. TCP transport uses a
  fixed-size buffer and its maximum size is the same as the initial size.

### HTTP Transport

- `retry_timeout` : The time in milliseconds to continue retrying after a failed
  HTTP request. The interval between retries is an exponential backoff starting
  at 10ms and doubling after each failed attempt up to a maximum of 1 second.
  Default: 10000 (10 seconds)
- `request_timeout` : The time in milliseconds to wait for a response from the
  server. This is in addition to the calculation derived from the
  `request_min_throughput` parameter. Default: 10000 (10 seconds)
- `request_min_throughput` : Minimum expected throughput in bytes per second for
  HTTP requests. If the throughput is lower than this value, the connection will
  time out. This is used to calculate an additional timeout on top of
  `request_timeout`. This is useful for large requests. You can set this value
  to `0` to disable this logic.

## Other considerations

- The Sender is not thread-safe. For multiple threads to send data to QuestDB,
  each thread should have its own Sender instance. An object pool can also be
  used to re-use Sender instances.
- The Sender instance has to be closed after it is no longer in use. The Sender
  implements the `java.lang.AutoCloseable` interface, and therefore the
  [try-with-resource](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
  pattern can be used to ensure that the Sender is closed.
- The method `flush()` can be called to force sending the internal buffer to a
  server, even when the buffer is not full yet.

### Health check

To monitor your active connection, there is a `ping` endpoint:

```shell
curl -I http://localhost:9000/ping
```

Returns (pong!):

```shell
HTTP/1.1 204 OK
Server: questDB/1.0
Date: Fri, 2 Feb 2024 17:09:38 GMT
Transfer-Encoding: chunked
Content-Type: text/plain; charset=utf-8
X-Influxdb-Version: v2.7.4
```

Determine whether an instance is active and confirm the version of InfluxDB Line
Protocol with which you are interacting.
