# High-Availability Navigation for Off‑Highway Vehicles — Evaluation & Test Report

Prepared By: N.Sukeerthi, N.Ganesh and A.Kailash

---

## Table of Contents
1. Executive Summary
2. Background & Problem Statement
3. Objectives and Expected Outputs
4. System Overview (Architecture & Components)
5. Sensor Suite & BOM Tiers
6. Fusion, Integrity & Graceful Degradation
7. Evaluation Criteria (Core Metrics)
8. Verification & Test Scenarios
9. Test Matrix
10. Test Procedures (Step-by-step)
11. Data Logging Schema (Raw Data Fields)
12. Metric Definitions & Computation Methods
13. Fault Injection & Safety Controls
14. Pass / Fail Logic and Acceptance Criteria
15. Long-duration / Endurance Test Plan
16. Risk Register (Prioritized)
17. Pilot Report Template (deliverable structure)
18. Deliverables & Next Steps
19. Appendix: Helpful Formulas and Conventions

---

## 1. Executive Summary
Goal: Deliver high-availability, high-integrity localization with bounded error and graceful degradation for off-highway vehicles operating in GNSS-challenged environments (urban canyons, dense foliage, mines, construction sites).

Strategy: Heterogeneous multi-sensor fusion (tight GNSS+INS + LiDAR/VIO/odometry + map-aiding), layered estimators (real-time EKF + sliding-window smoother), active integrity monitoring (innovation tests, RAIM-like checks), multi-hypothesis fault handling, and tiered BOM for cost/availability trade-offs. Evaluation focuses on precise core metrics, repeatable verification scenarios, and a rigorous acceptance plan.

---

## 2. Background & Problem Statement
Off-highway vehicles require consistent, bounded, and integrity-assured localization for path following, safety, geofencing, asset tracking, and autonomy. GNSS visibility and quality degrade in many operational environments (urban canyons, dense forestry, steep pits), causing multipath, attenuation, intermittent fixes, and sometimes full GNSS denial. Relying on GNSS-only localization is insufficient for availability, safety, and precise tasks. Solution must avoid over-reliance on external infrastructure, operate with intermittent communications, and gracefully degrade.

---

## 3. Objectives and Expected Outputs
Deliver a validated system and test artifacts that include:
- System Architecture Diagram with sensor suite, fusion pipeline, and integrity monitoring.
- Evaluation Plan & Test Matrix covering scenarios, metrics, thresholds, and pass/fail criteria.
- Demo Routes & Test Scripts (urban canyon, forestry, mining).
- Risk Register with mitigations and prioritized actions.
- Pilot Report summarizing performance vs baselines (GNSS-only, INS-only, VIO-only), raw logs, and recommendations.

User-specified core metric targets are used as acceptance criteria (see Section 7).

---

## 4. System Overview (High-Level Architecture)
- Sensors: Multi-band GNSS (L1/L2/L5 where available), IMU (high-rate MEMS or tactical-grade), wheel encoders/CAN odometry, LiDAR (solid-state or spinning), cameras (global-shutter), optional UWB/pseudolite.
- Time sync: 1PPS + PTP (IEEE 1588) recommended; GNSS receiver providing PPS/TOW.
- Compute: Industrial edge (e.g., NVIDIA Jetson family or Intel NUC) with thermal design and fallback modes.
- Estimators:
  - Real-time tightly-coupled GNSS+INS EKF for control-rate pose + covariance.
  - Loosely-coupled VIO and LiDAR odometry subsystems (with independent health flags).
  - Sliding-window factor graph (GTSAM/Ceres) for smoothing & map updates.
- Integrity: Innovation monitoring, RAIM-equivalent checks, residual-based FDI, multi-hypothesis evaluation, and health-state machine (GREEN/YELLOW/RED).
- Maps & corrections: Local map store, periodic uplink of maps, NTRIP RTK where available, deferred correction support.

Key outputs published: pose (x,y,z), orientation, covariance, HPL (or analogous protection level), health flags, solution mode (RTK/Float/Standalone), time-since-last-fix.

---

## 5. Sensor Suite & BOM Tiers (Summary)

Core (minimum viable) sensor set:
- Multi-band GNSS receiver (RTK-capable)
- IMU (≥200 Hz)
- Wheel odometry via CAN
- One global-shutter camera
- Solid-state LiDAR (optional depending on budget)
- Edge compute (industrial CPU + optional GPU)

Recommended BOM tiers:
- Tier C (Cost-conscious): consumer RTK-capable GNSS, high-performance MEMS IMU, wheel encoders, single camera, 16-ch LiDAR or none, Jetson Xavier NX / NUC i7.
- Tier M (Balanced): multi-band RTK, tactical MEMS IMU, 32–64 ch LiDAR, stereo camera, Orin NX/AGX.
- Tier H (High performance): multi-band GNSS + pseudolites/UWB, tactical/FOG IMU, 64+ ch LiDAR, multi-camera array, Orin AGX, redundant GNSS antennas.

Mounting & environmental considerations: IP67 enclosures, wipers/heaters, vibration isolation for IMU, EMI shielding, antenna separation where possible.

---

## 6. Fusion, Integrity & Graceful Degradation

Estimator Stack:
- Tightly-coupled EKF: IMU raw, GNSS pseudorange & carrier, wheel odometry, LiDAR/VIO pose priors -> real-time pose + covariance.
- Sliding-window factor graph: incorporates LiDAR scan-to-map, visual matches, GNSS constraints for smoothing and map updates.
- Redundant & independent estimators: VIO and LiDAR odometry run separately to provide cross-checking and alternate hypotheses.

Integrity Monitoring:
- Innovation-based NIS (normalized innovation squared) per sensor channel.
- RAIM-style checks for GNSS (position-domain residuals, carrier-phase ambiguity checks).
- Residual-based FDI (GLRT / chi-squared thresholds) to isolate failing sensors and reject them automatically.
- Publish HPL or analogous protection radius derived from covariance + integrity scaling.
- Health-state machine with explicit operational modes and associated autonomy envelopes:
  - GREEN: full autonomy.
  - YELLOW: degraded (speed limits, reduced autonomy functions).
  - RED: localization insufficient — safe-stop or operator control.

Graceful Degradation Strategies:
- On GNSS loss: increase GNSS covariance, rely on LiDAR/VIO + wheel odometry, enable map matching to bound drift, increase IMU process noise gradually to account for bias growth.
- On LiDAR/camera degradation: fall back to GNSS+INS with increased conservatism.
- On RTK correction loss: switch from RTK-Fixed -> RTK-Float/PPP and increase HPL accordingly.

---

## 7. Evaluation Criteria (Core Metrics) — As Provided

Position Accuracy
- Target: ≤ 0.5–1.0 m CEP95 for navigation; ≤ 10 cm for precision tasks.

Heading Accuracy
- Target: ≤ 0.5–1.0° under dynamics; ≤ 2° at low speed.

Availability / Uptime
- ≥ 99% over 8-hour operation in GNSS-challenged zones.

Continuity Under GNSS Outage
- Drift ≤ 1% of distance travelled OR ≤ 1 m/min in typical operating conditions.

Latency & Update Rate
- < 100 ms latency; ≥ 10–50 Hz pose updates depending on control loop needs.

Robustness to Environment
- Operates in multipath, foliage, dust, rain, vibration; maintains performance after shocks.

Energy & Compute Efficiency
- Fits within on-vehicle compute (ARM/NVIDIA edge); power budget ≤ 30–60 W for localization stack.

Calibration & Maintenance Overhead
- Minimal field calibration; auto-recalibration within < 5 minutes after sensor replacement.

---

## 8. Verification & Test Scenarios (Overview)
- Urban canyon route with GNSS shadowing (buildings, tunnels).
- Dense canopy route (forestry) with variable speeds and intermittent stops.
- Mining site with high dust and dynamic obstacles.
- Static cold start and hot start behavior.
- Deliberate sensor faults (IMU bias spikes, wheel slip, camera blinding).
- Long-duration runs (≥ 8 hours) for drift and availability benchmarking.

Each scenario should be run multiple times (recommended 3–5 passes) and include some controlled fault-injection events and GNSS outage windows.

---

## 9. Test Matrix (Example)

Scenarios, Speed buckets, GT method, Reps, Primary checks

- Open-sky baseline
  - Speeds: 0–5 / 5–15 / >15 kph
  - GT: RTK surveyed
  - Reps: 3
  - Targets: CEP95 ≤ 0.5 m, Availability ≥ 99.9%

- Urban canyon
  - Speeds: 0–5 / 5–15 kph
  - GT: RTK + total station sections
  - Reps: 5
  - Targets: CEP95 ≤ 1–2 m, HPL ≤ 5 m, GNSS outage recovery

- Dense canopy (forest)
  - Speeds: 0–5 / 5–15 kph with stops
  - GT: reference INS/RTK or LiDAR tie-in
  - Reps: 5
  - Targets: CEP95 ≤ 2–3 m, drift ≤ 1% distance, availability ≥ 99% over 8h

- Mining (dust)
  - Speeds: 0–10 kph
  - GT: RTK where possible; reference INS
  - Reps: 5
  - Targets: CEP95 ≤ 1–2 m, operational robustness in occlusions

- Tunnel / GNSS denial
  - Speeds: 0–10 kph
  - GT: reference INS
  - Reps: 3
  - Targets: drift ≤ 1 m/min; recovery to <1.5 m within 10 s after GNSS restoration

- Static cold/hot starts
  - GT: RTK
  - Reps: multiple
  - Targets: time-to-first-fix and convergence to <1 m

- Fault injection set
  - IMU bias spikes, wheel slip, camera occlusion/blinding, LiDAR occlusion, RTK loss
  - Observe detection, isolation, and recovery.

- Long-duration endurance
  - Continuous 8+ hour run
  - Targets: Availability ≥ 99% for 8h; thermal/computational stability; controlled drift.

---

## 10. Test Procedures (Step-by-step)

Pre-test (30–60 min)
- Verify time synchronization (PPS/PTP); log offset checks.
- Start RTK base and verify rover lock.
- Inspect sensors: camera lens, LiDAR dome, antenna mounting, IMU mount, battery/power conditioner.
- Start data recorder (bag or CSV) and CPU/power logger; record run ID and operator names.

Run execution
- Drive predefined route, following speed buckets and maneuvers.
- At planned GNSS-obscuring points, mark timestamps (event_log).
- Introduce controlled GNSS outages: antenna covered or RF-blocking for defined durations (10 s, 30 s, 120 s).
- Inject controlled faults in a safe, supervised area (see Section 13).
- Keep a manual event log with timestamps matching recorder timebase.

Post-run
- Stop recorders and collect files; checkout system logs and power readings.
- Note environmental conditions and any manual events.

Repeat per scenario as per test matrix.

---

## 11. Data Logging Schema (Raw Data Fields)
Minimum CSV fields to capture (all timestamps must be in a common timebase or include synchronizable stamps):

Estimator output (est.csv)
- timestamp (ISO8601 or epoch seconds, hardware stamped)
- x, y, z (meters in local ENU)
- yaw, pitch, roll (degrees or radians; be consistent)
- covariance (at least horizontal σ_x, σ_y, or full 3x3 entries)
- hpl (horizontal protection level) or analogous field (optional but recommended)
- health_flag (GREEN/YELLOW/RED)
- solution_mode (RTK-FIX, RTK-FLOAT, SBAS, Standalone)
- age_of_fix_ms
- latency_ms (time from sensor reference to published solution)
- source_notes (e.g., fused, INS-only)

Ground truth (gt.csv)
- timestamp
- x, y, z (meters, same frame as estimator)
- yaw (heading deg)
- gt_method (RTK/TotalStation/RefINS)

Diagnostics (diagnostics.csv)
- timestamp
- gnss_fix (bool)
- gnss_snr (per-satellite aggregated or mean)
- imu_status (OK/DEGRADED)
- camera_status (OK/DEGRADED)
- lidar_status (OK/DEGRADED)
- event (string)

Power & system stats (power.csv, system_stats.csv)
- timestamp
- voltage_v, current_a, power_w
- cpu_percent, gpu_percent, temp_c, throttling_events

Event log (events.csv)
- timestamp
- event_id
- event_type (GNSS_OUTAGE, IMU_FAULT, CAMERA_COVER, OPERATOR_NOTE)
- description

All CSVs should include run_id and vehicle_id columns.

---

## 12. Metric Definitions & Computation Methods

Time alignment
- Use hardware stamps or correct offsets via recorded PPS events. Convert timestamps to seconds (float) for alignment.

Position error
- e(t) = sqrt((x_est − x_gt)^2 + (y_est − y_gt)^2)

CEP95
- 95th percentile of the radial horizontal error distribution. Compute on the set {e_i} as percentile(e, 95).

RMSE
- sqrt(mean(e_i^2))

Heading error
- h_err(t) = minimal absolute angular difference between estimated heading and GT heading (degrees). Normalize differences to ±180°.

Availability
- Define solution_valid if: health_flag == GREEN AND HPL <= threshold (site threshold) AND age_of_fix <= max_age_allowed.
- availability = total_time_valid / total_run_time * 100%

Drift under GNSS outage
- Detect outage windows from diagnostics (GNSS fix loss or SNR below threshold).
- For each window starting at t0, compute drift_60 = e(t0 + 60 s) − e(t0) (or e(t0+60s) if e(t0) ~ 0).
- drift_percent = drift_60 / distance_traveled_during_60s * 100%
- Accept if drift_60 ≤ 1 m AND drift_percent ≤ 1%.

Latency
- latency_i = t_solution_published − t_sensor_reference.
- Report p50/p90/p99; pass if p99 < 100 ms.

Resource constraints
- Average power draw ≤ budget (30–60 W as specified).
- Monitor CPU/GPU usage and thermal throttling events.

Calibration & maintenance
- Auto-recalibration time measured from swap/restart to achieve 'calibrated' state. Target < 5 min.

---

## 13. Fault Injection & Safety Controls

Controlled fault injections (conduct in secure area, with safety officer):
- GNSS outage: physically cover antenna or use RF shielded container for durations: 10s, 30s, 120s.
- IMU bias spike: apply software-injected bias to IMU stream for short duration (or perform sudden motion safely).
- Wheel slip: perform a controlled slip maneuver or simulate reduced encoder accuracy.
- Camera blinding: cover lens or shine bright light (safely) for short durations.
- LiDAR occlusion: cover dome or place tarp for short interval.
- RTK loss: disable NTRIP client for defined interval.

Safety controls:
- Always reduce vehicle speed for fault injection.
- Operator with kill switch and vehicle controller overrides must be present.
- Annotate events precisely into event_log with timestamps.

---

## 14. Pass / Fail Logic and Acceptance Criteria

Per-run pass conditions (must satisfy all):
- CEP95 ≤ scenario target (see Test Matrix).
- Heading 95th percentile ≤ scenario target.
- Availability per run ≥ scenario threshold.
- Drift checks for GNSS outages pass (drift_60 ≤ 1 m and drift_percent ≤ 1%).
- Latency p99 < 100 ms and steady update rate within ±10% of nominal.

Pilot acceptance (over campaign):
- ≥ 90% of runs pass per-run criteria.
- Long-duration run (≥ 8 hours) availability ≥ 99%.
- No unmitigated safety-critical Hazardously Misleading Information (HMI) events.

Precision tasks (≤ 0.10 m):
- Dedicated precision tests with total station GT; require CEP95 ≤ 0.10 m for pass.

---

## 15. Long-duration / Endurance Test Plan

- One continuous 8+ hour run per vehicle, with an operational mix resembling typical duty cycles.
- Objectives:
  - Measure overall availability and continuity.
  - Identify thermal throttling, compute overload, and storage constraints.
  - Periodic RTK spot-checks (every 1 hour) at surveyed control points to detect slow drift.
- Instrumentation:
  - Continuous power logging.
  - CPU/GPU usage and temperature.
  - Environmental logs (temperature, dust indicator).
- Acceptance: availability ≥ 99% over full run; no more than a small number (configurable) of thermal throttle events.

---

## 16. Risk Register (Prioritized)

1. GNSS multipath / outage (Likelihood: High, Impact: High)  
   Mitigation: tightly-coupled GNSS+INS, LiDAR/VIO fallback, local pseudolites/UWB in constrained sites, RAIM & innovation monitoring.

2. Map staleness (Likelihood: High, Impact: High)  
   Mitigation: map versioning, incremental SLAM updates, conservative covariances for map matching, frequent resurvey.

3. Sensor contamination (dust/mud) (Likelihood: High, Impact: Medium)  
   Mitigation: wipers/heaters/shrouds, automated sensor health checks, degrade to GNSS+INS when necessary.

4. Power ripple / EMI (Likelihood: Medium, Impact: High)  
   Mitigation: power conditioning, shielding, EMI testing, antenna separation.

5. IMU drift & thermal variation (Likelihood: Medium, Impact: High)  
   Mitigation: factory & field calibration routines, temperature compensation, bias estimation in filter.

6. Regulatory constraints for pseudolites/UWB (Likelihood: Medium, Impact: Medium)  
   Mitigation: early regulator engagement, use only approved frequency bands, fallback localization.

7. Thermal throttling / compute overload (Likelihood: Medium, Impact: Medium)  
   Mitigation: thermal design, staged estimator modes, watchdogs and resource monitoring.

8. Lack of ground-truth availability under canopy (Likelihood: High, Impact: Medium)  
   Mitigation: temporary RTK base stations; instrumented reference vehicles; LiDAR tie-ins.

---

## 17. Pilot Report Template (Deliverable Structure)

Title page
- Project name, vehicle ID, run ID, dates, authors, summary.

Executive summary
- High-level results: availability, CEP95, drift, key incidents, recommendation.

Test conditions & dataset
- Routes, start/end times, weather, vehicle configuration, BOM.

Metrics summary table
- CEP95, RMSE, 95% heading, availability, drift metrics, latency, power.

Comparative baseline table
- GNSS-only | INS-only | VIO-only | Fused solution (sample format included below)

Visuals
- XY overlays (GT vs solutions) for each scenario.
- Error CDFs and time-series plots.
- HPL or availability heatmap along route.
- CPU/GPU/Power & Temperature graphs.

Events & fault summary
- Table of injected faults and system response timestamps.

Forensics
- Annotated logs, root-cause analysis of failures.

Recommendations
- BOM adjustments, algorithmic tuning, operational constraints.

Appendix
- Raw metrics CSV references, rosbag IDs, calibration logs, system configs.

Sample comparative table (example)
| Solution | Horizontal RMSE (m) | 95% CEP (m) | Availability (%) | Max drift in 60s GNSS outage (m) |
|---|---:|---:|---:|---:|
| GNSS-only | 4.2 | 6.1 | 86% | N/A |
| INS-only | 8.7 | 12.4 | 100% (propagating) | 8.7 |
| VIO-only | 2.9 | 4.3 | 92% | 3.4 |
| Fused (proposed) | 1.1 | 1.6 | 98.5% | 1.2 |

(These illustrative values should be replaced with measured run outputs.)

---

## 18. Deliverables & Next Steps (Recommended)
Short-term:
- Choose BOM tier and identify 1–3 vehicles for pilot.
- Procure sensors & mounting gear (include environmental protection).
- Implement PPS/PTP time sync on chosen compute hardware and validate timestamps.
- Integrate a core tightly-coupled GNSS+INS EKF and one SLAM pipeline (LIO-SAM or VINS variant).
- Run initial open-sky tests and produce first pilot report.

Medium-term:
- Execute full test matrix including fault injections and long-duration runs.
- Iterate estimator tuning and integrity thresholds.
- Implement map freshness processes and automated map checks.

Deliverables expected from pilot:
- Full PDF pilot report with raw metrics and plots (structure above).
- Rosbag/CSV dataset IDs and storage locations.
- Calibration & commissioning logs.
- Risk mitigation action list and timelines.

---

## 19. Appendix: Helpful Formulas & Conventions

Time conversion
- ISO8601 -> epoch seconds: use consistent timezone (UTC recommended).

CEP95
- CEP95 = percentile(radius_errors, 95)

RMSE
- RMSE = sqrt(mean(error^2))

Heading minimal difference
- diff = atan2(sin(a−b), cos(a−b)) (in radians), convert to degrees for reporting.

Drift over 60 s
- drift_60 = error(t0 + 60s) − error(t0)
- drift_percent = drift_60 / distance_traveled_between_t0_and_t0+60s * 100%

Availability
- availability_pct = 100 * (samples_where_solution_valid / total_samples)

HPL (protection level)
- If not calculated by estimator, approximate HPL = k * sqrt(σ_x^2 + σ_y^2) where k is an integrity multiplier chosen conservatively (user must calibrate k per site requirements and safety analysis). This is NOT a certified RAIM HPL algorithm; for certified systems, follow formal GNSS integrity standard methods.

Coordinate frames
- Use local ENU (meters) with a known origin; provide conversion notes when using lat/lon GT.

