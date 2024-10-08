# OpAMP implementation in Go

[Open Agent Management Protocol (OpAMP)](https://github.com/open-telemetry/opamp-spec)
is a network protocol for remote management of large fleets of data collection Agents.

OpAMP allows Agents to report their status to and receive configuration from a server and to receive agent package updates from the server.

OpAMP provides several key features:

- Agent reporting (e.g., type, version, and host OS details) to the OpAMP control plane.

- Configuration management.

- Telemetry ingestion into an OTLP-compliant observability backend.

- Auto-updating capabilities.

- Connection credentials management.

The protocol is vendor-agnostic, so the Server can remotely monitor and manage a fleet of different Agents that implement OpAMP, including a fleet of
mixed agents from different vendors.

This repository is work-in-progress of an OpAMP implementation in Go, **a simple OpAMP control plane** consisting of an OpAMP server and let an OpenTelemetry Collector connect to it via **an OpAMP Supervisor**.

## OpAMP
Observability vendors and cloud providers offer proprietary (độc quyền) solutions for agent management. In the open source observability space, there is an emerging standard that you can use for agent management: Open Agent Management Protocol (OpAMP).

The OpAMP specification defines how to manage a fleet of telemetry data agents. These agents can be OpenTelemetry collectors, Fluent Bit or other agents in any arbitrary combination.
OpAMP is a client/server protocol that supports communication over HTTP and over WebSockets:

* **The OpAMP server** is part of the control plane and acts as the orchestrator, managing telemetry agents by sending instructions through the OpAMP protocol.
* **The OpAMP client** is part of the data plane, with 2 options are in-process implementation and out-of-process implementation with the help of a supervisor taking care of the OpAMP specific communication with the OpAMP server and at the same time controls the telemetry agent (configuring and upgrading purposes). 
* The Supervisor is a part of the Control Plane, managing the state and behavior of the OpAMP server and by extension, the OTel collectors. Note that the **supervisor/telemetry communication** is not part of OpAMP.

![img](https://opentelemetry.io/docs/collector/img/opamp.svg)

In this implementation, 
- **The Supervisor** (exists as a separate binary) manages the OpenTelemetry Collector by writing configurations to `config.yaml`.
- **The OpAMP Client** in the Supervisor communicates with the OpAMP Backend, sending and receiving data.
- **The OpenTelemetry Collector** reads its configuration from `config.yaml` and operates based on that configuration.
- The Supervisor can control the lifecycle of the OpenTelemetry Collector (start, stop, etc.) by sending commands (e.g., exec, kill) and handling outputs (stdout/stderr).
- **The Telemetry Backend** receives telemetry data (logs and metrics) from the OpenTelemetry Collector via OTLP.
- Finally, **the OpAMP Backend** receives data from the OpAMP Client for management purposes.  

![alt text](image.png)

Here is a system consisting of OpAMP server supervisor and the basic server that will be implemented in this project 

![img](https://opentelemetry.io/blog/2022/opamp/opamp-server-supervisor-agent-relations.png)

1. The OpenTelemetry Collector, configured with pipeline(s) to:
* (A) receive signals from downstream sources
* (B) export signals to upstream destinations, potentially including telemetry about the collector itself (represented by the OpAMP `own_xxx` connection settings).
2. The bi-directional OpAMP control flow between the control plane implementing the server-side OpAMP part and the collector (or a supervisor controlling the collector) implementing OpAMP client-side.

Remark: **The OTel collector (OpAMP Client)** communicates with **the Control Plane (OpAMP Server)** to receive commands or configurations, while **the Control Plane** acts as the centralized control for managing multiple OTel collectors.

## Setup
First, in the `./opamp-go/internal/examples/supervisor/supervisor.yaml` file, change the `$OTEL_COLLECTOR` with the actual file path. For example, if you installed the collector in `/usr/local/bin/` then `$OTEL_COLLECTOR` should be /`usr/local/bin/otelcol`.

```yaml
server:
  endpoint: ws://127.0.0.1:4320/v1/opamp

agent:
  executable: $OTEL_COLLECTOR
```

After that, run the program: 
```shell
# run the backend server
cd 3_opamp-go/internal/examples/server
go run .
```

```shell
# launch the supervisor
cd 3_opamp-go/internal/examples/supervisor
go run .
```

The OpAmp server interface can be visited via [http://localhost:4321/](http://localhost:4321/), where user can see the collector list and managed by the supervisor.

## Overview

This is an example of the log of an agent indicated that this agent is healthy:
```json
{"level":"info","ts":1724384369.5610683,"caller":"service@v0.107.0/service.go:116","msg":"Setting up own telemetry..."}
{"level":"info","ts":1724384369.56119,"caller":"service@v0.107.0/telemetry.go:96","msg":"Serving metrics","address":":8888","metrics level":"Normal"}
{"level":"info","ts":1724384369.5624216,"caller":"service@v0.107.0/service.go:195","msg":"Starting otelcol...","Version":"0.107.0","NumCPU":12}
{"level":"info","ts":1724384369.5624487,"caller":"extensions/extensions.go:36","msg":"Starting extensions..."}
{"level":"info","ts":1724384369.5624774,"caller":"extensions/extensions.go:39","msg":"Extension is starting...","kind":"extension","name":"health_check"}
{"level":"info","ts":1724384369.5624988,"caller":"healthcheckextension@v0.107.0/healthcheckextension.go:32","msg":"Starting health_check extension","kind":"extension","name":"health_check","config":{"Endpoint":"localhost:13133","TLSSetting":null,"CORS":null,"Auth":null,"MaxRequestBodySize":0,"IncludeMetadata":false,"ResponseHeaders":null,"CompressionAlgorithms":null,"ReadTimeout":0,"ReadHeaderTimeout":0,"WriteTimeout":0,"IdleTimeout":0,"Path":"/","ResponseBody":null,"CheckCollectorPipeline":{"Enabled":false,"Interval":"5m","ExporterFailureThreshold":5}}}
{"level":"info","ts":1724384369.5628986,"caller":"extensions/extensions.go:56","msg":"Extension started.","kind":"extension","name":"health_check"}
{"level":"info","ts":1724384369.5635157,"caller":"prometheusreceiver@v0.107.0/metrics_receiver.go:307","msg":"Starting discovery manager","kind":"receiver","name":"prometheus/own_metrics","data_type":"metrics"}
{"level":"info","ts":1724384369.571969,"caller":"prometheusreceiver@v0.107.0/metrics_receiver.go:285","msg":"Scrape job added","kind":"receiver","name":"prometheus/own_metrics","data_type":"metrics","jobName":"otel-collector"}
{"level":"info","ts":1724384369.572008,"caller":"healthcheck/handler.go:132","msg":"Health Check state change","kind":"extension","name":"health_check","status":"ready"}
{"level":"info","ts":1724384369.5720198,"caller":"service@v0.107.0/service.go:221","msg":"Everything is ready. Begin running and processing data."}
{"level":"info","ts":1724384369.5720243,"caller":"localhostgate/featuregate.go:63","msg":"The default endpoints for all servers in components have changed to use localhost instead of 0.0.0.0. Disable the feature gate to temporarily revert to the previous default.","feature gate ID":"component.UseLocalHostAsDefaultHost"}
{"level":"info","ts":1724384369.572085,"caller":"prometheusreceiver@v0.107.0/metrics_receiver.go:376","msg":"Starting scrape manager","kind":"receiver","name":"prometheus/own_metrics","data_type":"metrics"}

```
The result when the program is set up properly:

![img](result.png)

### Some experiments

Additional configuration (in this example, service.version2 is added):

![img](result3.png)

![img](result4.png)

The reusult when **adding an invalid configuration**

![img](result2.png)

```log
Error: failed to get config: cannot unmarshal the configuration: decoding failed due to the following error(s):

'' has invalid keys: ahaha
2024/08/23 10:59:47 collector server run finished with error: failed to get config: cannot unmarshal the configuration: decoding failed due to the following error(s):

'' has invalid keys: ahaha
```


## References
[1] [Manage your OpenTelemetry collector deployment](https://opentelemetry.io/docs/collector/management/)

[2] [Using OpenTelemetry OpAMP to modify service telemetry on the go](https://opentelemetry.io/blog/2022/opamp/)