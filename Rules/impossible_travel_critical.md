
# Impossible Travel – Critical

## Description
Detects physically impossible travel between consecutive authentication events.

## Logic
travel.tier = "impossible"

## Severity
Critical

## Rationale
Travel speeds exceeding 1500 km/h are not physically achievable via commercial or private transportation.

## Data Requirements
- source.geo.location
- prev_login.source.geo.location
- event.outcome = success

## False Positive Considerations
- Incorrect GeoIP mapping
- Shared infrastructure or VPN exit nodes
