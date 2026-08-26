---
title: "Granite Lab Industries: Global Kubernetes Metadata Standards"
url: "https://docs.granitelab.example.com/global-metadata-standards.html"
---

# Granite Lab Industries: Global Kubernetes Metadata Standards

## Overview

To support cost allocation, observability, and automated governance, all Kubernetes and OpenShift objects deployed to Granite Lab clusters MUST follow these metadata standards.

## Mandatory labels

Every Kubernetes object (Deployments, Pods, Services, NetworkPolicies, and so on) MUST include these labels in `metadata.labels`:

- `granitelab.com/environment`: One of `dev`, `qa`, `staging`, or `prod`.
- `granitelab.com/business-unit`: The owning department (for example `retail`, `finance`, or `platform`).
- `granitelab.com/app-name`: The logical application name.

## Mandatory annotations

All Deployments and StatefulSets MUST include:

- `granitelab.com/support-contact`: A valid email address for the responsible team (for example `platform-team@granitelab.example.com`).
