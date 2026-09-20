# Technical Specification: Dynamic Avian Constraint Layer for UTM Systems

**Version:** 1.0  
**Date:** September 2026  
**Status:** Open Technical Contribution (No Commercial Interest)

---

## 1. Executive Summary

This specification defines a dynamic wildlife constraint layer for UTM (Unmanned Traffic Management) systems. The layer addresses the gap between regulatory requirements for environmental geo-zones (EASA U-space Regulation (EU) 2021/664, Article 4) and the lack of mechanisms to represent dynamic hazards such as bird flocks.

The core mechanism is a relay chain of UAS (drones) that continuously detect, track, and hand over bird flocks along a corridor. The output is machine-readable dynamic geo-fences (UAS Volume Reservations or UREPs) fed directly into the UTM network.

Key deliverables for implementation:

- Handoff protocol with failure handling and splitting/merging logic.
- Geo-fence generation rules and JSON data format.
- Test scenarios with success criteria for validation.
- Regulatory mapping to EASA and NASA UTM ConOps.

---

## 2. Problem Statement

Current UTM implementations treat environmental hazards as static. However, wildlife (especially migratory birds) is inherently dynamic. This creates a conflict:

- EASA U-space Regulation (EU) 2021/664, Article 4 allows environmental geo-zones to protect wildlife.
- Article 8 requires geo-awareness services.
- Article 9 requires dynamic constraints and notifications.

Yet, there is no standardized method to generate dynamic geo-zones based on real-time wildlife activity. As a result, UTM systems cannot proactively mitigate the risk of bird strikes or protect sensitive habitats during active migration periods.

NASA UTM Concept of Operations (ConOps) v2.0, Section 2.4.5 explicitly lists "bird activity" as a hazard type, but the current approach relies on static advisories or manual reporting. There is no mechanism to turn bird activity into a dynamic, machine-readable constraint for conflict detection and resolution.

Existing avian radar systems provide high-quality local detection near airports and airfields. However, they are typically fixed installations optimised for relatively small volumes of airspace. They do not readily scale to continuous coverage along long migration corridors or the rural and remote routes, where many future BVLOS operations are expected to take place. Consequently, the areas of highest potential interaction between migratory flocks and low-altitude unmanned traffic remain largely under-served by current sensor infrastructure.

---

## 3. Proposed Mechanism

A relay chain of UAS equipped with thermal imaging and lightweight CNN-based detection continuously monitors bird flocks along a corridor. Each UAS covers a 40–50 km sector. When a flock approaches the sector boundary, a handoff protocol transfers tracking responsibility to the next UAS.

**Workflow:**

1. **Detection:** Onboard thermal camera + CNN detects and classifies flocks in real time.
2. **Tracking:** Kalman filter predicts flock position and velocity.
3. **Handoff:** Leading UAS computes handoff point and transmits target coordinates to the next UAS.
4. **Geo-fence generation:** Dynamic polygon around the flock is generated with uncertainty radius.
5. **Integration:** Geo-fences are transmitted to the UTM Service Supplier (USS) as UVRs or UREPs.

---

## 4. Detection and Classification

**Sensor requirements:**

- Thermal camera: resolution ≥ 640×512, FOV 30–45°.
- Payload weight: ≤ 0.5 kg.

**Algorithm:**

- Lightweight CNN (MobileNetV3) for detection and classification.
- Input: thermal frames at 30 Hz.
- Output: bounding boxes, species estimate, count, altitude, speed, direction.

**Performance targets:**

- Detection range: ≥ 500 m.
- Classification accuracy: ≥ 85% for 3 common species.
- Latency: ≤ 200 ms per frame.

**Limitations:**

- Performance degrades in heavy rain/fog.
- Small flocks (<10 birds) may be missed.
- Species classification is probabilistic, not deterministic.

---

## 5. Handoff Protocol

The handoff protocol ensures continuous tracking as flocks move across sectors. It uses JSON messages over MAVLink or IP.

**Handoff initiation:**

- Trigger: flock centroid enters handoff zone (last 20% of sector).
- Leading UAS computes predicted position at boundary and uncertainty radius.

**Message types:**

### Handoff Request

```json
{
  "type": "handoff_request",
  "flock_id": "FLOCK-2026-001",
  "predicted_position": {
    "lat": 46.2000,
    "lon": 6.1000,
    "alt_m": 120
  },
  "velocity_vector": {
    "speed_ms": 18.0,
    "heading_deg": 270
  },
  "uncertainty_radius_m": 50,
  "timestamp": "2026-09-15T10:20:00Z"
}
```

### Handoff Confirmation

```json
{
  "type": "handoff_confirmation",
  "flock_id": "FLOCK-2026-001",
  "receiver_uas_id": "UAS-002",
  "acquisition_status": "acquired",
  "estimated_acquisition_time_s": 15,
  "timestamp": "2026-09-15T10:20:30Z"
}
```

**Failure handling:**

- If no confirmation within 30 s, leading UAS continues tracking and retries.
- If receiver UAS fails to acquire, fallback: leading UAS extends tracking until next handoff opportunity.

**Splitting/merging:**

- Splitting: new flock IDs generated; separate handoffs for each sub-group.
- Merging: unified flock ID; handoff based on centroid of merged group.

---

## 6. Geo-fence Generation

Geo-fences are dynamic polygons around active flocks, updated every 30–60 seconds.

**Parameters:**

- Base radius: 200 m (default).
- Dynamic adjustment: ±50% based on flock size and speed.
- Altitude band: ±100 m around flock altitude.
- Uncertainty buffer: added based on tracking confidence.

**JSON format for USS:**

```json
{
  "type": "geo_fence",
  "fence_id": "GF-2026-001",
  "flock_id": "FLOCK-2026-001",
  "geometry": {
    "type": "Polygon",
    "coordinates": [
      [
        [6.1000, 46.2000],
        [6.1050, 46.2050],
        [6.1100, 46.2000],
        [6.1050, 46.1950],
        [6.1000, 46.2000]
      ]
    ]
  },
  "altitude_band_m": [50, 250],
  "valid_from": "2026-09-15T10:20:00Z",
  "valid_to": "2026-09-15T10:21:00Z",
  "confidence_level": "high",
  "source_uas_id": "UAS-001"
}
```

**Success criteria:**

- Update latency: ≤ 60 s.
- Position error: ≤ 50 m at handoff.
- Tracking gap: ≤ 10 s during handoff.

---

## 7. Corridor Deployment

### Sector design

- Length: 40–50 km per UAS.
- Overlap: 10–15 km between adjacent sectors.
- Handoff zone: last 20% of sector length.

### Fleet sizing

- Minimum: 2 UAS for one handoff.
- For N km corridor: N/40 + 1 UAS (round up).

### Seasonal activation

- Full deployment during migration peaks (March–April, September–October).
- Reduced deployment (every other sector) during off-peak.

### Visual representation of the relay chain

```
       [SECTOR 1]                 [SECTOR 2]                 [SECTOR 3]
   +------------------+       +------------------+       +------------------+
   |                  |       |                  |       |                  |
   |    ( UAS-001 )    |======>|    ( UAS-002 )    |======>|    ( UAS-003 )    |
   |   (Leading)       |       |  (Receiver)      |       | (Next Receiver)  |
   |                  |       |                  |       |                  |
   +----------+-------+       +----------+-------+       +----------+-------+
              |                            |                            |
              v                            v                            v
         [Flock Detected]          [Flock Handed Over]          [Flock Tracked]
          (Lat: 46.2000,            (Lat: 46.1900,              (Lat: 46.1800,
           Lon: 6.1000)             Lon: 6.0900)               Lon: 6.0800)
```

---

## 8. Test Scenarios

### Scenario A: Single flock, 2 UAS, linear corridor

- **Objective:** Validate handoff protocol and tracking continuity.
- **Setup:** Simulated target (drone) acts as flock.
- **Metrics:** handoff success rate, tracking gap duration, position error.
- **Success criteria:** handoff success ≥ 90%, gap < 10 s, error < 50 m.

### Scenario B: Multiple flocks, 3 UAS

- **Objective:** Test concurrent tracking and handoff.
- **Setup:** Two simulated targets at different altitudes.
- **Metrics:** concurrent tracking accuracy, handoff coordination.
- **Success criteria:** ≥ 85% simultaneous tracking, no missed handoffs.

### Scenario C: Flock direction change

- **Objective:** Measure handoff adaptation to sudden direction changes.
- **Setup:** Target changes heading by 90° mid-track.
- **Metrics:** recalculation time, tracking recovery time.
- **Success criteria:** recovery within 15 s, no loss of track.

### Scenario D: Sensor degradation

- **Objective:** Evaluate performance in degraded conditions.
- **Setup:** Artificial fog/rain simulation.
- **Metrics:** detection range reduction, classification accuracy.
- **Success criteria:** detection range ≥ 300 m, accuracy ≥ 75%.

### Scenario E: Handoff failure

- **Objective:** Validate failure handling and fallback logic.
- **Setup:** Receiver UAS simulates failure.
- **Metrics:** fallback success rate, extended tracking duration.
- **Success criteria:** fallback success ≥ 95%, no track loss.

---

## 9. Data Format Specification

All messages use JSON with UTC timestamps (ISO 8601).

**Common fields:**

- `type`: message type.
- `flock_id`: unique identifier.
- `timestamp`: UTC time.
- `source_uas_id`: identifier of sending UAS.

Payload-specific fields as defined in sections 5 and 6.

**Compatibility:**

- MAVLink encapsulation for onboard transmission.
- IP/MQTT for UTM integration.
- ASTM F3548-compatible data elements.

---

## 10. Regulatory Alignment

### Mapping to EASA U-space Regulation (EU) 2021/664

- **Article 4:** environmental geo-zones — satisfied by dynamic geo-fences.
- **Article 8:** geo-awareness services — satisfied by real-time updates.
- **Article 9:** dynamic constraints — satisfied by UVR/UREP format.
- **Article 15:** data exchange — satisfied by JSON/MAVLink/IP formats.

### Mapping to NASA UTM ConOps v2.0

- **Section 2.4.5** (bird activity as hazard) — satisfied by dynamic constraint mechanism.
- **Section 3.2** (USS responsibilities) — satisfied by integration with USS.
- **Technical Memorandum 2014-218299** (wildlife monitoring use case) — satisfied by relay chain design.

---

## 11. Implementation Roadmap

### Phase 1 (Months 1–2): Simulation and offline testing

- Develop detection algorithm, simulate handoff logic, create geo-fence generator.
- Deliverables: algorithm prototype, simulation results, test scripts.

### Phase 2 (Months 3–4): Hardware-in-loop testing

- Integrate with real UAS telemetry, test handoff with simulated targets.
- Deliverables: HIL test results, updated success metrics.

### Phase 3 (Months 5–6): Field testing with simulated targets

- Deploy 2–3 UAS in a 100 km corridor, use drone targets.
- Deliverables: field test report, final metrics, lessons learned.

### Phase 4 (Months 7–8): Integration with USS testbed

- Connect to USS prototype, validate UVR/UREP integration.
- Deliverables: integration report, compatibility confirmation.

---

## 12. Open Questions

These are areas requiring further research and validation:

1. Optimal sector sizing based on UAS endurance and flock speed.
2. Real-world classification accuracy for mixed-species flocks.
3. Regulatory pathway for dynamic environmental UVRs.
4. Interoperability with existing UTM data exchange protocols.
5. Handling of very large flocks (>1000 birds) with single UAS.
6. Cybersecurity requirements for handoff messages.
7. Long-term reliability of onboard CNN in varying conditions.
8. Cost-benefit analysis for large-scale deployment.

---

## 13. About the Author

Alex Malakhov

Independent researcher — Mathematical and computer modeling  
Email: m4prjcts@gmail.com

This specification is offered as an open technical contribution to the UTM community. No commercial interest, no funding requested. The author is available for technical review and remote consultation upon request.
