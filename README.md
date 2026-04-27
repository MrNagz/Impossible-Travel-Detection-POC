
# Impossible Travel Detection – Proof of Concept

## Overview

This project demonstrates a proof-of-concept implementation of an **Impossible Travel** detection pipeline using the Elastic Stack.

The objective is to identify suspicious or physically implausible authentication activity by comparing consecutive login events for a given user and calculating the implied travel speed between them.

Rather than relying on a single static rule, this approach focuses on:

* contextual enrichment
* stateful user tracking
* realistic physical constraints
* tiered detection logic

This design aims to reduce false positives while maintaining strong detection fidelity.

---

## Problem Statement

Traditional impossible travel detections often suffer from high false positive rates due to:

* lack of context between authentication events
* inconsistent field normalization
* geolocation inaccuracies
* absence of realistic travel constraints

This project explores a more structured approach that separates ingestion, enrichment, and detection logic into distinct stages.

---
## Proof of Concept Scope

This project is intended as a proof of concept and not a production-ready detection.

The focus is on demonstrating:
- ingestion and normalization workflows
- stateful enrichment using transforms
- realistic travel-based detection logic
- reduction of false positives through contextual analysis
---
## Architecture

flowchart LR
    A[SSH Auth Logs<br/>system.auth] --> B[Default Pipeline<br/>logs-system.auth-2.6.3]
    B --> C[Custom Pipeline<br/>logs-system.auth@custom]
    C --> D[GeoIP Normalize Pipeline<br/>auth_geoip_normalize]

    D --> E[Indexed Logs<br/>logs-system.auth-*]

    E --> F[Transform<br/>auth_latest_by_user]
    F --> G[Latest Login Index<br/>auth_latest_by_user]

    G --> H[Enrich Policy<br/>auth_latest_by_user_policy]

    E --> I[Travel Pipeline<br/>impossible_travel_enrich]
    H --> I

    I --> J[Enriched Events<br/>travel.* fields]

    J --> K[Detection Rules]
    K --> L[Alerts / Dashboard]

The detection pipeline is composed of multiple stages:

### 1. Log Ingestion

Authentication logs are collected from a Linux-based VPS using Elastic Agent (`system.auth` dataset). These logs include SSH login activity such as:

* user identity
* timestamp
* source IP address
* authentication outcome

---

### 2. Normalization and GeoIP Enrichment

A custom ingest pipeline is introduced via the dataset-specific hook:

```
logs-system.auth@custom
```

This pipeline calls a dedicated normalization pipeline:

```
auth_geoip_normalize
```

Responsibilities:

* Normalize authentication outcome fields (`event.outcome`)
* Enrich `source.ip` with GeoIP data:

  * `source.geo.location`
  * `source.geo.country_name`
* Enrich ASN information:

  * `source.as.organization.name`

This step ensures that all downstream processing operates on consistent, enriched data.

---

### 3. Stateful User Context (Transform)

A continuous transform maintains the most recent successful login per user:

```
auth_latest_by_user
```

This creates a lightweight lookup index representing the latest known authentication state for each user, enabling efficient comparisons without querying large volumes of historical data.

---

### 4. Enrichment with Previous Login

An enrich policy:

```
auth_latest_by_user_policy
```

is used to attach previous login context to new authentication events under:

```
prev_login
```

This provides:

* previous login timestamp
* previous source IP
* previous geolocation
* previous ASN information

---

### 5. Travel Calculation Pipeline

A second ingest pipeline:

```
impossible_travel_enrich
```

performs the core logic:

* Calculates geographic distance between logins
* Calculates time difference
* Derives travel speed (km/h)

The pipeline includes suppression logic:

* Ignore events with time difference greater than 48 hours
* Ignore events with distance less than 300 km

---

### 6. Classification Logic

Each authentication event is categorized into a travel tier:

| Tier       | Speed (km/h) | Interpretation                        |
| ---------- | ------------ | ------------------------------------- |
| Normal     | < 500        | Realistic travel                      |
| Possible   | 500–900      | Consistent with commercial air travel |
| Unlikely   | 900–1500     | Improbable but technically possible   |
| Impossible | > 1500       | Exceeds physical travel constraints   |

Additional contextual fields are derived:

* `travel.country_change`
* `travel.asn_change`
* `travel.reason`
* `travel.pipeline_version`

---

## Detection Strategy

Detection rules are tiered to balance sensitivity and noise reduction:

### Critical

```
travel.tier = "impossible"
```

### High

```
travel.tier = "unlikely"
AND (travel.country_change = true OR travel.asn_change = true)
```

### Medium

```
travel.tier = "possible"
AND travel.country_change = true
AND travel.asn_change = true
```

This approach ensures:

* high-confidence signals alert immediately
* lower-confidence signals require corroborating indicators

---

## Testing Methodology

Testing was conducted in two phases:

### Synthetic Testing

Controlled events were generated to simulate:

* normal travel behavior
* unlikely travel speeds
* physically impossible travel scenarios

This validated the pipeline logic and classification accuracy.

### Real-World Testing

Live SSH authentication events were generated from geographically distinct locations.

Validation confirmed:

* GeoIP enrichment applied correctly to public IPs
* transform accurately tracked latest user login state
* enrich policy correctly attached prior login context
* travel calculations and classification executed as expected

---

## Observations

During testing, it was observed that:

* Overlay networks (e.g., Tailscale) produce non-geolocatable IP addresses
* Such events do not support GeoIP-based correlation and must be excluded or handled separately

This highlights a key limitation of geo-based detections when dealing with private or tunneled network traffic.

---

## Design Considerations

* Separation of normalization and detection logic into distinct pipelines
* Use of transforms for efficient state management
* Use of enrich policies to avoid expensive runtime joins
* Application of suppression logic to reduce false positives
* Alignment with ECS field conventions

---

## Conclusion

This proof of concept demonstrates a structured and extensible approach to impossible travel detection using the Elastic Stack.

By combining:

* normalized ingestion
* stateful context
* enrichment
* and tiered classification

the system produces meaningful, prioritized detection signals while minimizing noise.

This design can be extended to additional authentication sources such as:

* VPN logs
* identity providers
* cloud authentication services

---

## Future Improvements

* Incorporate user baseline behavior modeling
* Add session overlap detection
* Integrate device fingerprinting
* Expand ASN and country-based anomaly scoring
* Automate enrich policy refresh workflows

---
