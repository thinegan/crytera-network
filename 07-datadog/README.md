# Datadog – Security Observability & Dashboards

This document describes how **Datadog** provides the **central security observability and operational analytics layer** for the 2026 network and application security architecture.

Datadog aggregates telemetry from **Cloudflare, OPNsense, Caddy, Suricata, Envoy Gateway, Coraza, CrowdSec, and Kubernetes workloads** into a centralized platform for monitoring, correlation, investigation, and historical analysis.

Datadog is **out of band** and does not participate in the request path.

> **Datadog observes security controls. It does not enforce them.**

All traffic enforcement remains with the appropriate security control:

* Cloudflare
* OPNsense / pf
* Suricata IDS/IPS
* Coraza WAF
* CrowdSec bouncers
* Cloudflare Worker / KV enforcement

---

## 🎯 Role of Datadog

**Primary purpose:**
👉 **Observe, correlate, investigate, and visualize security events**

Datadog provides:

* Centralized security log ingestion
* Infrastructure and application observability
* Geo-based attacker visibility
* Attack-pattern trending
* Cross-layer event correlation
* Security-control monitoring
* Enforcement visibility
* Historical investigation
* Operational alerting
* Dashboarding and reporting

Datadog does **not** sit inline with production traffic.

A Datadog outage therefore does not directly determine whether a request is allowed or blocked.

---

## 🌐 Request Path vs Telemetry Path

The request path and telemetry path are intentionally separate.

### Request / Enforcement Path

```text
Internet
   |
   v
Cloudflare
   |
   v
OPNsense Firewall
   |
   v
Caddy
   |
   | TLS termination
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
Kubernetes Services / Pods
```

Each control is responsible for its own enforcement decision.

### Telemetry Path

```text
Cloudflare ───────────────┐
OPNsense ─────────────────┤
Caddy ────────────────────┤
Suricata ─────────────────┤
Envoy Gateway ────────────┼──► Datadog
Coraza ───────────────────┤
CrowdSec #1 ──────────────┤
CrowdSec #2 ──────────────┤
Kubernetes ───────────────┘
```

Datadog consumes telemetry from these systems without becoming part of the traffic path.

---

# 📥 Log Sources Ingested

## ☁️ Cloudflare

**Purpose:** Edge security and public request visibility.

Cloudflare telemetry can provide:

* HTTP request metadata
* Edge response status
* Firewall events
* Managed Challenge events
* Bot/scanner activity
* Origin response information
* Client country
* Client IP information
* Edge enforcement events

### Useful fields

Typical normalized fields include:

```text
client.ip
http.host
http.method
http.url
http.status_code
http.useragent
geo.country
cloudflare.action
cloudflare.rule_id
```

### Used for

* Edge attack visibility
* Challenge/block analysis
* Scanner detection
* Origin traffic analysis
* Geographic trends
* Correlation with downstream controls

Cloudflare is an **enforcement layer**, while Datadog provides visibility into what Cloudflare observed and enforced.

---

## 🔥 OPNsense / pf

**Purpose:** Network perimeter and firewall visibility.

Telemetry may include:

* Firewall denies
* NAT activity
* Connection metadata
* CrowdSec dynamic pf enforcement
* Interface/network events

### Used for

* Perimeter attack visibility
* Blocked connection analysis
* Source IP investigation
* CrowdSec firewall enforcement validation
* Network-level incident correlation

OPNsense remains the enforcement point.

Datadog only observes the resulting events.

---

## 🔐 Caddy

**Purpose:** Reverse-proxy and TLS-termination observability.

Caddy provides structured access logs containing information such as:

* Client IP
* Host
* HTTP method
* URI
* Status
* User agent
* Request duration
* Upstream information

Caddy is particularly important because it is the **TLS termination point** before traffic reaches the internal inspection and gateway layers.

### Used for

* Request visibility
* TLS termination analysis
* Upstream failures
* HTTP error trends
* Client attribution
* Request latency
* Correlation with Envoy and WAF events

Caddy is a reverse proxy and TLS termination layer, not the primary WAF.

---

# 🛡️ Suricata – Network IDS / IPS

**Purpose:** Network-level threat detection and prevention.

Suricata operates as an **IDS/IPS**, providing both detection and rule-driven enforcement.

### Log type

Primarily:

```text
eve.json
```

Typical telemetry includes:

* Alerts
* Network protocol events
* Flow information
* DNS events
* HTTP events where available
* TLS events
* IPS/drop events

### Used for

* Network reconnaissance
* Scanner detection
* Exploit signatures
* Protocol anomalies
* Known attack patterns
* Suspicious network activity
* IPS enforcement visibility

### Important distinction

Suricata is not merely an alerting sensor.

```text
Traffic
   |
   v
Suricata
   |
   ├── Detection
   |
   └── IPS enforcement
            |
            └── Drop / Block
```

Datadog monitors both the detection events and the resulting enforcement activity.

Suricata can inspect the decrypted HTTP traffic available at its observation point, subject to protocol parsing, stream reassembly, IPS topology, and rule configuration.

---

# 🚪 Envoy Gateway

**Purpose:** Kubernetes gateway and HTTP routing observability.

Envoy Gateway provides telemetry around:

* HTTP requests
* Routes
* Upstream services
* Response codes
* Request latency
* Connection failures
* Upstream failures
* Client identity
* Gateway errors

### Used for

* Request-volume analysis
* 4xx / 5xx analysis
* Latency monitoring
* Route-level investigation
* Upstream failure detection
* CrowdSec behavioral detection input
* Correlation with Coraza WAF events

Envoy Gateway is the **routing layer**.

It is not itself the primary WAF.

---

# 🛡️ Coraza WAF + OWASP CRS

**Purpose:** Application-layer HTTP inspection and enforcement.

Coraza provides the HTTP-aware WAF layer with **OWASP CRS** rules.

### Used for

* SQL injection detection
* XSS detection
* Path traversal
* Local file inclusion attempts
* Remote code execution patterns
* Sensitive-file probing
* Protocol violations
* CRS anomaly-score enforcement

Typical WAF telemetry includes:

```text
client.ip
http.method
http.uri
http.status_code
waf.rule_id
waf.rule_message
waf.anomaly_score
waf.action
```

### Example events

```text
GET /.env
GET /.git/config
GET /wp-login.php
GET /?id=' OR 1=1
```

Datadog is used to determine:

* Which WAF rules triggered
* Which paths were targeted
* Which clients generated violations
* Whether requests were blocked
* Whether false positives are occurring
* Which applications are receiving attack traffic

Coraza remains the **HTTP enforcement layer**.

---

# 🤖 CrowdSec

There are two distinct CrowdSec security domains.

They should be represented separately in Datadog.

---

## CrowdSec #1 – Network / Perimeter

**Purpose:** Network and infrastructure behavioral detection.

Typical inputs can include:

* OPNsense telemetry
* Caddy logs
* SSH-related events
* Firewall events
* Other perimeter security telemetry

Flow:

```text
Network / Infrastructure Logs
            |
            v
       CrowdSec #1
            |
            v
       Detection
            |
            v
        Decision
            |
            v
 OPNsense Firewall Bouncer
            |
            v
           pf
            |
            v
          BLOCK
```

### Datadog monitors

* Scenario triggers
* Decisions
* Bouncer activity
* Banned IPs
* Ban duration
* Recurring offenders
* Geographic distribution
* Enforcement activity

---

## CrowdSec #2 – Kubernetes / Application

**Purpose:** Application-layer behavioral detection.

CrowdSec #2 consumes application telemetry, primarily from the Kubernetes gateway/application layer.

Examples include repeated:

* `401`
* `403`
* `404`

and other behavior-based security signals.

Flow:

```text
Envoy Gateway
      |
      v
CrowdSec #2
      |
      v
Behavioral Detection
      |
      v
CrowdSec Decision
      |
      | 4 hour decision
      v
Cloudflare Worker / KV
      |
      v
Cloudflare
      |
      v
BLOCK
```

CrowdSec #2 is therefore **not inline**.

It observes behavior over time, creates a decision, and delegates enforcement to the Cloudflare edge.

### Datadog monitors

* Scenario matches
* Decisions created
* Decision expiry
* Enforcement requests
* Worker/KV errors
* Blocked IPs
* Recurring offenders
* False-positive indicators

---

# 📊 Dashboards Overview

## ⭐ Security Overview Dashboard

The primary dashboard provides a high-level view of the complete security architecture.

### Focus

* Requests observed
* Requests blocked
* WAF blocks
* Suricata alerts
* Suricata IPS drops
* OPNsense firewall denies
* CrowdSec decisions
* Cloudflare blocks/challenges
* HTTP 4xx / 5xx
* Top attacking IPs
* Top attacking countries
* Security events over time

The goal is to answer:

> **What is happening across the security stack right now?**

---

## ⭐ Suricata – IDS / IPS Dashboard

### Focus

* Top signatures
* Alert volume
* IPS drop volume
* Protocol distribution
* Source IPs
* Destination services
* Attack categories
* Geographic distribution
* Time-based trends

### Value

* Network threat visibility
* Reconnaissance detection
* Exploit attempt visibility
* IPS enforcement validation
* Rule effectiveness monitoring

---

## ⭐ Coraza WAF Dashboard

### Focus

* CRS rule IDs
* WAF violations
* Blocked requests
* Anomaly scores
* Targeted paths
* HTTP methods
* Client IPs
* Countries
* Application/service
* 403 trends

### Value

* Application-layer attack visibility
* WAF tuning
* False-positive investigation
* Rule effectiveness
* Exploit-attempt analysis

---

## ⭐ CrowdSec #1 Dashboard

### Focus

* Network scenarios
* Decisions
* Bans
* Bouncer activity
* Ban duration
* Recurring offenders
* Source countries

### Value

* Perimeter behavioral detection
* Firewall enforcement validation
* Attacker recurrence analysis

---

## ⭐ CrowdSec #2 Dashboard

### Focus

* 401/403/404 behavioral patterns
* Scenario matches
* Decisions
* Decision age
* Cloudflare enforcement
* Worker/KV activity
* Recurring offending IPs

### Value

* Application abuse detection
* Behavioral escalation analysis
* Edge remediation validation
* Detection-to-enforcement visibility

---

## ⭐ Cloudflare Edge Dashboard

### Focus

* Requests
* Blocks
* Challenges
* Scanner/probe activity
* Countries
* Hostnames
* HTTP methods
* Response codes
* Origin responses

### Value

* Edge security visibility
* Public attack trends
* Cloudflare enforcement validation
* Correlation with origin-side controls

---

## ⭐ Gateway / Application Dashboard

### Focus

* Request volume
* 2xx / 3xx / 4xx / 5xx
* Latency
* Upstream errors
* Route-level traffic
* Service-level traffic
* WAF-triggered requests

### Value

* Application health
* Security/application correlation
* Detection of abnormal traffic patterns

---

# 🧠 Correlation Strategy

Datadog enables cross-layer correlation between independent security controls.

For example:

| Layer       | Event                           |
| ----------- | ------------------------------- |
| Cloudflare  | Edge block / challenge          |
| OPNsense    | Firewall deny                   |
| Caddy       | HTTP request observed           |
| Suricata    | Network signature / IPS drop    |
| Envoy       | HTTP request / response         |
| Coraza      | WAF violation / block           |
| CrowdSec #1 | Perimeter behavioral decision   |
| CrowdSec #2 | Application behavioral decision |
| Worker / KV | Edge remediation                |
| Kubernetes  | Application response            |

This allows investigation questions such as:

* Did Cloudflare block the request before it reached the origin?
* Did the request reach Caddy?
* Did Suricata detect or drop it?
* Did Envoy route it?
* Did Coraza block it?
* Did the same IP generate repeated 401/403/404 responses?
* Did CrowdSec create a decision?
* Was the decision propagated to Cloudflare?
* Did subsequent requests get blocked at the edge?

---

# 🔎 Detection vs Enforcement

Datadog documentation should preserve the distinction between **observing a security event** and **enforcing a security policy**.

| Component   | Detection |     Decision |          Enforcement |
| ----------- | --------: | -----------: | -------------------: |
| Cloudflare  |         ✅ |            ✅ |                    ✅ |
| OPNsense    |         ✅ |            ✅ |                    ✅ |
| Suricata    |         ✅ |  Rule-driven |                ✅ IPS |
| Caddy       |      Logs |            ❌ |       Proxy behavior |
| Envoy       | Telemetry |            ❌ |              Routing |
| Coraza      |         ✅ | Rule/anomaly |                ✅ WAF |
| CrowdSec #1 |         ✅ |            ✅ |     OPNsense bouncer |
| CrowdSec #2 |         ✅ |            ✅ | Cloudflare Worker/KV |
| Datadog     | ✅ Observe |            ❌ |                    ❌ |

The key principle is:

> **Datadog never becomes the enforcement dependency.**

---

# 📈 Security Metrics

Important metrics to expose in Datadog include:

### Network

```text
suricata.alerts
suricata.ips_drops
firewall.denies
network.connections
```

### HTTP / Gateway

```text
http.requests
http.4xx
http.5xx
http.latency
envoy.upstream_errors
```

### WAF

```text
waf.requests
waf.blocks
waf.rule_matches
waf.anomaly_scores
```

### CrowdSec

```text
crowdsec.scenario_matches
crowdsec.decisions
crowdsec.bans
crowdsec.bouncer_events
```

### Cloudflare

```text
cloudflare.requests
cloudflare.blocks
cloudflare.challenges
cloudflare.origin_errors
```

### Infrastructure

```text
kubernetes.pods
kubernetes.restarts
node.cpu
node.memory
node.disk
```

These metrics should be correlated with logs wherever possible.

---

# 🚨 Security-Control Health

Security observability is not limited to attacker activity.

Datadog should also monitor whether the security controls themselves are operating correctly.

### Suricata

Monitor:

* Engine availability
* Rule loading
* Alert rate
* IPS drop rate
* Interface health
* Capture errors

### Coraza

Monitor:

* WAF errors
* Rule loading
* Block rate
* CRS rule IDs
* Anomaly scores
* Processing errors

### CrowdSec

Monitor:

* LAPI availability
* Scenario processing
* Decision creation
* Decision expiry
* Bouncer connectivity
* Enforcement failures

### Cloudflare Worker / KV

Monitor:

* Worker errors
* KV errors
* Decision propagation failures
* Stale decisions
* Enforcement latency

### Envoy / Caddy

Monitor:

* Availability
* Upstream failures
* 5xx responses
* Latency
* Connection failures

This ensures Datadog can answer both:

> **Are attackers being detected?**

and:

> **Are our security controls actually functioning?**

---

# 🌍 Geographic Analysis

Geo-based dashboards provide visibility into:

* Top source countries
* Attack volume by country
* WAF violations by country
* Suricata events by country
* CrowdSec decisions by country
* Cloudflare enforcement by country

Geographic data is useful for **trend analysis and investigation**, but should not be treated as proof that traffic is malicious.

Client identity should be derived from trusted proxy metadata only.

For example:

```text
Cloudflare
    |
    v
OPNsense
    |
    v
Caddy
    |
    v
Envoy
    |
    v
Kubernetes
```

Headers such as:

```text
CF-Connecting-IP
CF-IPCountry
X-Forwarded-For
```

must only be trusted when they arrive through the known and controlled proxy chain.

---

# 🔗 Detection → Decision → Enforcement

Datadog should make the security feedback loops visible.

### Network / Perimeter CrowdSec

```text
Event
  |
  v
Detection
  |
  v
CrowdSec Decision
  |
  v
OPNsense Bouncer
  |
  v
pf BLOCK
  |
  v
Datadog
```

### Kubernetes / Application CrowdSec

```text
HTTP Behavior
     |
     v
CrowdSec #2
     |
     v
Decision
     |
     v
Worker / KV
     |
     v
Cloudflare BLOCK
     |
     v
Datadog
```

### WAF

```text
HTTP Request
     |
     v
Coraza
     |
     v
CRS Rule Match
     |
     v
WAF BLOCK
     |
     v
Datadog
```

### Network IPS

```text
Network Traffic
     |
     v
Suricata
     |
     v
Signature / Protocol Detection
     |
     v
IPS DROP
     |
     v
Datadog
```

Datadog observes each stage without becoming part of the enforcement chain.

---

# 🧩 Observability Design Principles

### 1. Observe out of band

Datadog must not be required for a request to succeed or fail.

### 2. Preserve source identity

Normalize client IP, country, hostname, request ID, and relevant proxy metadata consistently across logs.

### 3. Correlate using common identifiers

Where available, correlate:

* Client IP
* Request ID
* Hostname
* URI
* Timestamp
* User agent
* WAF rule ID
* Suricata signature ID
* CrowdSec scenario
* CrowdSec decision ID

### 4. Monitor enforcement, not just detection

A security alert without confirmation of enforcement can leave an incomplete operational picture.

### 5. Monitor the security controls themselves

A silent security system can mean either:

* no attacks occurred, or
* the security control stopped working.

Datadog should make that distinction observable.

### 6. Keep detection and enforcement ownership explicit

Each security component should have a defined enforcement responsibility.

---

# 🏗️ Security Observability Architecture

```text
                           ┌─────────────────────┐
                           │     Cloudflare      │
                           │ Edge Security       │
                           └──────────┬──────────┘
                                      │
                           ┌──────────▼──────────┐
                           │      OPNsense       │
                           │ Firewall / pf       │
                           └──────────┬──────────┘
                                      │
                           ┌──────────▼──────────┐
                           │       Caddy         │
                           │ TLS / Reverse Proxy │
                           └──────────┬──────────┘
                                      │
                           ┌──────────▼──────────┐
                           │      Suricata       │
                           │      IDS / IPS      │
                           └──────────┬──────────┘
                                      │
                           ┌──────────▼──────────┐
                           │   Envoy Gateway     │
                           │ HTTP / Gateway API  │
                           └──────────┬──────────┘
                                      │
                           ┌──────────▼──────────┐
                           │   Coraza + CRS      │
                           │       WAF           │
                           └──────────┬──────────┘
                                      │
                           ┌──────────▼──────────┐
                           │     Kubernetes      │
                           │     Workloads       │
                           └─────────────────────┘


 Cloudflare ───────────────┐
 OPNsense ─────────────────┤
 Caddy ────────────────────┤
 Suricata ─────────────────┤
 Envoy ────────────────────┼──────► Datadog
 Coraza ───────────────────┤
 CrowdSec #1 ──────────────┤
 CrowdSec #2 ──────────────┤
 Kubernetes ───────────────┘
```

---

# 🧠 Why Datadog

Datadog provides a centralized operational view across multiple independent security controls.

Its primary value is:

* Centralized telemetry
* Security event correlation
* Historical analysis
* Operational dashboards
* Infrastructure monitoring
* Application monitoring
* Geographic analysis
* Enforcement visibility
* Security-control health monitoring

Datadog is therefore best described as the **security observability plane**, rather than another inline security control.

---

# 📌 Key Principles

* **Cloudflare** protects the public edge.
* **OPNsense** enforces network perimeter policy.
* **Caddy** terminates TLS and proxies trusted traffic.
* **Suricata** detects and blocks network threats.
* **Envoy Gateway** provides Kubernetes gateway and routing.
* **Coraza + OWASP CRS** enforces HTTP WAF policy.
* **CrowdSec #1** detects network/perimeter abuse and delegates enforcement to OPNsense.
* **CrowdSec #2** detects application behavior and delegates enforcement to Cloudflare.
* **Datadog** observes and correlates all of the above.
* **Datadog never blocks traffic.**

---

# 🔚 Summary

Datadog provides the centralized **security observability and operational analytics layer** for the 2026 architecture.

It brings together telemetry from:

```text
Cloudflare
     +
OPNsense
     +
Caddy
     +
Suricata
     +
Envoy Gateway
     +
Coraza
     +
CrowdSec #1
     +
CrowdSec #2
     +
Kubernetes
     |
     v
  Datadog
```

This allows security events to be investigated across the complete request lifecycle:

```text
Observe
   ↓
Correlate
   ↓
Investigate
   ↓
Validate Detection
   ↓
Validate Enforcement
```

The architecture maintains a strict separation of responsibilities:

> **Security controls enforce. Datadog observes.**

This separation keeps Datadog out of the critical request path while still providing centralized visibility into detection, decision-making, enforcement, application behavior, and security-control health.
