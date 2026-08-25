---
title: "Granite Lab Industries: Enterprise Network Connectivity and Database Access"
url: "https://docs.granitelab.example.com/network/enterprise-connectivity"
---

# Granite Lab Industries: Enterprise Network Connectivity and Database Access

## Overview

Granite Lab Industries operates a segmented enterprise network. OpenShift clusters run on a dedicated application VLAN. External databases and other sensitive data services reside on isolated VLANs protected by a default-deny firewall policy. Communication between VLANs is permitted only through explicit, documented firewall rules.

This document describes the lab network topology, allowed paths, and the troubleshooting steps application teams must follow when OpenShift-hosted workloads cannot reach external databases.

## VLAN reference

| VLAN | Name | Subnet | Gateway | Purpose |
|------|------|--------|---------|---------|
| VLAN10 | Management | `172.16.10.0/24` | `172.16.10.1` | Jump hosts, Ansible, cluster administration, backup orchestration |
| VLAN15 | Database | `172.16.15.0/24` | `172.16.15.1` | PostgreSQL, MySQL, Oracle, and other persistent data services |
| VLAN20 | Corporate | `172.16.20.0/24` | `172.16.20.1` | Developer workstations, SSO clients, internal web apps |
| VLAN25 | OpenShift | `172.16.25.0/24` | `172.16.25.1` | OpenShift worker and control-plane nodes, ingress, egress NAT |
| VLAN30 | DMZ | `172.16.30.0/24` | `172.16.30.1` | Public-facing load balancers, API gateways, partner integrations |
| VLAN40 | Backup | `172.16.40.0/24` | `172.16.40.1` | Backup appliances, tape gateways, off-cluster snapshot targets |
| VLAN50 | Monitoring | `172.16.50.0/24` | `172.16.50.1` | Prometheus, Grafana, log collectors, SNMP traps |

All inter-VLAN traffic passes through the Granite Lab core firewall pair (`fw-core-01` / `fw-core-02`). There is no direct L2 routing between application and database VLANs.

## OpenShift platform addressing (VLAN25)

The lab OpenShift cluster uses the following infrastructure addresses on VLAN25:

| Host / Service | IP address | Notes |
|----------------|------------|-------|
| OpenShift API / Ingress VIP | `172.16.25.10` | Shared VIP for API and ingress |
| Worker nodes | `172.16.25.21` – `172.16.25.29` | Application egress source addresses |
| Egress NAT pool | `172.16.25.100` – `172.16.25.110` | SNAT pool for pod traffic leaving the cluster |
| Internal registry mirror | `172.16.25.50` | Pull-through cache; not on database VLAN |

Pod egress to external networks is source-NATed to the VLAN25 egress pool (`172.16.25.100`–`172.16.25.110`). Firewall rules for database access MUST reference this NAT pool, not individual pod CIDRs.

## Database subnet (VLAN15)

VLAN15 is a restricted data tier. Inbound connections from other VLANs are denied unless a named firewall rule exists. Outbound connections from VLAN15 to application VLANs are denied by default.

| Database server | Hostname | IP address | Port | Engine | Allowed consumers |
|-----------------|----------|------------|------|--------|-------------------|
| `db-postgres-prod-01` | `db-postgres-prod-01.granitelab.example.com` | `172.16.15.10` | `5432` | PostgreSQL 16 | OpenShift production namespaces |
| `db-postgres-dev-01` | `db-postgres-dev-01.granitelab.example.com` | `172.16.15.11` | `5432` | PostgreSQL 16 | OpenShift non-production namespaces |
| `db-mysql-legacy-01` | `db-mysql-legacy-01.granitelab.example.com` | `172.16.15.20` | `3306` | MySQL 8.0 | Approved legacy apps only |
| `db-oracle-fin-01` | `db-oracle-fin-01.granitelab.example.com` | `172.16.15.30` | `1521` | Oracle 19c | Finance workloads (change-controlled) |
| `db-admin-bastion` | `db-admin-bastion.granitelab.example.com` | `172.16.15.254` | `22` | SSH jump | VLAN10 management hosts only |

Database servers do not initiate connections to VLAN25. All access is client-initiated from OpenShift application pods.

## Global firewall policy

Unless overridden by an explicit allow rule below:

1. **Default deny** between all VLANs at the core firewall.
2. **No hairpin routing** — VLAN25 cannot reach VLAN15 (or any other VLAN) without a named rule.
3. **Return traffic** is permitted only for established sessions matching an allowed inbound rule.
4. **ICMP** (ping) from VLAN25 to VLAN15 is denied unless a rule explicitly allows diagnostics for a change window.
5. **DNS** — OpenShift nodes and pods resolve internal names through `dns-core-01.granitelab.example.com` (`172.16.10.53`). Database hostnames in this document are internal zones only.

## Explicit firewall rules (inter-VLAN)

### OpenShift (VLAN25) → Database (VLAN15)

| Rule ID | Source | Destination | Ports | Protocol | Purpose |
|---------|--------|-------------|-------|----------|---------|
| FW-25-15-001 | `172.16.25.100`–`172.16.25.110` (egress NAT) | `172.16.15.10` | `5432` | TCP | Production PostgreSQL |
| FW-25-15-002 | `172.16.25.100`–`172.16.25.110` (egress NAT) | `172.16.15.11` | `5432` | TCP | Development PostgreSQL |
| FW-25-15-003 | `172.16.25.100`–`172.16.25.110` (egress NAT) | `172.16.15.20` | `3306` | TCP | Legacy MySQL (requires CAB approval) |
| FW-25-15-004 | `172.16.25.105`–`172.16.25.106` (restricted NAT) | `172.16.15.30` | `1521` | TCP | Finance Oracle — limited SNAT range only |

> **Important:** Rules FW-25-15-001 through FW-25-15-003 use the full egress NAT pool. Rule FW-25-15-004 uses a **subset** of the pool (`172.16.25.105`–`172.16.25.106`). Pods whose egress is SNATed outside that range cannot reach the Oracle finance database even if the application configuration is correct.

### Other enterprise rules (reference)

| Rule ID | Source | Destination | Ports | Protocol | Purpose |
|---------|--------|-------------|-------|----------|---------|
| FW-10-ALL-001 | `172.16.10.0/24` | All VLANs | `22`, `443` | TCP | Management administration |
| FW-20-30-001 | `172.16.20.0/24` | `172.16.30.0/24` | `443` | TCP | Corporate users to DMZ web tier |
| FW-30-25-001 | `172.16.30.0/24` | `172.16.25.10` | `443` | TCP | DMZ to OpenShift ingress |
| FW-25-50-001 | `172.16.25.0/24` | `172.16.50.0/24` | `9090`, `3100` | TCP | OpenShift to monitoring (metrics, Loki) |
| FW-15-40-001 | `172.16.15.0/24` | `172.16.40.0/24` | `10000` | TCP | Database backup to backup VLAN |
| FW-ANY-15-DENY | Any | `172.16.15.0/24` | Any | Any | Implicit deny — logged and dropped |

VLAN15 has **no** allow rule for VLAN20 (corporate), VLAN30 (DMZ), or direct pod CIDRs. Only the documented SNAT ranges on VLAN25 may reach database listeners.

## Application connectivity requirements

When an OpenShift application connects to an external database on VLAN15, all of the following must be true:

1. **Correct hostname** — Use the FQDN from the database inventory table (for example `db-postgres-prod-01.granitelab.example.com`), not a stale IP or external alias.
2. **Correct port** — Match the listener port in the inventory table.
3. **Firewall rule exists** — A rule in the FW-25-15-* series must cover the target database IP and port.
4. **Egress NAT alignment** — The pod's egress must be SNATed to an address included in the rule's source range. Finance Oracle (FW-25-15-004) requires the restricted pool `172.16.25.105`–`172.16.25.106`.
5. **No conflicting NetworkPolicy** — In-cluster `NetworkPolicy` objects must allow egress to the database IP and port. Cluster network policy and enterprise firewall policy are independent; both must permit the flow.
6. **Database-side access control** — The database `pg_hba.conf`, MySQL `bind-address`, or Oracle `sqlnet.ora` must allow connections from the VLAN25 SNAT range.

## Troubleshooting connectivity from OpenShift to VLAN15 databases

Use this checklist when an application reports timeouts or "connection refused" to an external database.

### Step 1: Verify DNS resolution from a debug pod

```sh
oc run netcheck --rm -it --image=registry.access.redhat.com/ubi9/ubi-minimal -- \
  bash -c "nslookup db-postgres-prod-01.granitelab.example.com"
```

Expected result: `172.16.15.10`. If resolution fails, fix DNS before checking firewalls.

### Step 2: Test TCP connectivity from a debug pod

```sh
oc run netcheck --rm -it --image=registry.access.redhat.com/ubi9/ubi-minimal -- \
  bash -c "timeout 5 bash -c '</dev/tcp/172.16.15.10/5432' && echo OPEN || echo CLOSED"
```

- **OPEN** — Path is reachable from the pod; investigate application credentials, TLS, or database ACLs.
- **CLOSED / timeout** — Continue with steps 3–5.

### Step 3: Confirm the firewall rule covers the target

Look up the database IP in the inventory table and verify a rule in the FW-25-15-* series allows VLAN25 egress NAT to that IP and port. If no rule exists, open a firewall change request referencing the rule ID format (`FW-25-15-00x`).

### Step 4: Confirm egress SNAT range

Identify which NAT address the pod uses (network team can correlate flow logs on `fw-core-01`). If the application targets Oracle finance (`172.16.15.30`) but egress uses `172.16.25.102`, the connection will be dropped — only `172.16.25.105`–`172.16.25.106` are permitted by FW-25-15-004.

### Step 5: Review in-cluster NetworkPolicy

Ensure the application namespace allows egress to `172.16.15.0/24` on the required port. A `default-deny-all` policy without a matching egress allow rule blocks traffic before it reaches the enterprise firewall.

### Step 6: Review database host ACLs

On the database server, confirm the VLAN25 SNAT range is permitted. PostgreSQL example:

```text
host  all  all  172.16.25.100/28  scram-sha-256
```

## Common failure scenarios

| Symptom | Likely cause | Resolution |
|---------|--------------|------------|
| Timeout to `172.16.15.10:5432` | Missing or wrong FW-25-15-001 rule; egress not using NAT pool | Verify firewall rule and SNAT configuration |
| Timeout to `172.16.15.30:1521` | Egress SNAT outside `172.16.25.105`–`172.16.25.106` | Request finance network policy exception or use approved egress class |
| `Connection refused` | Database service down or listening on wrong interface | Check database service on VLAN15; verify bind address |
| DNS resolves but TCP fails | Firewall deny or NetworkPolicy block | Run troubleshooting steps 3 and 5 |
| Works from VLAN10 jump host, fails from OpenShift | Jump host has FW-10-ALL-001; OpenShift requires FW-25-15-* | Do not use jump-host success as proof of OpenShift access |
| Intermittent failures | NAT pool exhaustion or asymmetric routing | Engage network team; verify connection tracking on `fw-core-01` |

## Change management

New database dependencies from OpenShift require:

1. A firewall change request citing source NAT range, destination IP, port, and business justification.
2. Database team approval for VLAN25 SNAT range in host-level ACLs.
3. Platform team review of namespace `NetworkPolicy` egress rules.
4. Assignment of a permanent rule ID in the `FW-25-15-*` series before production deployment.

Emergency break-glass access to VLAN15 is available only from `db-admin-bastion` (`172.16.15.254`) via VLAN10 management hosts. Break-glass does not grant OpenShift pods direct access.

## Support contacts

| Team | Contact | Scope |
|------|---------|-------|
| Network / Firewall | `network-team@granitelab.example.com` | FW-25-15-* rules, SNAT, routing |
| Database | `dba-team@granitelab.example.com` | Listener config, credentials, host ACLs |
| OpenShift Platform | `platform-team@granitelab.example.com` | Egress NAT, NetworkPolicy, debug pods |
