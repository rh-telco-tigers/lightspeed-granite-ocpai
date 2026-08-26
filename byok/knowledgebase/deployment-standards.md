---
title: "Granite Lab Industries: Container Deployment Standards"
url: "https://docs.granitelab.example.com/deployment-standards.html"
---

# Granite Lab Industries: Container Deployment Standards

## Overview

All containerized applications running on OpenShift must follow Granite Lab security and resource management guidelines.

## Replica requirements

- All Deployments MUST set `replicas` to at least `2` for high availability.

## Security context constraints (SCC)

Pods and containers must run with least privilege. The following `securityContext` settings are mandatory for every container:

- `allowPrivilegeEscalation: false`
- `runAsNonRoot: true`
- `capabilities.drop: ["ALL"]`
- `seccompProfile.type: RuntimeDefault`

## Resource management

Containers MUST NOT be deployed without resource requests and limits. When a developer does not specify them, use these defaults:

- **Requests:** `cpu: 100m`, `memory: 128Mi`
- **Limits:** `cpu: 500m`, `memory: 512Mi`

## Health probes

All containers MUST define both `livenessProbe` and `readinessProbe`. If the protocol is unknown, default to a TCP socket check on the container's primary port.

## Metadata

Apply Granite Lab global metadata standards (environment, business-unit, app-name) to both Deployment metadata and the Pod template metadata.
