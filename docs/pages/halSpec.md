# xDSL HAL Documentation

## Version History

| Date | Comment | Version |
| --- | --- | --- |
| 2026-08-24 | Initial release of this specification | 1.0.0 |

## Acronyms

- `HAL` \- Hardware Abstraction Layer
- `RDK-B` \- Reference Design Kit for Broadband Devices
- `xDSL` \- the family of Digital Subscriber Line technologies, ADSL through VDSL2
- `ADSL` \- Asymmetric Digital Subscriber Line
- `VDSL` \- Very high speed Digital Subscriber Line
- `FAST` \- ITU-T G.9700/G.9701, marketed as G.fast
- `ATM` \- Asynchronous Transfer Mode
- `PTM` \- Packet Transfer Mode
- `TR-181` \- the Broadband Forum `Device:` data model, cited at version `Device:2.13` as identified in [halSpecDetailed.md](halSpecDetailed.md)
- `JSON` \- JavaScript Object Notation
- `TCP` \- Transmission Control Protocol
- `SELT` \- Single-Ended Line Test
- `QLN` \- Quiet Line Noise, a line diagnostic result
- `UER` \- Uncalibrated Echo Response, a SELT result

## Description

The xDSL HAL is the contract between `RdkXdslManager` and a vendor's DSL implementation. **The
contract is a JSON schema carried over a TCP socket, not a C header linked into the caller**, which is
why there is no `rdkb-halif-xdsl` repository: the interface definition is
`hal_schema/xdsl_hal_schema.json`, and the rest of this repository is the manager in front of it.

```mermaid
flowchart TD
    Mgr["RdkXdslManager"] <--> Cli["json_hal_client"]
    Cli <--> Socket["JSON HAL socket (TCP port 4098)"]
    Socket <--> Srv["Vendor JSON HAL Server"]
```

**Schema Consumer:** The schema `hal_schema/xdsl_hal_schema.json` is consumed by `RdkXdslManager`
and by no other component. `RdkXdslManager` is the only client of the JSON HAL socket and is the
caller throughout this document. It translates each TR-181 parameter read and write into a JSON
message exchange with the vendor's server, and other RDK-B components, such as WAN Manager, reach
DSL state through the TR-181 data model the manager publishes rather than through the schema.

The schema defines parameters under four TR-181 families — `Device.DSL`, `Device.FAST`,
`Device.ATM` and `Device.PTM` — and the ATM and PTM trees carry the DSL line into the WAN data path.
`API Surface` gives the size of each family. Per-parameter detail is in
[halSpecDetailed.md](halSpecDetailed.md), which is to this schema what inline Doxygen is to a C
header.

Diagrams are fenced Mermaid, which renders on GitHub; the generated Doxygen site shows their source
text instead.

## Optional Components

There is no schema or configuration variant: the repository ships one schema file and one client
configuration file. The manager reads `/etc/rdk/conf/xdsl_manager_conf.json`, which names the
deployed schema `/etc/rdk/schemas/xdsl_hal_schema.json` in `hal_schema_path` and the server port
`4098` in `server_port`. A vendor installs the schema at that path, and a `getSchemaResponse` is
expected to return it.

Three surfaces are optional:

- **ATM and PTM Status Subscription:** `xdsl_hal_init()` subscribes only the DSL line link status.
  `atm_hal_subscribeStatusEvent()` and `ptm_hal_subscribeStatusEvent()` are called separately by the
  middle layer when those link types are in use.
- **Optional Parameters:** `getParameterOptionalList` and `setParameterOptionalList` hold the reads
  and writes a platform may decline; `Platform or Product Customization` gives their sizes.
- **Line Diagnostics:** The SELT, QLN and UER parameters under `Device.DSL` are exercised only when
  a diagnostic is requested, and `dslDiagnosticsStateEnumList` includes `None` for a line on which
  no diagnostic has run.

Structural defects in the schema that affect individual branches are listed in `Contract Defects` in
[halSpecDetailed.md](halSpecDetailed.md).

## Component Runtime Execution Requirements

The xDSL HAL is a vendor process reached over a TCP socket, not a library loaded into the manager.
Failure to meet these requirements will result in undefined behaviour.

### Initialization and Startup

`xdsl_hal_init()` in `source/TR-181/integration_src.shared/xdsl_hal.c` brings the client up in a
fixed order before any parameter is read or written, and a caller must not vary it:

- `json_hal_client_init("/etc/rdk/conf/xdsl_manager_conf.json")` — loads `hal_schema_path` and
  `server_port` from the configuration file named by `XDSL_JSON_CONF_PATH`. It does not open the
  socket.
- `json_hal_client_run()` — starts the client socket thread, which connects to the server.
- `json_hal_is_client_connected()` — polled until the connection is up. The socket thread connects
  asynchronously, so this poll is the only way to learn that it succeeded.
- `subscribe_dsl_link_event()` — registers the DSL line link-status subscription. Events published
  before this point are not delivered.

**Connection Retry:** The connection retry is bounded. `xdsl_hal_init()` polls up to
`HAL_CONNECTION_RETRY_MAX_COUNT` (`10`) times, one second apart — a fixed interval, not an
exponential backoff. If the connection is not up after the tenth poll, `xdsl_hal_init()` returns an
error and the interface is unusable until it is called again.

A failure to subscribe the link event is logged but does not fail initialization, so a caller that
depends on link-status notifications confirms that they arrive.

Third party vendors must have their JSON HAL server listening on port `4098` before the manager
starts, because the manager waits ten seconds at most for the connection.

### Threading Model

`json_hal_client_run()` starts the client socket thread, which owns the socket for the lifetime of
the client: it receives responses, matches them to waiting requests and dispatches published events
to the registered callback.

- Event callbacks run **on the client socket thread**, not the caller's. A callback must be safe to
  run concurrently with the caller's own work and must not block, because the same thread receives
  the next response.
- A caller serializes each complete request exchange per client instance — from
  `json_hal_client_get_request_header()` to the return of `json_hal_client_send_and_get_reply()` or
  `json_hal_client_send_and_get_reply_with_timeout()`, and every `json_hal_client_subscribe_event()`
  call, which runs a whole exchange — because the transport assigns request identifiers, records
  pending requests and writes the socket without synchronization.

Vendors may implement internal threading and event mechanisms in the server to meet their
operational requirements, provided each response carries the `reqId` of the request it answers.

### Process Model

**Process Separation:** The manager and the vendor implementation are separate processes
communicating over a TCP socket on the loopback interface. There is no vendor library in the
manager's address space, no shared memory and no direct function call. The isolation covers memory,
not data: received names and values are copied into fixed manager buffers, and the name parse writes
a `%64s` field plus its terminator into a 64-byte buffer, so an over-long name can corrupt manager
memory through the parsing path. `Memory Model` gives the size limits of the parameter buffers.

One client instance per manager process is expected. The server accepts more than one connection,
with a listen backlog of `32`, but this contract gives a second concurrent client no meaning.

The two processes restart independently. A caller that has restarted, or that suspects the server
has, re-establishes its subscriptions rather than assuming they survived.

### Memory Model

No memory is shared across the HAL boundary; only serialized JSON documents cross it. Per exchange
the caller holds a request and a reply `json_object`, and releases both with `json_object_put()`.

#### Caller Responsibilities

- Release the request from `json_hal_client_get_request_header()` and the reply returned by
  `json_hal_client_send_and_get_reply()` once the exchange completes, on the failure path as well as
  the success path. A failed exchange leaves the reply handle unwritten, so initialise it to `NULL`
  before the call.
- `json_hal_get_param()` copies a parameter's name and value into the caller's `hal_param_t`, and
  that copy outlives the reply. A string or object read directly from the reply through `json-c`,
  such as the string `json_object_get_string()` returns, belongs to the reply; copy what is kept
  before releasing the reply.
- Treat the message buffer passed to an event callback as valid only for the duration of the
  callback.
- Keep each parameter name to at most 255 bytes and each value to at most 2047 bytes, leaving room
  for the terminating NUL in `hal_param_t`, and check the length before `json_hal_add_param()`.
  Treat a `json_hal_get_param()` field whose final byte is not NUL as truncated and unterminated,
  and read that parameter directly from the reply instead. `Contract Defects` in
  [halSpecDetailed.md](halSpecDetailed.md) describes the mechanism.

#### Module Responsibilities

- The transport owns its receive buffer. A reply larger than one 16384-byte read is reassembled
  across reads; a request must arrive in a single read, or the server discards it without replying.
  `Contract Defects` in [halSpecDetailed.md](halSpecDetailed.md) records both limits.
- The transport owns the request-tracking list that matches replies to waiting callers and releases
  it in `json_hal_client_terminate()`.
- Vendor servers release whatever they allocate per request.
- Strings crossing the interface are JSON strings and must be valid UTF-8 and zero-terminated once
  decoded.

### Power Management Requirements

This interface places no power-management obligation on the HAL: the schema carries no host power
control, and the manager's HAL integration takes part in no suspend or resume path. A vendor
implementation does not register for power events on the manager's behalf.

`Device.DSL.Line.{i}.PowerManagementState` reports the DSL line's own ITU-T link power state, `L0` to
`L4` from `powerManagementStateEnumList`. It is read-only status of the copper line, not a control of
the gateway's power state, and is in neither set-parameter list.

### Asynchronous Notification Model

The caller sends `subscribeEvent` and the server answers with `result`; thereafter the server sends
unsolicited `publishEvent` messages whenever the subscribed parameter changes.

- **Subscribable Parameters:** Two events only. `subscribeEventSupportedList` admits
  `dslLineLinkStatus` and `fastLineLinkStatus`, that is `Device.DSL.Line.{i}.LinkStatus` and
  `Device.FAST.Line.{i}.LinkStatus`.
- **Notification Types:** `subscribe_dsl_link_event()` requests `onChange`. This schema has no
  `onChangeSync` or `onChangeSyncTimeout` notification type.
- **Subscription Status:** `json_hal_client_subscribe_event()` returns `RETURN_OK` without reading
  `Result.Status`, so success shows that the callback is registered, not that the vendor accepted
  the subscription, and the call leaves no reply for `json_hal_get_result_status()`. Callback
  registration happens only inside that function, so a caller that also needs the status sends its
  own `subscribeEvent` through `json_hal_client_send_and_get_reply()` and reads that reply; the
  server can record it as a further subscription. `Event Model` in
  [halSpecDetailed.md](halSpecDetailed.md) compares the two routes.

Events are delivered into the callback surface declared in
`source/TR-181/integration_src.shared/xdsl_hal.h`: `dsl_link_status_callback`, installed by
`xdsl_hal_registerDslLinkStatusCallback()`, and `xtm_status_callback`, installed by
`atm_hal_registerStatusCallback()` and `ptm_hal_registerStatusCallback()`. One event handler receives
every `publishEvent` and dispatches on the parameter path to its DSL line, ATM or PTM branch, so a
new subscription's events arrive at that same entry point. `Event Model` in
[halSpecDetailed.md](halSpecDetailed.md) compares the schema's notification types with the
transport's.

### Blocking calls

`json_hal_client_send_and_get_reply()` and `json_hal_client_send_and_get_reply_with_timeout()` are
synchronous: each blocks the calling thread until the matching reply arrives or the wait expires, so
HAL requests are not issued from a thread with latency obligations of its own. The untimed form
waits about 10 seconds and the timed form about 10 to 120 seconds. These are nominal budgets, not
hard deadlines, so a caller that needs a deadline enforces its own. `RdkXdslManager` uses the
untimed form for every request. Connection set-up blocks for up to ten one-second polls, as
`Initialization and Startup` describes. `publishEvent` delivery is asynchronous and no caller blocks
waiting for it. A caller must not issue a request from an event callback, because the socket thread
the callback runs on could then neither receive the reply nor expire the request.
`Blocking and timeouts` in [halSpecDetailed.md](halSpecDetailed.md) gives the budget arithmetic.

### Internal Error Handling

A `result` message answers `setParameters`, `deleteObject` and `subscribeEvent`, and its
`Result.Status` takes one of the four `resultStatusEnumList` values:

| `Result.Status` | Meaning | Caller action |
| --- | --- | --- |
| `Success` | The vendor handler reported success. This acknowledges the request; it does not prove that a write was applied. This is the schema default. | Proceed; where the effect of a write matters, read the parameter back. |
| `Failed` | The vendor handler reported failure. The transport sends it when a registered action handler returns failure, and a vendor handler may also write it directly. It does not prove that nothing was applied. | Treat as a runtime failure. Before resending a `setParameters`, `deleteObject` or `subscribeEvent`, read the affected state back (`Retry behaviour on the write path` in [halSpecDetailed.md](halSpecDetailed.md)). |
| `Invalid Argument` | The request was malformed or a value was outside its constraint. The pinned transport never constructs it; only a vendor handler that writes it directly sends it. | Correct the request against [halSpecDetailed.md](halSpecDetailed.md); do not retry it unchanged. |
| `Not Supported` | The server has no handler for the action, or the vendor does not support the parameter or action. A server built with `JSON_SCHEMA_VALIDATION_ENABLED` also sends it in place of a handler's reply that fails schema validation, after the handler has run, so a write, delete or subscription may already have taken effect. | Do not conclude from this status alone that nothing was applied or that the parameter is absent. Read the affected state back before resending a `setParameters`, `deleteObject` or `subscribeEvent`, or ceasing to use the parameter (`Retry behaviour on the write path` in [halSpecDetailed.md](halSpecDetailed.md)). |

The transport's handling of a request decides what a caller sees:

- **Malformed Requests:** A malformed request gets no reply. The pinned server validates no inbound
  message: a request that is not parseable JSON, or that lacks `reqId` or `action`, is discarded
  without a reply, so the call ends only when its wait budget runs out (`Blocking calls`).
  `module` and `version` are not checked, and parameter values reach the vendor handler unvalidated.
- **Sender Validation:** The sender validates. A caller validates its requests against
  `hal_schema/xdsl_hal_schema.json` before sending them, and a vendor server applies the parameter
  constraints in its handlers.
- **Status Discrimination:** Read `Result.Status` directly to tell the statuses apart.
  `json_hal_get_result_status()` reduces a `result` to true or false, so `Failed`,
  `Invalid Argument` and `Not Supported` are indistinguishable through it, and `json_rpc_common.h`
  defines constants for `Success`, `Failed` and `Not Supported` only, so `Invalid Argument` is
  compared as a literal.

Vendor servers report internal failures through `Result.Status` rather than by closing the
connection or omitting a reply, and log any failure that cannot be expressed as a status with the
`reqId` of the exchange.

### Persistence Model

**No Server Persistence:** This contract asks the HAL server to persist nothing. No action commits
or reloads configuration, and no parameter requests that a value survive a restart; a
`setParameters` states the intended value of a TR-181 parameter and is answered with a status.
Persistence of the TR-181 data model belongs to `RdkXdslManager` and the CCSP framework, whose
parameter definitions are in `config/RdkXdslManager.xml`. The vendor server is the authority on
current state, so the manager reads a value back rather than assuming that a written value
persisted.

## Non functional requirements

The following non-functional requirements apply to a vendor implementation of the xDSL JSON HAL
server and to the manager that calls it.

### Logging and debugging requirements

Both sides of the interface should log errors and significant state changes so that a failure can be
triaged from the logs. Logging should use `syslog`; `printf` is discouraged unless `syslog` is
unavailable.

Logs should be categorised by the following levels, as defined by the standard Linux logging system
and listed in descending order of severity:

- **FATAL:** critical conditions, typically a crash or a failure requiring immediate attention.
- **ERROR:** non-fatal conditions that nonetheless prevent normal operation.
- **WARNING:** potentially harmful situations that are not yet errors.
- **NOTICE:** significant but non-error events.
- **INFO:** general informational messages describing normal operation.
- **DEBUG:** detail useful only when diagnosing a problem.
- **TRACE:** fine-grained tracing of internal flow.

Each entry should carry a timestamp, the level and a message describing the event. An entry
describing a message exchange should include its `reqId`, which correlates a reply with its request
(`Method Sequencing`). A vendor server should write its log file under `/var/tmp/` or
`/rdklogs/logs/`, following the RDK-B HAL convention. The manager logs through the RDK-B trace
facility.

Implementations must not log credentials, secrets, security-sensitive information, or other sensitive data.
The rule applies at every severity, `DEBUG` and `TRACE` included, and to crash dumps, telemetry and
diagnostic captures. Persistent equipment identity, such as serial numbers, hardware addresses and
vendor or firmware identifiers, is sensitive data for this interface: a log entry that refers to such
a parameter records the parameter path and the outcome, never the value.

### Memory and performance requirements

The transport's size bounds are given under `Memory Model` and the server's listen backlog under
`Process Model`. Two performance characteristics follow from the process model:

- **Round Trips:** Every read and write is an inter-process round trip. A caller that needs many
  parameters requests them in one `getParameters` message, whose `params` array exists for that
  purpose.
- **Manager Polling:** Manager polling is paced at 250 milliseconds. Where the manager waits on a
  vendor state change it polls every 250 milliseconds, which bounds how quickly a polling caller
  observes a change. A caller needing lower latency subscribes to the link-status event instead.

When idle, the interface is quiescent: no exchange occurs until a caller sends a request or the
server publishes a subscribed event.

### Quality Control

Third-party analysis tools such as `Coverity`, `Black Duck` and `Valgrind` should be used on both the
vendor server and the manager, so that memory leaks, corruption and licence-compliance defects are
found before deployment. A zero-warning policy applies to compilation, with all warnings enabled by
default in the build files.

A vendor server should also be tested against malformed and unexpected input, and such a test asserts
the transport behaviour set out under `Internal Error Handling` rather than a rejection status the
transport never constructs.

Any change to a file listed in `Source References` obliges a review of this document by the
repository's code owner, `@rdkcentral/wanmanager-maintainers`, as named in `.github/CODEOWNERS`. A
change to `hal_schema/xdsl_hal_schema.json` in particular invalidates the counts in `API Surface`
and the enumerations in `Data Structures and Defines` and `State Diagram`.

### Licensing

Licensed under the Apache License, Version 2.0 (the "License"). You may not use the files of this
repository except in compliance with the License. See `LICENSE`; `COPYING` is a symlink to the same
text. A vendor implementation of this interface is expected to be released under compatible terms.

### Build Requirements

This repository is an Autoconf, Automake and Libtool C project, `RdkXdslManager-rdkb`, and its
deliverable is the **executable `xdslmanager`** rather than a shared library: a caller runs the
manager rather than linking against it. The manager links the JSON HAL transport and its parser with
`-ljson_hal_client -ljson-c`. `configure.ac` requires a C and a C++ compiler, and the one dependency
it declares a version for is D-Bus, `dbus-1 >= 1.6.18`.

The transport's `json-c` dependency carries two figures, and either alone misleads:

- **Declared Minimum:** `json-c (0.11)`, stated in the `json-hal-library` README.
- **Upstream Build Revision:** `json-c-0.15-20200726`, named as the `json-c`
  branch in `json-hal-library`'s `cov_docker_script/component_config.json`.

**Production Deployment:** Unpatched upstream `json-c 0.11` must not be deployed in production. It
is affected by `CVE-2013-6370`, a buffer overflow; `CVE-2013-6371`, a hash-collision denial of
service; and `CVE-2020-12762`, an integer overflow leading to an out-of-bounds write, fixed in
`json-c 0.15`. The transport hands bytes read from the loopback socket to the `json-c` parser before
any validation, so a hostile or malfunctioning local peer reaches the parser first. A production
deployment links `json-c 0.15` or later, or a distribution build carrying backports of those three
fixes.

### Variability Management

Each interface is versioned using [Semantic Versioning 2.0.0](https://semver.org/spec/v2.0.0.html),
and a vendor implementation complies with a specific version of it. Adjusting the interface is an
architecture decision; vendors align their implementation with a designated version and move to a
newer one as demand requires.

**Schema Version:** The version that governs a message on the wire is the HAL schema version,
`0.0.1`, carried by the `schemaVersion` definition and required in the `version` field of every
message. The schema states its own rule for it:

> DO NOT modify the value of the version string. HAL operation cannot be performed without correct
> supported version.

A message carrying any other value is invalid against the schema. The schema version advances
independently of the repository's release tags.

**Compile-Time Flags:** No compile-time flag varies the HAL contract. The manager's configure
switches `--enable-gtestapp`, which builds the gtest binary, and `--enable-notify`, which enables
systemd notification, do not alter it. The transport flag `JSON_BLOCKING_SUBSCRIBE_EVENT` gates the
`onChangeSync` notification types, which this schema does not use.

### Platform or Product Customization

The vendor supplies the JSON HAL **server**, and the schema is the contract it satisfies.
Customization is therefore a matter of **which parameters a platform implements**, which the schema
expresses through its supported and optional lists:

| List | Entries | What a caller may conclude |
| --- | --- | --- |
| `getParameterSupportedList` | 348 | A conforming server answers these reads. |
| `getParameterOptionalList` | 45 | A platform may decline these; expect `Not Supported`. |
| `setParameterSupportedList` | 18 | The writable surface a conforming server accepts. |
| `setParameterOptionalList` | 6 | Writable only where the platform implements it. |

The lists hold object paths as well as parameters. Of the 24 set-list entries, 22 are parameter
definitions and two are the `Device.ATM.Link.{i}.` and `Device.PTM.Link.{i}.` object paths, so 22 of
369 parameter definitions are writable and the interface is predominantly read-only.
[halSpecDetailed.md](halSpecDetailed.md) gives the access of each parameter.

**Single DSL Line:** The shipped integration layer handles one DSL line. `XDSL_MAX_LINES` is `1` in
`xdsl_hal.c` and bounds the link-event subscription loop, while `config/RdkXdslManager.xml` allows
128 instances, in its `maxInstance` elements, of `Device.DSL.Line`, `Device.DSL.Channel`,
`Device.ATM.Link` and `Device.PTM.Link`. A platform supporting more than one DSL line needs that
constant revisited.

## Interface API Documentation

The interface is defined by `hal_schema/xdsl_hal_schema.json`, a JSON Schema draft-07 document and
the single authority for what may cross the interface. This section describes its shape, and
[halSpecDetailed.md](halSpecDetailed.md) documents every parameter it defines.

Using the interface takes three steps:

1. **Configure:** the server port and the schema path in `config/xdsl_manager_conf.json` match the
   deployment.
2. **Link:** the manager links `json_hal_client` from `json-hal-library`.
3. **Initialize:** the ordering in `Initialization and Startup` is followed exactly.

### Theory of operation and key concepts

Every interaction is a single JSON object sent over the socket, carrying the envelope and the payload
set out in `Data Structures and Defines`. A caller sends a request and receives either a response
action or a generic `result`.

The interface is a **projection of the TR-181 data model onto that message exchange**. There are no
functions to call and no handles to hold: a parameter is addressed by its TR-181 path as a string,
and the operations are read, write, delete, subscribe and describe.

#### Object Lifecycles

No action creates an object. TR-181 instances are addressed by path — for example
`Device.DSL.Line.{i}.` for a line, where `{i}` is the instance number — and come into existence
because the vendor's hardware or the data model layer creates them.

`deleteObject` removes an instance, but its `anyOf` admits only `dslLineObjectName`, whose pattern
is `^Device\.DSL\.Line\.\d+\.$`, so the only object a caller may delete is a DSL line instance. A
`deleteObject` naming a channel, a bonding group or an ATM or PTM link is invalid against the
schema but still reaches the vendor's `deleteObject` handler, which alone can refuse it
(`Internal Error Handling`).

#### Method Sequencing

- **Initialization First:** Initialization precedes every request. The ordering in
  `Initialization and Startup` is mandatory.
- **Schema Mismatch:** The caller sends `getSchema` before assuming the server's schema path, and
  the answer is untrusted input. `getSchemaResponse` carries the server's schema path in
  `SchemaInfo.FilePath`. The caller compares it, as a whole string, with the `hal_schema_path` it
  already holds from `config/xdsl_manager_conf.json`, and treats any other value as a configuration
  fault to report. It must not open, `stat`, read, execute, log or interpolate the returned path, or
  use it to locate the schema it validates against. The shipped example response returns a path that
  differs from the configured one, as [halSpecDetailed.md](halSpecDetailed.md) records.
- **Subscription First:** The caller sends `subscribeEvent` before expecting `publishEvent`. Events
  are delivered only to a subscribed caller, and only two parameters may be subscribed.
- **Correlation:** Correlation is by `reqId`. A response carries the `reqId` of the request it
  answers, which is how the client matches it to the waiting call. Distinct identifiers do not make
  concurrent submission safe: exchanges follow the serialization rule in `Threading Model`.

#### State-Dependent Behavior

The values a read returns depend on the line's link state, and a successful exchange does not
indicate that the line is up. Statistics and test parameters under `Device.DSL.Line.{i}.` are
meaningful only once the line has trained; while `Device.DSL.Line.{i}.LinkStatus` reports
`Initializing` or `EstablishingLink`, counters may be zero or stale.

A SELT or QLN result is defined only once `dslDiagnosticsStateEnumList` reports `Complete`, as
distinct from `Requested`, `Canceled` and its error states. `State Diagram` sets out the two status
enumerations.

### Data Structures and Defines

A JSON HAL has no header, so its public types are the schema definitions a caller constructs or
interprets. `Transport and Protocol` in [halSpecDetailed.md](halSpecDetailed.md) gives their full
constraints and the members each action requires.

- **Envelope:** Every message carries `module`, the `moduleName` `const` `xdslhal`; `version`, the
  `schemaVersion` `const` `0.0.1`; `action`, one of the eleven members in `API Surface`; and
  `reqId`, a numeric string matching `^[0-9]+$` and not an integer, so `"reqId": "100001"` is valid
  and `"reqId": 100001` is not.
- **Payload Shapes:** `params` is an array of parameter entries, `Result` an object whose only
  property is `Status`, a `resultStatusEnumList` member, and `SchemaInfo` an object whose only
  property is `FilePath`; the `API Surface` table shows which action carries which. The `FilePath`
  pattern, `^(.+)/([^/]+)$`, requires only a `/` with characters on both sides, so it gives a caller
  no path protection and the comparison rule in `Method Sequencing` applies.
- **Parameter Datatypes:** An entry's `type` label is one of seven values — `boolean`, `string`,
  `int`, `unsignedInt`, `unsignedLong`, `hexBinary` and `base64` — each mapping to an `eParamType`
  member in `json-hal-library`, which adds a `long` this schema does not use; a `hexBinary` value
  also matches `typeHex`, an even number of hexadecimal digits. Three bonding-group definitions
  constrain `type` with an inline enumeration instead (`Contract Defects` in
  [halSpecDetailed.md](halSpecDetailed.md)), and `Parameter Reference` gives each parameter's type.
- **Client Types:** `hal_config_t` holds the client configuration, including the `hal_schema_path`
  and `server_port` values read from the configuration file, and a parameter in transit is a
  `hal_param_t`, whose size limits are given under `Memory Model`.
- **Named Enumerations:** The schema defines eleven enumerated value lists, catalogued in the
  `Enumeration Appendix` of [halSpecDetailed.md](halSpecDetailed.md); `State Diagram` compares the
  two that describe line and link state.

### API Surface

**Action Values:** The complete action vocabulary is eleven members; there is nothing else a
caller may put in the `action` field.

| Action | Direction | Payload bound by `allOf` |
| --- | --- | --- |
| `getSchema` | client to server | none — bare envelope |
| `getSchemaResponse` | server to client | `SchemaInfo` |
| `getParameters` | client to server | `params` |
| `getParametersResponse` | server to client | `params` |
| `setParameters` | client to server | `params` |
| `deleteObject` | client to server | `params` |
| `subscribeEvent` | client to server | `params` |
| `publishEvent` | server to client, unsolicited | `params` |
| `getActiveSubscriptions` | client to server | none — bare envelope |
| `getActiveSubscriptionsResponse` | server to client | none — bare envelope |
| `result` | server to client | `Result` |

Three facts about this vocabulary are easy to get wrong:

- **No Write Response:** There is no `setParametersResponse`. A write is acknowledged by the generic
  `result` action.
- **Object Deletion:** The `deleteObject` action is instantiable in this schema, unlike in the GPON
  and telco voice JSON HALs, but for DSL line instances only, per `Object Lifecycles`.
- **Active Subscriptions Response:** The `getActiveSubscriptionsResponse` action has no payload
  binding, so a caller must not infer a `params` array from the response name.

The **data surface** is the four TR-181 family trees below; the counts are parameter definitions,
not logical paths.

| Family tree | Parameter definitions | Principal objects |
| --- | --- | --- |
| `Device.DSL` | 241 | `Line`, `Channel`, `BondingGroup`, `Diagnostics` |
| `Device.FAST` | 61 | `Line` and its statistics |
| `Device.ATM` | 45 | `Link`, its QoS and statistics |
| `Device.PTM` | 22 | `Link` and its statistics |

Those 369 definitions cover **512 logical TR-181 paths**, because 20 use regular-expression
alternation and match more than one path each. Twenty-five object definitions and eleven enumerated
lists complete the 422 definitions. [halSpecDetailed.md](halSpecDetailed.md) documents every
parameter definition with its expansion, type, constraint, access and description, together with the
object index, the enumeration appendix, worked message examples for each workflow and the schema's
contract defects.

### Sequence Diagram

The exchange below brings the client up, reads a parameter, then subscribes and receives an event.
Every function named is declared by `json-hal-library`.

```mermaid
sequenceDiagram
    participant Mgr as RdkXdslManager
    participant Cli as json_hal_client
    participant Srv as Vendor JSON HAL Server
    Mgr->>Cli: json_hal_client_init(conf_path)
    Note over Cli: reads hal_schema_path and server_port
    Mgr->>Cli: json_hal_client_run()
    Cli->>Srv: TCP connect on port 4098
    Mgr->>Cli: json_hal_is_client_connected()
    Note over Mgr,Cli: polled up to 10 times, 1s apart
    Cli-->>Mgr: TRUE
    Mgr->>Cli: json_hal_client_get_request_header("getParameters")
    Mgr->>Cli: json_hal_client_send_and_get_reply(request, reply)
    Cli->>Srv: module, version, action, reqId with params
    Srv-->>Cli: getParametersResponse with the same reqId
    Cli-->>Mgr: reply
    Mgr->>Cli: json_hal_get_param(reply, 0, GET_RESPONSE_MESSAGE, param)
    Mgr->>Cli: json_hal_client_subscribe_event(cb, name, "onChange")
    Cli->>Srv: subscribeEvent on Device.DSL.Line.1.LinkStatus
    Srv-->>Cli: result with the same reqId
    Note over Cli: reply consumed and released inside the call
    Cli-->>Mgr: RETURN_OK, and no reply object
    Note over Srv: line state changes later
    Srv-->>Cli: publishEvent, unsolicited
    Cli-->>Mgr: event callback on the client socket thread
```

Two details in the diagram are contractual:

- **Subscribe Arguments:** The `json_hal_client_subscribe_event()` function takes three arguments —
  the callback, the event name and the notification type — as `json_hal_client.h` declares. The
  two-argument form in the `json-hal-library` README does not compile.
- **Reply Helper:** The reply's `action` decides the helper. Read a `getParametersResponse` entry
  with `json_hal_get_param()`, and call `json_hal_get_result_status()` only on a `result` reply,
  such as the reply to `setParameters` or `deleteObject`, checking its return code before reading
  `status`, which it leaves unwritten on any other reply.

`Asynchronous Notification Model` sets out what the subscription's `RETURN_OK` proves.

### State Diagram

This interface exposes **two different status enumerations**, bound to different parameters and
describing different things:

| Enumeration | Bound to | Describes |
| --- | --- | --- |
| `linkStatusEnumList` | `Device.DSL.Line.{i}.LinkStatus`, `Device.FAST.Line.{i}.LinkStatus` | the DSL link's training progression |
| `lineStatusEnumList` | `Device.DSL.Line.{i}.Status` | the TR-181 generic interface status |

**Training Progression:** The training progression is `linkStatusEnumList`, and it is the one a
caller monitoring a line watches: `Device.DSL.Line.{i}.LinkStatus` is the only DSL parameter a
caller can subscribe to rather than poll. Its six values are below.

```mermaid
stateDiagram-v2
    state "Disabled" as Disabled
    state "NoSignal" as NoSignal
    state "Initializing" as Initializing
    state "EstablishingLink" as EstablishingLink
    state "Up" as Up
    state "Error" as Error
    note right of Disabled : Six reported values of linkStatusEnumList.
    note right of NoSignal : The schema constrains the value set, not the order.
    note right of Up : Any value may follow any other.
    note right of Error : Each change is delivered by publishEvent.
```

**Value Order:** No edge is drawn, because the schema constrains the set of values, not their order.
A real line usually moves from `Initializing` through `EstablishingLink` to `Up`, but a caller
treats any reported value as possible at any time and does not, for instance, wait for
`EstablishingLink` as a precondition for `Up`. With `notificationType` `onChange`, each change in
the reported value produces a `publishEvent` carrying the new value.

`lineStatusEnumList` carries the seven standard TR-181 interface status values — `Up`, `Down`,
`Unknown`, `Dormant`, `NotPresent`, `LowerLayerDown` and `Error`. It reports whether the interface
object is usable, not how far the line has got through training: a caller reads it for interface
reachability and reads `LinkStatus` for line progress.

## Source References

- [rdkcentral/xdsl-manager/hal_schema/xdsl_hal_schema.json](https://github.com/rdkcentral/xdsl-manager/blob/main/hal_schema/xdsl_hal_schema.json) — the interface definition
- [rdkcentral/xdsl-manager/hal_schema/example_getSchemaResponse_msg.json](https://github.com/rdkcentral/xdsl-manager/blob/main/hal_schema/example_getSchemaResponse_msg.json) — the shipped schema-location response
- [rdkcentral/xdsl-manager/config/xdsl_manager_conf.json](https://github.com/rdkcentral/xdsl-manager/blob/main/config/xdsl_manager_conf.json) — client configuration: schema path and server port
- [rdkcentral/xdsl-manager/config/RdkXdslManager.xml](https://github.com/rdkcentral/xdsl-manager/blob/main/config/RdkXdslManager.xml) — the manager's TR-181 data model
- [rdkcentral/xdsl-manager/source/TR-181/integration_src.shared/xdsl_hal.c](https://github.com/rdkcentral/xdsl-manager/blob/main/source/TR-181/integration_src.shared/xdsl_hal.c) — client start-up, exchanges and event dispatch
- [rdkcentral/xdsl-manager/source/TR-181/integration_src.shared/xdsl_hal.h](https://github.com/rdkcentral/xdsl-manager/blob/main/source/TR-181/integration_src.shared/xdsl_hal.h) — callback types and registration functions
- [rdkcentral/xdsl-manager/source/TR-181/integration_src.shared/xdsl_apis.c](https://github.com/rdkcentral/xdsl-manager/blob/main/source/TR-181/integration_src.shared/xdsl_apis.c) — link-status notification to WAN Manager
- [rdkcentral/xdsl-manager/source/TR-181/integration_src.shared/Makefile.am](https://github.com/rdkcentral/xdsl-manager/blob/main/source/TR-181/integration_src.shared/Makefile.am) — transport link flags
- [rdkcentral/xdsl-manager/source/RdkXdslManager/Makefile.am](https://github.com/rdkcentral/xdsl-manager/blob/main/source/RdkXdslManager/Makefile.am) — the `xdslmanager` executable
- [rdkcentral/xdsl-manager/configure.ac](https://github.com/rdkcentral/xdsl-manager/blob/main/configure.ac) — package, dependencies and configure switches
- [rdkcentral/xdsl-manager/.github/CODEOWNERS](https://github.com/rdkcentral/xdsl-manager/blob/main/.github/CODEOWNERS) — document owner
- [rdkcentral/xdsl-manager/LICENSE](https://github.com/rdkcentral/xdsl-manager/blob/main/LICENSE) — licence
- [rdkcentral/xdsl-manager/CHANGELOG.md](https://github.com/rdkcentral/xdsl-manager/blob/main/CHANGELOG.md) — release history
- [rdkcentral/json-hal-library/json_hal_client.h](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/json_hal_client.h) — client API
- [rdkcentral/json-hal-library/json_hal_client.c](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/json_hal_client.c)
  — reply wait budgets, subscription and result helpers
- [rdkcentral/json-hal-library/json_hal_common.h](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/json_hal_common.h) — `hal_config_t`, `hal_param_t` and `eParamType`
- [rdkcentral/json-hal-library/json_hal_common.c](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/json_hal_common.c) — parameter helpers
- [rdkcentral/json-hal-library/json_hal_server.h](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/json_hal_server.h) — server API and `JSON_BLOCKING_SUBSCRIBE_EVENT`
- [rdkcentral/json-hal-library/json_hal_server.c](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/json_hal_server.c) — request dispatch and generic replies
- [rdkcentral/json-hal-library/json-rpc-common/json_rpc_common.h](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/json-rpc-common/json_rpc_common.h) — status constants and buffer size
- [rdkcentral/json-hal-library/tcp_client.h](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/tcp_client.h)
  — loopback server address
- [rdkcentral/json-hal-library/tcp_client.c](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/tcp_client.c) — reply reassembly
- [rdkcentral/json-hal-library/tcp_server.c](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/tcp_server.c) — loopback bind, listen backlog and request reads
- [rdkcentral/json-hal-library/CMakeLists.txt](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/CMakeLists.txt) — `JSON_SCHEMA_VALIDATION_ENABLED`
- [rdkcentral/json-hal-library/README.md](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/README.md) — declared `json-c` minimum
- [rdkcentral/json-hal-library/cov_docker_script/component_config.json](https://github.com/rdkcentral/json-hal-library/blob/86a0a300b976f8e3295064af8fb3fd1c793c9e64/cov_docker_script/component_config.json) — `json-c` revision built upstream

