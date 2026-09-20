# CONCEPT: DYNAMIC AVIAN CONSTRAINT LAYER FOR UTM SYSTEMS

**Version:** 1.0

**Date:** September 2026

**Status:** Open Technical Contribution (No Commercial Interest)


## 1. CONTEXT

NASA UTM Concept of Operations (ConOps) v2.0, Section 2.4.5, identifies
bird activity as a hazard type within the UTM framework. However, the
current approach relies on static advisories and manual reporting.
There is no mechanism to transform real-time wildlife activity into
dynamic, machine-readable constraints for conflict detection and
resolution.

EASA U-space Regulation (EU) 2021/664 addresses environmental geo-zones
(Article 4), geo-awareness (Article 8), and dynamic constraints
(Article 9), but similarly lacks a standardized method for generating
dynamic geo-zones based on real-time wildlife presence.

This concept proposes a mechanism to bridge that gap.


## 2. CORE IDEA

A relay chain of UAS (unmanned aerial systems) equipped with thermal
imaging and lightweight CNN-based detection continuously monitors
bird flocks along a migration corridor. Each UAS covers a 40-50 km
sector. When a flock approaches the sector boundary, a handoff
protocol transfers tracking responsibility to the next UAS in the
chain.

The output is a series of dynamic geo-fences — UAS Volume Reservations
(UVRs) or UREP-compatible data — fed directly into the UTM network.
These geo-fences function as temporary constraints, enabling automated
deconfliction for other UAS operations in the affected corridor.


## 3. WHY A RELAY CHAIN

A single UAS cannot cover an entire migration corridor (typically
200-500 km). A relay chain solves this by:

  - Dividing the corridor into manageable sectors (40-50 km each).
  - Ensuring continuous tracking through structured handoff protocols.
  - Allowing scalable deployment (add/remove sectors seasonally).
  - Providing redundancy (overlap zones between adjacent sectors).


## 4. KEY COMPONENTS

  Detection:
  
    - Thermal camera (640x512, FOV 30-45 degrees).
    
    - Lightweight CNN (MobileNetV3) for real-time detection and
      classification.
      
    - Detection range: >= 500 m.

  Tracking:
  
    - Kalman filter for position and velocity prediction.
    
    - Flock centroid, heading, speed, uncertainty radius.

  Handoff:
  
    - JSON messages over MAVLink or IP.
    
    - Triggered when flock enters handoff zone (last 20% of sector).
    
    - Failure handling: retry, fallback to extended tracking.
    
    - Splitting/merging logic for flock dynamics.

  Geo-fence Generation:
  
    - Dynamic polygon around active flocks.
    
    - Updated every 30-60 seconds.
    
    - Altitude band: +/-100 m around flock altitude.
    
    - Transmitted to USS as UVR/UREP.


## 5. REGULATORY ALIGNMENT

NASA UTM ConOps v2.0:
  - Section 2.4.5 (bird activity as hazard) — addressed by dynamic
    constraint mechanism.
  - Section 3.2 (USS responsibilities) — addressed by integration
    with USS.

EASA U-space Regulation (EU) 2021/664:
  - Article 4 (environmental geo-zones) — addressed by dynamic
    geo-fences.
  - Article 8 (geo-awareness) — addressed by real-time updates.
  - Article 9 (dynamic constraints) — addressed by UVR/UREP format.


## 6. ENVISAGED DEPLOYMENT

  Phase 1: Simulation and offline testing (months 1-2).
  
  Phase 2: Hardware-in-loop testing (months 3-4).
  
  Phase 3: Field testing with simulated targets (months 5-6).
  
  Phase 4: Integration with USS testbed (months 7-8).


## 7. OPEN QUESTIONS

  1. Optimal sector sizing based on UAS endurance and flock speed.
  2. Real-world classification accuracy for mixed-species flocks.
  3. Regulatory pathway for dynamic environmental UVRs.
  4. Interoperability with existing UTM data exchange protocols.
  5. Handling of very large flocks (>1000 birds) with single UAS.
  6. Cybersecurity requirements for handoff messages.
  7. Long-term reliability of onboard CNN in varying conditions.
  8. Cost-benefit analysis for large-scale deployment.


## 8. ABOUT THE AUTHOR

Alex Malakhov

Independent researcher — Mathematical and computer modeling  
Email: m4prjcts@gmail.com

This concept is offered as an open technical contribution to the UTM
community. No commercial interest, no funding requested. The author is
available for technical review and remote consultation upon request.
