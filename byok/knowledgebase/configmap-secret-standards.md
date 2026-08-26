---
title: "Granite Lab Industries: ConfigMap and Secret Standards"
url: "https://docs.granitelab.example.com/openshift/configmap-secret-standards.html"
---

# Granite Lab Industries: ConfigMap and Secret Standards

## Overview

Granite Lab requires separation of configuration from application code to maintain 12-Factor App compliance.

## ConfigMap standards

- **Usage:** ConfigMaps MUST hold all non-sensitive configuration (environment variables, JSON or XML config files, and similar data).
- **Immutability:** ConfigMaps SHOULD set `immutable: true` when values are not expected to change at runtime. This forces a safe rollout for updates and reduces API server load.
- **Naming:** Follow the pattern `<app-name>-config`.

## Secret standards

- **Usage:** Secrets MUST hold passwords, API keys, TLS certificates, and tokens.
- **Declarative format:** Use `stringData` instead of `data` in YAML templates so values are readable before the cluster base64-encodes them.
- **Type:** Default to `type: Opaque` unless a specific structural type (such as `kubernetes.io/tls`) is required.
- **Immutability:** Secrets SHOULD use `immutable: true` when values rotate only with a planned pod restart.
