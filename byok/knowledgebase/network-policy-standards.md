---
title: "Granite Lab Industries: Network Policy Standards"
url: "https://docs.granitelab.example.com/network-policy-standards.html"
---

# Granite Lab Industries: Network Policy Standards

## Overview

Granite Lab operates a zero-trust network model within OpenShift clusters. Namespaces are isolated by default; developers must explicitly define allowed traffic.

## Default deny rule

Every namespace MUST contain a NetworkPolicy named `default-deny-all` that blocks all ingress and egress traffic by default:

- `podSelector: {}` (matches all pods)
- `policyTypes: ["Ingress", "Egress"]`
- No ingress or egress rules in this base policy

## Allowed egress: DNS

Because of the default deny rule, pods cannot resolve DNS without an explicit policy. Every namespace MUST include a NetworkPolicy named `allow-dns-egress` that allows UDP and TCP traffic on port 53 to the `openshift-dns` namespace.

## Ingress: OpenShift Ingress Controller

Applications that need external access via an OpenShift Route MUST create a NetworkPolicy named `allow-from-openshift-ingress` that allows ingress from pods matching the namespace selector `network.openshift.io/policy-group: ingress`.

## Intra-namespace communication

Microservices in the same namespace MUST use a NetworkPolicy named `allow-same-namespace`. The policy should allow ingress where `podSelector` matches the target application labels and `from.podSelector` matches the source application labels.
