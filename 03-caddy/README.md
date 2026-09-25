# Caddy Reverse Proxy

## Overview

Caddy is the **TLS termination and reverse proxy layer** for the 2026 network security architecture.

It sits behind Cloudflare and the OPNsense firewall and provides the controlled entry point from the network perimeter into the Kubernetes environment.

Caddy is intentionally kept focused. Its primary responsibilities are to:

* Terminate TLS
* Reverse proxy HTTP traffic
* Preserve trusted client identity
* Forward trusted request headers
* Produce structured access logs
* Provide a controlled upstream connection to the Kubernetes Gateway

Caddy is **not the application firewall** and does not attempt to duplicate the security responsibilities of Suricata, Coraza, CrowdSec, or other downstream controls.

---

## Architecture Position

```text
Internet
   |
   v
Cloudflare
   |
   | HTTPS
   v
OPNsense Firewall
   |
   | HTTPS
   v
+-----------------------------------+
|          CADDY REVERSE PROXY      |
|                                   |
|          192.168.1.3              |
|                                   |
|  TLS termination                  |
|  Reverse proxy                    |
|  Client identity propagation     |
|  Structured access logging        |
|  Trusted header forwarding        |
+------------------+----------------+
                   |
                   | HTTP
                   v
              Suricata IDS/IPS
                   |
                   | HTTP
                   v
             Envoy Gateway
                   |
                   v
                Coraza
                   |
                   v
              Kubernetes
```

Caddy therefore forms the controlled transition between the **network perimeter** and the **network/application security layers leading to the Kubernetes Gateway**.

Caddy is not the final security enforcement point.

---

## Responsibilities

### TLS termination

Caddy terminates the HTTPS connection received from the public edge.

This provides a controlled point at which encrypted traffic becomes HTTP before being forwarded into the internal security and gateway path.

The resulting HTTP traffic is inspected by Suricata before reaching Envoy Gateway.

```text
Cloudflare
    |
    | HTTPS
    v
Caddy
    |
    | TLS termination
    v
HTTP
    |
    v
Suricata IDS/IPS
    |
    v
Envoy Gateway
```

### Reverse proxy

Caddy forwards requests to the internal Kubernetes Gateway path.

Current upstream:

```text
http://192.168.100.152:80
```

Caddy should only proxy traffic to the intended gateway endpoint.

Applications are not exposed directly through Caddy.

Caddy must not bypass the configured Suricata/Envoy security path.

### Client identity preservation

Requests originate from Cloudflare, so Caddy must preserve the original client identity supplied through the trusted Cloudflare proxy path.

Relevant headers include:

* `CF-Connecting-IP`
* `X-Forwarded-For`
* `CF-IPCountry`

The original client IP is important for downstream request logging, security analysis, Suricata telemetry, Coraza processing, and CrowdSec correlation.

Caddy must therefore distinguish between:

```text
Trusted proxy headers
```

and:

```text
Untrusted client-supplied headers
```

Client identity should only be accepted from the trusted proxy chain.

### Access logging

Caddy produces structured access logs for operational and security visibility.

Logs should provide, where available:

* Client IP
* Host
* HTTP method
* URI
* Query string
* HTTP status
* Request duration
* User agent
* Referer
* Response size

These logs provide an authoritative record of requests observed at the reverse-proxy boundary.

Caddy logs also provide useful telemetry for correlation with:

* Cloudflare
* OPNsense
* Suricata
* Envoy Gateway
* Coraza
* CrowdSec
* Datadog

---

## TLS

Cloudflare connects to Caddy using HTTPS.

The intended connection is:

```text
Cloudflare
    |
    | HTTPS
    v
Caddy
    |
    | TLS termination
    v
HTTP
    |
    v
Suricata IDS/IPS
    |
    v
Envoy Gateway
```

Cloudflare SSL/TLS mode:

```text
Full (Strict)
```

Caddy therefore presents a valid certificate for the requested hostname.

Certificates may be managed through:

* Let's Encrypt
* Another trusted public CA
* An appropriately managed internal CA where applicable

TLS certificates and their renewal are part of the Caddy configuration and should be managed independently from application workloads.

Certificate issuance and renewal failures should be observable through operational monitoring.

---

## Trusted Proxy Model

Caddy is positioned behind Cloudflare.

Therefore the public request path is:

```text
Client
   |
   v
Cloudflare
   |
   v
OPNsense
   |
   v
Caddy
```

Caddy must not treat arbitrary Internet-supplied forwarding headers as authoritative.

The trusted proxy configuration should ensure that client identity is derived from the known Cloudflare proxy path.

Relevant headers include:

```text
CF-Connecting-IP
X-Forwarded-For
CF-IPCountry
```

### Client IP flow

```text
Original Client
      |
      | client IP
      v
Cloudflare
      |
      | CF-Connecting-IP
      v
Caddy
      |
      | trusted client identity
      v
Suricata
      |
      v
Envoy Gateway
      |
      v
Coraza
```

This allows downstream services to correlate application requests with the actual Internet client rather than only the Cloudflare proxy.

### Trust boundary

Client identity headers must **never be blindly trusted from arbitrary external clients**.

The firewall and reverse-proxy configuration must ensure that the public Internet cannot directly reach the internal gateway and inject trusted forwarding headers around the Cloudflare/Caddy path.

The network source address and the application-level client identity represented by an HTTP header are separate pieces of information and should not be conflated.

---

## Upstream Connectivity

Caddy forwards traffic to the Kubernetes Gateway through the internal network.

Current gateway endpoint:

```text
192.168.100.152:80
```

The intended flow is:

```text
Caddy
  |
  | HTTP
  v
192.168.100.152:80
  |
  v
Suricata IDS/IPS
  |
  v
Envoy Gateway
  |
  v
Coraza
  |
  v
Kubernetes Services
```

The upstream endpoint should not be directly exposed to the Internet.

Only the intended Caddy path should provide public reverse-proxy access into the gateway.

Direct access to Envoy Gateway or Kubernetes services is not part of the public architecture.

---

## Network Exposure

Caddy is reachable from the Internet **only through the intended Cloudflare → OPNsense path**.

The public ingress path is:

```text
Internet
   |
   v
Cloudflare
   |
   v
OPNsense
   |
   v
Caddy
```

The internal security path then continues:

```text
Caddy
   |
   v
Suricata IDS/IPS
   |
   v
Envoy Gateway
   |
   v
Coraza
   |
   v
Kubernetes
```

Direct access to the Kubernetes gateway is not part of the intended public architecture.

The firewall should therefore prevent unintended external access to:

```text
192.168.100.152
```

and other internal Kubernetes endpoints.

---

## Security Responsibilities

Caddy deliberately does **not** implement the full application-security stack.

Caddy is responsible for:

```text
TLS
  +
Reverse Proxy
  +
Trusted Client Identity
  +
Access Logging
```

The downstream security layers are responsible for their respective controls:

```text
Suricata
  |
  +-- Network IDS/IPS
  +-- Protocol inspection
  +-- Signature detection
  +-- IPS enforcement

Envoy Gateway
  |
  +-- Gateway API
  +-- HTTP routing

Coraza
  |
  +-- HTTP WAF
  +-- OWASP CRS
  +-- Application-aware request filtering

CrowdSec
  |
  +-- Behavioral detection
  +-- Correlation
  +-- Security decisions
```

Caddy is not responsible for:

```text
WAF rules
Application attack detection
Behavioral IP banning
Kubernetes authorization
Application authentication
```

Keeping these responsibilities separate reduces configuration complexity and avoids duplicating security controls between services.

---

## Request Flow

A normal request follows:

```text
Client
   |
   v
Cloudflare
   |
   | HTTPS
   v
OPNsense
   |
   | HTTPS
   v
Caddy
   |
   | TLS termination
   |
   | HTTP + trusted client identity
   v
Suricata IDS/IPS
   |
   v
Envoy Gateway
   |
   v
Coraza WAF
   |
   v
Kubernetes Service
   |
   v
Application Pod
```

Caddy's responsibility ends when the request has been successfully handed to the configured internal security/gateway path.

---

## Failure Behavior

Caddy should fail closed with respect to its configured upstream path.

If the Kubernetes Gateway or its upstream security path cannot be reached, Caddy should return an appropriate upstream error rather than attempting to bypass the configured architecture.

Examples include:

```text
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

Caddy must not:

* Route around the configured gateway
* Connect directly to application pods
* Bypass Suricata
* Bypass Envoy Gateway
* Create an alternate public path
* Fall back to an unapproved upstream

This preserves the intended security boundary even when downstream services are unavailable.

### Availability monitoring

Caddy should be monitored for:

* Process health
* Listener availability
* Certificate expiry
* Certificate renewal failures
* Upstream connectivity failures
* Excessive 502/503/504 responses
* Configuration reload failures
* Unexpected restarts
* Access-log delivery failures

A reverse proxy failure should be visible as an infrastructure event rather than silently appearing as application downtime.

---

## Operational Principles

Caddy configuration should follow these principles.

### Keep it simple

Caddy should perform only the functions required of a reverse proxy and TLS termination layer.

### Keep the trust boundary explicit

Only known upstream proxies should be trusted for client identity.

### Keep the upstream controlled

Traffic should only be forwarded to the intended internal gateway path.

### Preserve the security path

Caddy must not provide an alternate route around Suricata, Envoy Gateway, or Coraza.

### Keep logs useful

Access logs should contain sufficient information for troubleshooting and security analysis without unnecessary noise.

### Avoid duplicated security controls

Application security belongs downstream of Caddy.

Caddy should not become another WAF or behavioral security engine.

### Make failures observable

Certificate, proxy, upstream, and configuration failures should be visible through monitoring and alerting.

---

## 📊 Relationship to Other Security Layers

| Layer             | Primary Responsibility                                  |
| ----------------- | ------------------------------------------------------- |
| **Cloudflare**    | Public edge filtering, challenges, and edge enforcement |
| **OPNsense / pf** | Firewall and network enforcement                        |
| **Caddy**         | TLS termination and reverse proxy                       |
| **Suricata**      | Network IDS/IPS                                         |
| **Envoy Gateway** | Kubernetes gateway and HTTP routing                     |
| **Coraza**        | HTTP WAF and OWASP CRS                                  |
| **CrowdSec**      | Behavioral detection and automated decisions            |
| **Datadog**       | Security and operational observability                  |
| **Kubernetes**    | Application workloads                                   |

Caddy is therefore a **transport and proxy layer**, not the central security decision engine.

---

## 📡 Request Path vs Telemetry Path

The request path is:

```text
Internet
   |
   v
Cloudflare
   |
   v
OPNsense
   |
   v
Caddy
   |
   v
Suricata IDS/IPS
   |
   v
Envoy Gateway
   |
   v
Coraza WAF
   |
   v
Kubernetes
```

Telemetry is collected independently:

```text
Cloudflare ───────────────┐
OPNsense ─────────────────┤
Caddy ────────────────────┤
Suricata ─────────────────┤
Envoy ────────────────────┼──> Datadog
Coraza ──────────────────┤
CrowdSec ────────────────┘
```

Caddy contributes access telemetry to this observability path but does not perform centralized security enforcement.

---

## 🧱 Defense in Depth

Caddy is intentionally one component of a layered security architecture.

A request may be processed by several independent controls:

```text
Cloudflare
    |
    | Edge security
    v
OPNsense
    |
    | Firewall / network policy
    v
Caddy
    |
    | TLS termination / proxy
    v
Suricata
    |
    | Network IDS/IPS
    v
Envoy Gateway
    |
    | Gateway routing
    v
Coraza
    |
    | HTTP WAF
    v
Kubernetes
```

Behavioral security operates independently:

```text
Application telemetry
        |
        v
     CrowdSec
        |
        v
     Decision
        |
        +------> Cloudflare enforcement
        |
        +------> OPNsense / pf enforcement
```

Caddy does not replace these controls and does not attempt to absorb their responsibilities.

---

## 🔐 Production Security Considerations

The Caddy deployment should maintain the following properties:

### No direct application exposure

Applications should only be reachable through the intended gateway path.

### No security-path bypass

Caddy must not provide a route that bypasses Suricata, Envoy Gateway, or Coraza.

### Controlled header trust

Forwarded client identity must only be accepted from trusted proxy sources.

### Controlled upstreams

Only explicitly configured internal gateway endpoints should be reachable through Caddy.

### Certificate lifecycle

Certificate issuance, renewal, expiry, and failures must be observable.

### Configuration lifecycle

Caddy configuration changes should be version-controlled, reviewed, and recoverable.

### Observable operation

Proxy errors, TLS failures, upstream failures, and abnormal request patterns should be visible in the monitoring stack.

---

## Summary

Caddy provides the controlled transition from the **network perimeter** into the **internal security and Kubernetes gateway path**.

```text
              NETWORK PERIMETER

                     |
                     v

                Cloudflare

                     |
                     v

                 OPNsense

                     |
                     v

              +-------------+
              |    CADDY    |
              |             |
              | TLS         |
              | Proxy       |
              | Client IP   |
              | Logging     |
              +------+------+
                     |
                     v
              Suricata IDS/IPS
                     |
                     v
                Envoy Gateway
                     |
                     v
                  Coraza
                     |
                     v
                 Kubernetes
```

Caddy's design goal is simple:

> **Terminate TLS, preserve trusted client identity, proxy traffic through the controlled security path to the Kubernetes Gateway, and provide reliable request logging — without becoming another application-security layer.**
