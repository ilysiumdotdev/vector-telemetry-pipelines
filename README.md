# Vector Telemetry Pipelines

Canonical Vector agent and aggregator pipelines to collect, normalize, and ship cross-domain telemetry to SIEM and archival stores.

![image](./docs/vector-open-graph.png)

## Table of Contents

- [Planned Features](#planned-features)
- [Overview](#overview)
- [Solution Architecture](#solution-architecture)
  - [Agent](#agent)
  - [Aggregator](#aggregator)
  - [Telemetry Stream & Routing](#telemetry-stream--routing)
    - [Sources](#sources)
    - [Transforms](#transforms)
    - [Sinks](#sinks)
- [Repository Structure](#repository-structure)
- [Tests](#tests)

## Planned Features

Work in progress and upcoming changes for this repository:

- [ ] Update all remap transforms to normalize fields to **[ECS](https://www.elastic.co/docs/reference/ecs) (Elastic Common Schema)** to support easier correlation of events across domains
- [ ] Add **Kafka** sources/sink to the core log streams to support fan-out patterns and advanced analysis of security telemetry

## Overview

This repository serves as the canonical deployment manifest for Vector agent/aggregator configurations for collecting, normalizing, transforming, and shipping telemetry data to a various destinations, including SIEM and archival storage.

While the primary value of this solution comes from its ability to collect and aggregate security-relevant telemetry across domains, such as cloud environments, on-premise platforms (Kubernetes), identity providers, and network devices, it also serves as a collector for operational telemetry data for troubleshooting, such as standard metrics and traces.

## Solution Architecture

This configuration has Vector deployed with both **agents** and **aggregators**:

### Agent

Agent configurations are located in `/config/agent`. This configuration has Vector act in a more traditional log collection agent context, where it is used to read logs from file and stdout locations on the host. Because the Vector agent is a **DaemonSet** on Kubernetes, we can ensure an agent instance exists on every node in the cluster at all times.

### Aggregator

Aggregator configurations are located in `/config/aggregator`. The aggregator is used to collect event from network-based locations, such as sockets for syslogs from physical network devices.

Aggregators handle the more complex processing in the pipeline, such as remapping fields, filtering events, and routing to destinations.

The Vector aggregator is deployed as a `Deployment` on Kubernetes, meaning the number of instances can be scaled automatically with the volume of incoming events.

### Telemetry Stream & Routing

#### Sources

| Name                  | Type | Description                                                                                                      | Value                                                                                                       |
| --------------------- | ---- | ---------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Hubble Logs           | Logs | Provides insights into all network traffic & network policy evaluations inside Kubernetes                        | Troubleshooting applicaion connectivity, identifying lateral movement between pods                          |
| Kubernetes Logs       | Logs | Application logs for every pod running on Kubernetes, collected via stdout                                       | Operational application logs, correlating security telemetry with application-level events                  |
| Kubernetes Audit Logs | Logs | Audit logs from the Kubernetes API server for resource operations (e.g., create/update/delete) and privilege use | Audit sensitive cluster operations (create deployment/cronjob) and command execution (e.g., `kubectl exec`) |

#### Transforms

1. Tag events with collection source and additional metadata (`@dataset`, `@namespace`) for upstream routing
2. Attempt to parse each event as JSON, since Vector rolls up collected logs under `.message`. If JSON parsing succeeds, the log fields are merged into the top level object to remove the `.message` key. Otherwise, the `.message` key and its plain string value are kept intact for querying.
3. Route events to their respective transform using `@source` and `@dataset` metadata with a `route` transform.
4. Use a targeted `remap` transform to streamline the schema of specific events (rename, move, drop, or normalize fields).

#### Sinks

- **Elastic** — SIEM and store for security/operational logs and metrics
- **S3** — Archival storage for logs with lifecycle rules for cost-effective long-term storage on S3 Glacier
- **Kafka** — Supports fan-out patterns to deliver events to multiple downstream consumers independently

## Repository Structure

```
vector-deployment/
├── config/
│   ├── agent/          # Agent configurations
│   └── aggregator/     # Aggregator configurations
├── kubernetes/
│   ├── base/           # Flux HelmRelease manifests, Kustomize config maps, TLS, and secrets
│   └── production/     # Production overlay (references base)
├── tests/
│   ├── agent/          # Unit tests for agent transforms
│   └── aggregator/     # Unit tests for aggregator transforms
├── docs/               # Documentation assets
└── .github/workflows/  # CI validation and test runs on pull request
```

| Path                 | Description                                                                                           |
| -------------------- | ----------------------------------------------------------------------------------------------------- |
| `config/agent/`      | Vector agent configurations — handles initial host-based log collection and handoff                   |
| `config/aggregator/` | Vector aggregator configuration — parses, routes, remaps, and ships events to upstream destinations   |
| `kubernetes/`        | Kubernetes deployment manifests (Flux + Kustomize) that mount `config/` into running Vector instances |
| `tests/`             | Vector unit test definitions paired with their respective `config/` directory                         |

## Tests

Unit tests are used to test Vector transforms with mock events to ensure they produce the desired event schema (e.g., fields are named properly, computed values are correct, etc.)

Tests live in `tests/agent/` and `tests/aggregator/`, mirroring the layout of `config/agent/` and `config/aggregator/`. Each test file targets a single transform and is run alongside its corresponding config directory.

To run the tests locally, use the following commands:

```bash
vector validate --config-dir config/agent/ --no-environment
vector validate --config-dir config/aggregator/ --no-environment
vector test --config-dir config/agent --config-dir tests/agent
vector test --config-dir config/aggregator --config-dir tests/aggregator
```
