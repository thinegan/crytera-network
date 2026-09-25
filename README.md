# Network Security Architecture

This document provides a **high-level overview of the 2026 network security architecture**, focusing on **defense in depth**, **clear trust boundaries**, and **cost-efficient protection** using Cloudflare, OPNsense, CrowdSec, Suricata, Caddy, Envoy Gateway, Coraza, and Kubernetes.

---

## 🌐 Traffic Flow Overview

The public traffic path consists of multiple independent security and routing layers.

```text
Internet
   │
   ▼
┌──────────────────────────────┐
│        Cloudflare            │
│                              │
│ Edge filtering / Challenge   │
│ Worker + KV enforcement      │◄──────────────┐
└──────────────┬───────────────┘               │
               │                               │
               ▼                               │
┌──────────────────────────────┐               │
│          OPNsense            │               │
│             pf               │               │
│                              │               │
│ GeoIP / blocklists           │               │
│ CrowdSec Firewall Bouncer    │               │
└──────────────┬───────────────┘               │
               │                               │
               ▼                               │
┌──────────────────────────────┐               │
│            Caddy             │               │
│       TLS termination        │               │
│       Reverse proxy          │               │
└──────────────┬───────────────┘               │
               │                               │
               │ decrypted HTTP                │
               ▼                               │
┌──────────────────────────────┐               │
│          Suricata            │               │
│          IDS / IPS           │               │
│       Alert + Block          │               │
└──────────────┬───────────────┘               │
               │                               │
               ▼                               │
┌──────────────────────────────┐               │
│       Envoy Gateway          │               │
│                              │               │
│        Coraza WAF            │               │
│          + CRS               │               │
└──────────────┬───────────────┘               │
               │                               │
               ▼                               │
┌──────────────────────────────┐               │
│     Kubernetes Services      │               │
│            / Pods            │               │
└──────────────┬───────────────┘               │
               │
               │ access / security telemetry
               ▼
┌──────────────────────────────┐
│    Kubernetes CrowdSec       │
│                              │
│ Envoy Gateway logs           │
│                              │
│ Behavioral detection:        │
│  • excessive 403             │
│  • excessive 404             │
│  • excessive 401             │
└──────────────┬───────────────┘
               │
               │ 4-hour decision
               ▼
┌──────────────────────────────┐
│    Cloudflare KV Bouncer     │
│          Worker              │
└──────────────┬───────────────┘
               │
               │ IP enforcement
               └──────────────────────────────► Cloudflare
```

The architecture deliberately separates **edge filtering, network enforcement, reverse proxying, network IDS/IPS, gateway routing, HTTP WAF enforcement, and behavioral detection**.

---

# 🔐 Security Architecture

There are two independent CrowdSec deployments operating in different security domains.

### CrowdSec #1 — Network / Perimeter

This deployment focuses on network and infrastructure abuse.

```text
OPNsense / Caddy / SSH / firewall telemetry
                    │
                    ▼
                 CrowdSec
                    │
                    ▼
              CrowdSec LAPI
                    │
                    ▼
       OPNsense Firewall Bouncer
                    │
                    ▼
                   pf
                    │
                    ▼
                 BLOCK
```

CrowdSec #1 can consume security telemetry such as:

* OPNsense/pf events
* Caddy access logs
* SSH authentication events
* Firewall-related events

Its enforcement path terminates at the OPNsense firewall:

> **CrowdSec decision → Firewall Bouncer → pf**

This provides network/perimeter-level remediation.

---

## CrowdSec #2 — Kubernetes Application Security

The second deployment operates at the application layer.

```text
Envoy Gateway
      │
      │ access logs
      ▼
Kubernetes CrowdSec
      │
      │ behavioral detection
      │
      ├── excessive 403
      ├── excessive 404
      └── excessive 401
      │
      ▼
CrowdSec decision
      │
      │ 4 hours
      ▼
Cloudflare Worker / KV
      │
      ▼
Cloudflare
      │
      ▼
Future requests blocked at edge
```

This deployment does **not** sit inline with application traffic.

Instead, it observes traffic and application outcomes through Envoy Gateway telemetry.

CrowdSec then evaluates behavior over time and can create a decision for an abusive source.

The decision is subsequently enforced through the Cloudflare Worker/KV path.

---

# 🔄 Application Security Feedback Loop

The Kubernetes application-security path creates a feedback loop between application telemetry and edge enforcement.

```text
                    ┌─────────────────────────┐
                    │       Cloudflare        │
                    │                         │
                    │   Edge enforcement      │
                    └────────────▲────────────┘
                                 │
                            Worker / KV
                                 │
                                 │
Internet ──► OPNsense ──► Caddy ──► Suricata ──► Envoy
                 │                              │
                 │                              │
           CrowdSec #1                     Coraza WAF
                 │                              │
                 │                              ▼
                 │                         HTTP response
                 │                              │
                 │                         Envoy logs
                 │                              │
                 │                              ▼
                 │                         CrowdSec #2
                 │                              │
                 │                         4h decision
                 │                              │
                 └───────────────┐              │
                                 │              │
                                 ▼              ▼
                              Network       Edge
                              enforcement  enforcement
```

The important distinction is that CrowdSec #2 does not directly block the HTTP request passing through Kubernetes.

Its enforcement sequence is:

```text
Envoy
  ↓
Application telemetry
  ↓
CrowdSec detection
  ↓
CrowdSec decision
  ↓
Worker / KV bouncer
  ↓
Cloudflare
  ↓
Subsequent requests blocked at edge
```

---

# 🧩 Security Layers

| Layer                | Component              | Role                                      | Enforcement           |
| -------------------- | ---------------------- | ----------------------------------------- | --------------------- |
| **L7 Edge**          | Cloudflare             | Edge filtering and challenges             | **Block / Challenge** |
| **L3/L4**            | OPNsense pf            | Firewall, GeoIP and threat feeds          | **Block**             |
| **Behavioral L3/L4** | CrowdSec #1            | Network, firewall and SSH abuse detection | **pf ban**            |
| **Reverse Proxy**    | Caddy                  | TLS termination / trusted entry point     | **Proxy**             |
| **Network IDPS**     | Suricata               | Post-TLS network inspection               | **Alert + Block**     |
| **Gateway**          | Envoy Gateway          | HTTP routing / gateway                    | **Route**             |
| **WAF**              | Coraza + OWASP CRS     | Application-layer inspection              | **Block**             |
| **Behavioral L7**    | CrowdSec #2            | 401/403/404 abuse detection               | **Decision**          |
| **Edge Remediation** | Cloudflare Worker + KV | Enforce CrowdSec #2 decisions             | **Block**             |
| **Workloads**        | Kubernetes             | Applications                              | **Serve**             |

Each layer has a specific responsibility and enforcement boundary.

---

# 🛡️ Defense in Depth

The security architecture can be viewed as several independent controls:

```text
Internet
   │
   ▼
Cloudflare
   │
   │ Edge security
   ▼
OPNsense
   │
   │ Firewall enforcement
   ▼
Caddy
   │
   │ TLS termination
   ▼
Suricata
   │
   │ IDS / IPS
   ▼
Envoy Gateway
   │
   │ HTTP routing
   ▼
Coraza
   │
   │ OWASP CRS / WAF
   ▼
Kubernetes
   │
   ▼
Applications
```

Alongside this request path:

```text
Envoy telemetry
      │
      ▼
CrowdSec #2
      │
      │ behavioral detection
      ▼
4-hour decision
      │
      ▼
Cloudflare Worker / KV
      │
      ▼
Cloudflare enforcement
```

And independently at the perimeter:

```text
Network / infrastructure telemetry
              │
              ▼
          CrowdSec #1
              │
              ▼
       Firewall Bouncer
              │
              ▼
             pf
```

---

# 🧠 Detection vs Enforcement

A key architectural principle is keeping **detection** separate from **enforcement** where appropriate.

For example:

### Suricata

```text
Traffic
   ↓
Detection
   ↓
IPS rule
   ↓
Block
```

Suricata can directly enforce an IPS decision.

### Coraza

```text
HTTP request
   ↓
WAF inspection
   ↓
CRS rule
   ↓
HTTP 403
```

Coraza directly enforces an HTTP WAF decision.

### CrowdSec #2

```text
HTTP telemetry
   ↓
Behavioral detection
   ↓
Decision
   ↓
Worker / KV
   ↓
Cloudflare
   ↓
Block
```

CrowdSec #2 therefore acts primarily as a **behavioral detector and decision engine**, with enforcement delegated to the Cloudflare bouncer.

This distinction is important when troubleshooting security events.

---

# 🌐 Trust Boundaries

The architecture contains several important trust transitions.

### Cloudflare → OPNsense

Only expected public traffic should reach the origin network.

### OPNsense → Caddy

The firewall controls access to the reverse proxy.

### Caddy → Suricata

TLS has been terminated and the traffic becomes inspectable HTTP.

### Suricata → Envoy Gateway

Network IDS/IPS inspection has occurred before Kubernetes gateway processing.

### Envoy → Coraza

The HTTP request is evaluated against application WAF policy.

### Envoy → Kubernetes

Only explicitly configured Gateway API routes should expose services.

### Envoy → CrowdSec

Access telemetry becomes input for behavioral analysis.

### CrowdSec → Cloudflare

Application-level behavioral decisions become edge enforcement.

---

# 📊 Security Event Flow

A single malicious request may therefore generate telemetry at multiple layers.

For example:

```text
Client
  │
  ▼
Cloudflare
  │
  ▼
OPNsense
  │
  ▼
Caddy
  │
  ▼
Suricata
  │
  ├── Alert / Block
  │
  ▼
Envoy Gateway
  │
  ▼
Coraza
  │
  ├── WAF match / 403
  │
  ▼
Application
  │
  ▼
Envoy access log
  │
  ▼
CrowdSec #2
  │
  ├── Behavioral correlation
  │
  ▼
4-hour decision
  │
  ▼
Cloudflare Worker / KV
  │
  ▼
Future edge requests blocked
```

This allows security events to be correlated across the stack rather than relying on a single security product.

---

# 🧱 Component Responsibilities

## Cloudflare

Public edge security and enforcement.

Responsible for:

* Edge filtering
* Challenges
* Request filtering
* Worker/KV enforcement
* Blocking previously identified abusive IPs

---

## OPNsense

Network perimeter enforcement.

Responsible for:

* pf firewall policy
* NAT
* Network filtering
* GeoIP controls
* Threat feeds
* CrowdSec firewall enforcement

---

## Caddy

Controlled reverse-proxy entry point.

Responsible for:

* TLS termination
* Reverse proxying
* Trusted client identity propagation
* Structured access logging

---

## Suricata

Network IDS/IPS.

Responsible for:

* Network inspection
* Protocol analysis
* Signature detection
* Security alerts
* IPS blocking

---

## Envoy Gateway

Kubernetes gateway.

Responsible for:

* Gateway API
* HTTP listeners
* HTTP routing
* Upstream service selection
* Gateway policy

---

## Coraza

Application WAF.

Responsible for:

* HTTP request inspection
* OWASP CRS
* Application-layer exploit detection
* WAF blocking

---

## CrowdSec #1

Network/perimeter behavioral detection.

Responsible for:

* Network abuse detection
* SSH abuse detection
* Firewall-related behavioral detection
* Creating firewall enforcement decisions

---

## CrowdSec #2

Application-layer behavioral detection.

Responsible for:

* Observing Envoy access telemetry
* Correlating repeated HTTP behavior
* Detecting excessive 401/403/404 activity
* Creating temporary IP decisions

Enforcement is delegated to the Cloudflare Worker/KV bouncer.

---

## Kubernetes

Application platform.

Responsible for:

* Services
* Deployments
* Pods
* Application workloads
* Business logic

---

# 📁 2026 Documentation Structure

The architecture is documented by security domain:

```text
2026/
├── README.md
│
├── caddy/
│   └── README.md
│
├── cloudflare/
│   └── README.md
│
├── crowdsec/
│   └── README.md
│
├── datadog/
│   └── README.md
│
├── firewall/
│   └── README.md
│
├── ingress/
│   └── README.md
│
└── suricata/
    └── README.md
```

Each directory documents the responsibilities and configuration of its own component.

The top-level README describes how the components interact.

---

# 🎯 Design Principles

### Defense in depth

No single component is responsible for the entire security boundary.

### Clear trust boundaries

Each transition between security layers has an explicit trust model.

### Separation of responsibility

Each security component operates at the layer where it has the appropriate context.

### Detect broadly, enforce appropriately

Detection mechanisms can observe activity without every detector needing to be an inline blocker.

### Behavioral feedback

Repeated application abuse can be converted into an edge-level enforcement decision.

### Observable enforcement

Security decisions should produce telemetry that can be correlated and investigated.

### Cost efficiency

The architecture uses multiple open-source and platform-native controls while keeping Cloudflare at the public edge.

---

# ✅ Summary

The 2026 architecture is:

```text
                         INTERNET
                            │
                            ▼
                    ┌──────────────┐
                    │  Cloudflare  │
                    │ Edge Security│
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   OPNsense   │
                    │     pf       │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Caddy     │
                    │ TLS / Proxy  │
                    └──────┬───────┘
                           │
                     decrypted HTTP
                           │
                           ▼
                    ┌──────────────┐
                    │  Suricata    │
                    │   IDS / IPS  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Envoy     │
                    │   Gateway    │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Coraza    │
                    │  OWASP CRS   │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Kubernetes   │
                    │ Services/Pods│
                    └──────────────┘
```

With two independent CrowdSec feedback paths:

```text
Network / Firewall telemetry
             │
             ▼
        CrowdSec #1
             │
             ▼
    OPNsense Firewall
         Bouncer
             │
             ▼
             pf


Envoy access telemetry
             │
             ▼
        CrowdSec #2
             │
             ▼
       4-hour decision
             │
             ▼
     Worker / KV Bouncer
             │
             ▼
         Cloudflare
             │
             ▼
      Edge enforcement
```

The resulting architecture has clearly separated responsibilities:

> **Cloudflare protects the public edge.**

> **OPNsense enforces network policy.**

> **Caddy terminates TLS and proxies traffic.**

> **Suricata provides network IDS/IPS.**

> **Envoy Gateway provides Kubernetes gateway and routing.**

> **Coraza provides HTTP WAF enforcement.**

> **CrowdSec #1 provides network/perimeter behavioral detection.**

> **CrowdSec #2 provides application behavioral detection and automated edge remediation.**

> **Kubernetes serves the applications.**
