---
title: "Architecture for Multi-Tenant Dataspace Environments"
date: 2026-02-06
description: >
  How EDC-V and CFM provide the operational model for running multi-tenant dataspace environments: shared runtime cells,
  Virtual Participant Agents (VPAs), and the separation of provisioning from trust decisions.
---

Operating dataspaces “as a service” changes the basic unit you manage. Instead of treating each participant organization as a separate connector deployment, **service virtualization** turns participant contexts into lightweight, repeatable units that the platform can provision and operate at scale.

This post is Part 1 of a short series:

- Part 1: **Architecture** (this post)
- Part 2: [Multi-Tenant EDC from a participant perspective](/blog/20260206-multi-tenant-edc-participant-perspective/)

If you want the “entry point” within the documentation tree, start at [Multi-Tenant Deployments](/documentation/for-adopters/multi-tenant-deployments/).

## Summary

At a high level, operating **EDC-V** at scale means running a **management plane** (**CFM**) plus a **shared runtime** that hosts isolated **VPA** contexts.

- **As an operator you run**: a management plane (CFM) plus shared runtime cells hosting Virtual Participant Agents (VPAs).
- **Your customers use**: a portal + apps that manage identities, publish catalogs, negotiate contracts, and configure data planes inside their participant context.
- **Data moves**: peer-to-peer between participants over open protocols: DCP (identity proofs), DSP (catalog + negotiation), DPS (control↔data plane signaling).

The key operational consequence is the architectural separation:

> **CFM provisions participant contexts, but it is not in the trust-decision path.**  
> CFM downtime blocks onboarding and provisioning, not live negotiation and data transfers.

## Why this model is worth implementing

Multi-tenant operation becomes attractive when you want to make EDC-based data sharing repeatable and operable across many organizations.

- **Operations cost**: operate shared cells and automate provisioning instead of running per-tenant stacks.
- **Onboarding**: shift from hand-crafted deployments to workflows and templates.
- **Scalability**: add capacity by scaling cells, not by multiplying bespoke deployments.
- **Sovereignty**: trust decisions stay peer-to-peer; data planes can run close to the data.
- **Interoperability**: DCP/DSP/DPS keep you compatible with external and self-hosted participants.

## The three-plane model

Think in three planes: a cloud-native foundation, a management plane that provisions participant contexts, and a runtime plane where dataspace protocols execute.

![EDC-V platform architecture — operations view](multi-tenant-edc-1.png)

| Layer | What it is | What it’s responsible for |
| --- | --- | --- |
| Cloud-native infrastructure | Kubernetes, storage, secrets, DNS, IAM, observability | Reliability primitives, persistence, identity, operational tooling |
| **CFM (Connector Fabric Manager)** | Management plane | Provisioning workflows, tenant metadata, VPA lifecycle automation |
| **VPAs (Virtual Participant Agents)** | Runtime plane | Protocol endpoints, policy decisions, data flow execution |

## Roles and operational boundaries

Roles are **logical roles**. Some map to human-facing UIs; others are machine identities used for automation and tightly controlled privileged access.

| Role | Typical client | Responsibilities |
| --- | --- | --- |
| Participant | Customer Portal (via a UI backend) | Manage participant-scoped resources such as catalogs/assets, policies, contracts, and data flow |
| Provisioner | CFM / provisioning automation | Onboard and manage participant contexts (create/configure tenants); must not manipulate participant-owned business data |
| Operator | Operations UI + platform tooling | Deploy and operate platform infrastructure; monitor, scale, and troubleshoot cells and shared services |
| Admin (emergency) | Restricted automation / privileged operator access | Full access for initial setup and emergency recovery; not intended for day-to-day use |

## The infrastructure foundation

Run EDC-V on standard cloud-native infrastructure and keep the foundation intentionally “boring”. Your goal at this layer is operational certainty: predictable failure modes, repeatable deployment patterns, and runbooks your teams already know how to execute.

Treat identity, secrets, persistence, networking, and observability as first-class dependencies. Prefer managed offerings and proven primitives over bespoke infrastructure—especially for IAM/IDP integration, secret storage, database backups, and telemetry pipelines.

| Component | Purpose |
|---|---|
| Kubernetes | Container orchestration and scaling |
| PostgreSQL | State persistence across components |
| Vault / STS | Secrets, key material, and token-related infrastructure |
| DNS | Request routing and DID resolution |
| Observability stack | Metrics, logging, and tracing |
| IAM / IDP | Authentication for operators and participants |

## The Connector Fabric Manager (CFM)

CFM is the **management plane**. It provisions participant contexts and automates the lifecycle of VPAs. You can think of it as an orchestration layer for service virtualization: it creates runtime, but it is not the runtime.

For the full architectural model and extension points, see the upstream documentation: [CFM system architecture](https://github.com/Metaform/connector-fabric-manager/blob/main/docs/developer/architecture/system.architecture.md).

CFM comprises three subsystems:

| Subsystem | Role |
|---|---|
| Tenant Manager (TM) | Persists tenancy and virtualization metadata; exposes a REST API; initiates deployments |
| Provision Manager (PM) | Executes stateful orchestrations (workflows) for onboarding and VPA lifecycle |
| Activity Agents | Asynchronously process orchestration steps in isolated security contexts |

The Tenant Manager is the metadata control point; the Provision Manager is the execution engine. Communication between them happens through NATS JetStream, which provides reliable, decoupled messaging that makes long-running orchestrations resilient to restarts.

Activity Agents are where you integrate with your cloud platform. Typical responsibilities include:

- Deploy runtime components to Kubernetes
- Configure Vault namespaces
- Set up DNS entries

### Architectural insight: provisioning is not runtime trust

The most important property to internalize is that **CFM is not in the trust-decision path**.

- Live data sharing continues during CFM maintenance or outages.
- Operationally, you can run separate SLOs for management plane vs runtime plane.
- Incident response becomes easier: onboarding can degrade without turning into a platform-wide outage.

## Virtual Participant Agents (VPAs)

VPAs are the unit of runtime isolation and administrative control in a service-virtualized EDC-V deployment. A VPA represents the runtime context for a participant profile, but it is **not** a dedicated per-tenant stack: shared services create an isolation context from VPA metadata (a configuration-based isolation model).

EDC-V commonly provisions three VPA types:

- **Control Plane VPA**: evaluates trust and policy decisions, publishes catalogs, negotiates contracts. See the [EDC Control Plane documentation](/documentation/for-adopters/control-plane/).
- **Credential Service VPA**: stores verifiable credentials and produces proofs. In the Eclipse ecosystem, the canonical wallet/credential implementation is Identity Hub; see [Identity Hub](/documentation/for-adopters/identity-hub/).
- **Data Plane VPA**: executes data flows once authorized; optimized for throughput and proximity to data. See [Data Plane](/documentation/for-adopters/data-plane/).

> **Context Isolation**: While VPAs share infrastructure, they are logically isolated. One participant context cannot see or access another participant's data, credentials, or configuration.

In production, expect **multiple instances of each type** for capacity and separation (for example multiple data planes per participant for protocol or environment separation).

## The mental model shift

Traditional connector deployments follow a simple equation: one connector equals one process. You deploy infrastructure per tenant, scale by adding containers, and manage operations on a per-tenant basis.

CFM-managed deployments invert that model: one runtime serves many participant contexts.

| Traditional deployment | CFM-managed deployment |
|---|---|
| One connector = one process | One runtime serves many VPAs |
| Deploy infrastructure per tenant | Provision VPA metadata |
| Scale by adding containers | Scale by adding cells |
| Manage operations per-tenant | Manage operations centrally |

This shift makes scaling sub-linear rather than linear with tenant count. You manage fewer cells with centralized tooling instead of hundreds of per-tenant deployments.

## What’s next

Continue with Part 2: [Multi-Tenant EDC from a participant perspective](/blog/20260206-multi-tenant-edc-participant-perspective/).
