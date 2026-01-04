![Flow Diagram](https://github.com/user-attachments/assets/9cb9014a-e21c-41c5-b8ff-f5307fe11bf1)

# Assumptions (explicit)

**Regional ride-hailing context:** unreliable mobile networks, intermittent GPS, variable device capabilities, and occasional server-to-device latency.

**System components:** mobile apps (driver/agent app, customer app), backend services (Ride Orchestrator, Location Service, Matching Service, Trip Service, Notification Service, Audit/Storage), persistent datastore, message bus (event streaming), and push + socket channels.

**Communication channels:** persistent socket (WebSocket/QUIC) where available, otherwise push notifications and periodic HTTP polling. Use exponential backoff and store-and-forward on devices.

**Data consistency model:** eventual consistency for ephemeral trip updates; strong consistency for authoritative state transitions (Ride Orchestrator as single source-of-truth with transactional semantics or optimistic concurrency with versioning).

**Timestamps** use monotonic server time for ordering of events where possible; client timestamps are used but validated/normalized by server.

**Security & privacy:** location transmitted with authentication and encryption. Geo-fencing limits and TTL applied to stored location data.

---

## Part 1 — Ride Lifecycle & State Machine

### Canonical states
*   Requested
*   Searching for agent
*   Agent Assigned
*   Agent Arrived
*   Trip Started
*   Trip Completed
*   Cancelled (Customer / Agent / System)

### State machine semantics (single authoritative orchestrator)
*   The Ride Orchestrator service is authoritative. Every transition must be accepted by the orchestrator which enforces allowed transitions, idempotency, and concurrency control (ride version number / optimistic lock).
*   Events coming from driver app, rider app, or automated rules are w/ timestamps and incremental `requestId` to support dedup and ordering.
*   Transitions produce audit events; state persisted with `lastUpdated`, `actor`, `reason`, and `causationId`.

### For each state

#### Requested
*   **Who can trigger:** Triggered by Customer (user app) creating a ride request.
*   **Allowed transitions:**
    *   To **Searching for agent** (automatic after request accepted)
    *   To **Cancelled (Customer)** — immediate cancel before matching.
*   **Disallowed transitions:** Directly to Agent Assigned / Trip Started / Agent Arrived / Trip Completed.
*   **Backend responsibilities:** Persist ride request, validate payment/eligibility, compute routing and ETA, enqueue request into Matching Service, publish `RequestCreated` event, notify candidate agents (via Location/Matching service), set request TTL and retry policy.

#### Searching for agent
*   **Who can trigger:** Entered by system after request creation. Driver responses can trigger assignment attempts.
*   **Allowed transitions:**
    *   To **Agent Assigned** (on successful accept)
    *   To **Cancelled** (Customer or System — e.g., no agents found, TTL expired)
*   **Disallowed transitions:** To Trip Started / Agent Arrived / Trip Completed directly.
*   **Backend responsibilities:** Maintain candidate list, track offers/outreach, support multi-stage offers (batch or sequential), enforce timeouts and retry, update rider with ETA/agent search progress, record metrics, and escalate to system cancellation after configured retry/time limits.

#### Agent Assigned
*   **Who can trigger:** System after agent accepts or is assigned (driver app accept or auto-assign).
*   **Allowed transitions:**
    *   To **Agent Arrived** (on agent arrival)
    *   To **Trip Started** (if start is allowed without explicit arrived state by rule — but better to require arrived)
    *   To **Cancelled** (Customer / Agent / System)
*   **Disallowed transitions:** To Trip Completed directly unless Trip Started.
*   **Backend responsibilities:** Lock the agent for this ride (prevent double-booking), send assignment notification to agent and rider, open tracking channels, create Trip record, start tracking TTL timers (e.g., agent must acknowledge within N sec), capture agent acknowledgement event, and store initial last-known location.

#### Agent Arrived
*   **Who can trigger:** Agent app (driver toggles arrived) or system inferred (geo-fence arrival) with validation.
*   **Allowed transitions:**
    *   To **Trip Started** (when rider boards or agent starts trip)
    *   To **Cancelled** (Customer / Agent / System)
*   **Disallowed transitions:** To Searching for agent or Agent Assigned (backwards) except via cancellation and new request.
*   **Backend responsibilities:** Validate arrival (e.g., verify location within arrival radius or require manual confirmation), timestamp arrival, notify rider, begin short waiting-period timers (free-wait threshold), log arrival location and ETA timers, enable fine-grained tracking (faster update cadence), and persist arrival audit.

#### Trip Started
*   **Who can trigger:** Agent (driver) or Orchestrator (if automatic start based on geofence or rider confirmation).
*   **Allowed transitions:**
    *   To **Trip Completed**
    *   To **Cancelled** (Agent / System) — Customer cancels generally not allowed once trip started, but may request emergency stop (policy).
*   **Disallowed transitions:** To Agent Arrived (backwards) except via cancellation/reopen (rare).
*   **Backend responsibilities:** Authoritatively mark trip start time, create billing sessions, start high-frequency tracking recording, stream location to fare calculation, set route optimization, enforce safety checks, escalate alerts on anomalies, and allocate resources for telemetry retention.

#### Trip Completed
*   **Who can trigger:** Agent (driver marks complete), Orchestrator (auto-complete by geofence or after agent-provided proof), or System (e.g., forced completion after inactivity thresholds).
*   **Allowed transitions:** Terminal (no further transitions except archival or re-open by admin/system)
*   **Disallowed transitions:** To Trip Started, Agent Arrived, etc.
*   **Backend responsibilities:** Record `endTime`, final fare calculation, persist trip path and final location, trigger receipts & post-trip flows, move to archival, release agent from lock, generate analytics events, and mark for billing/settlement.

#### Cancelled (Customer / Agent / System)
*   **Who can trigger:** Customer (rider), Agent (driver), or System (timeouts, safety, matching failures).
*   **Allowed transitions:** Terminal (no transitions further).
*   **Disallowed transitions:** Re-enter active states except via new ride creation.
*   **Backend responsibilities:** Record who cancelled, reason code, timestamps, cancellation fees if any, persist last-known locations for both parties, notify counterparties, release agent lock, compute compensation or penalties, store audit trail, and optionally keep ride record in admin with cancellation metadata.

### State transition rules & concurrency
*   **Optimistic locking:** Use `rideVersion` or a dedicated transaction mechanism in Orchestrator to avoid races (e.g., two drivers accept). The orchestrator must reject stale transitions with clear error codes; clients should handle retries and inform users.
*   **Idempotency:** All external events must carry `requestId` for dedup; Orchestrator must ignore duplicates.
*   **Validation:** Each transition validated by actor permissions (e.g., only assigned agent can mark arrived or start).

---

## Part 2 — Real-Time Location Tracking Logic

### Design goals
*   Resilient to spotty networks and GPS noise.
*   Low-latency for live Tracking but bounded data volume.
*   Safe fallback to last-known location when live is unavailable.
*   Strong audit trail with last valid fix, fix quality, and source.

### General location data model
*   **Each location update:** `{rideId, actorType (agent), actorId, lat, lon, timestamp_client, timestamp_server_received, accuracy_meters, provider (gps/wifi/cell), speed, heading, seqNumber, batteryStatus, networkType}`
*   **Server stores:** `lastValidLocation`, `lastSeenTime`, `qualityMetrics`, and a time-series buffer for the trip (with retention policy).
*   **Location Quality scoring:** derived from accuracy, speed, jump detection, and provider.

### Location update flow by phases

#### A) agent goes online (before any ride)
*   **Device behavior:**
    *   When agent opens app and goes Online, send a low-frequency "presence" ping immediately (e.g., every 30s–2min depending on energy vs availability).
    *   Use geohash/grid registration with Location Service for matching queries.
    *   If intermittent connectivity, device caches last-known location and sends on reconnection.
*   **Backend behavior:**
    *   Register agent as available in Matching Service with last-known coordinates and timestamp and availability flag.
    *   Emit presence events to Matching Service; do not store high-frequency traces when not on a trip (privacy/cost).
    *   Serve location queries for candidate matching based on last-seen location + drift heuristics.

#### B) After accepting a ride (agent Assigned / en route to pickup)
*   **Device behavior:**
    *   Increase update cadence (e.g., every 3–10s if moving, else 10–30s). Use adaptive cadence: higher when speed > threshold; lower when stationary.
    *   Annotate updates with `tripId` and `seqNumber`. Prioritize sending via persistent socket; fallback to HTTP post or queued background transmissions if socket unavailable.
    *   Include accuracy and battery; if GPS disabled, include coarse locations and mark accuracy high (low fidelity).
    *   If offline, buffer updates locally with monotonic `seqNumber` and transmit on reconnect.
*   **Backend behavior:**
    *   Location Service consumes updates, validates sequence numbers and timestamps, applies filtering (debounce, low-pass for small jitter), runs jump detection and smoothing, updates agent status and ETA estimate to rider.
    *   Emit position updates to rider app via socket; if offline, send push with summary ETA and last-known location.
    *   Persist updates to a time-series store with retention policy (high-frequency retention shorter, aggregated longer).

#### C) During pickup (arrived -> boarding)
*   **Device behavior:**
    *   Very high priority for updates; send immediate arrival ping upon toggling Arrived.
    *   If driver claims arrival but GPS is ambiguous, device can send “arrived-confirmation” plus short video/photo or proximity confirmation (e.g., BLE, QR, handshake) where supported.
*   **Backend behavior:**
    *   Validate arrival with geo-fence or cross-check sequence of locations; start waiting timer and notify rider.
    *   Prepare fare/session for immediate Trip Started transition.

#### D) During trip
*   **Device behavior:**
    *   Maintain high-frequency updates (every 1–5s if possible) and reliable transmit (`seqNumbers`, acks). Compress/aggregate when network constrained but keep last-known path fidelity.
    *   If network fails, locally cache updates and send on restore with accurate client timestamps and `seqNumbers`.
    *   Optionally send periodic heartbeat even if no movement, to indicate device alive.
*   **Backend behavior:**
    *   Location Service stores stream, calculates distance and fare in near-real-time, and validates anomalies.
    *   Stream location to rider app and to analytics and safety monitoring (e.g., deviation from route, prolonged stop).
    *   Persist authoritative `lastValidLocation` per trip and the path.

#### E) After trip completion
*   **Device behavior:**
    *   Send final stop/location and trip-complete event. Flush any buffered location history.
    *   Resume low-frequency presence pings or go offline.
*   **Backend behavior:**
    *   Finalize trip path, store full trace as trip artifact, aggregate trip metrics, trigger billing and receipts, release driver lock, and enforce retention/archival.

### How location updates are sent and consumed
*   **Primary channel:** Persistent socket (WebSocket over TLS or QUIC) with application-level acknowledgements and `seqNumbers`. Sockets should support automatic reconnect with exponential backoff.
*   **Secondary channels:** HTTPS POST to location endpoint (batched or single update) when socket unavailable; push notification as last resort for critical messages to wake the device.
*   **Consume flow:**
    *   Location Service receives updates, validates (`seqNumber`, timestamp), applies quality checks, writes to in-memory cache for quick reads (e.g., Redis), forwards to matching and trip services via event bus, and persists to durable store (time-series DB or append-only blob store).
    *   Rider apps subscribe to agent location updates over socket with throttling (server-side) to avoid flooding; server can downsample for clients with low bandwidth.
*   **Acknowledgements:** Server acks messages; device retries un-acked messages according to policy; dedupe by `seqNumber`.

### Handling last known location if live updates stop
*   **Define last-seen policy:** `lastValidLocation` with `lastSeenTimestamp` and quality flag. For UI/consumers, expose a TTL threshold: e.g., "fresh" if <15s, "stale" if 15–120s, "offline" if >120s (configurable by region/network).
*   **If live updates stop:**
    *   Rider UI and backend should show agent as "connection lost" but display last-known position and estimated drift radius (based on last speed and time elapsed).
    *   Orchestrator enforces safety timers: if agent offline during Assigned or En-route longer than threshold (e.g., no ack within N sec), trigger agent-check flow: ping device, send SMS or call fallback, and optionally reassign if unresponsive beyond policy window.
    *   For active trips, if agent network down but device has cached updates that are later uploaded, server reconciles by sequence numbers and client timestamps. Server marks gaps and flags possible GPS tampering if inconsistent.
*   **Server-side extrapolation:** Optionally extrapolate a probable position using last velocity/heading for short gaps (with low confidence). Do not use extrapolated position for critical actions (e.g., billing settlement, master state change) without confirmation.

---

## Part 3 — Failure & Edge-Case Handling

**Design goal:** maintain safety, fairness, and data consistency while minimizing false positives and churn.

### agent loses internet during an active ride
*   **Detection:** Socket disconnect + no heartbeat; `lastSeenTimestamp` older than threshold triggers offline flag.
*   **Immediate system behavior:**
    *   Keep trip state as **Trip Started**. Continue to accept buffered updates when they arrive.
    *   Mark agent as temporarily offline; notify rider: "Driver temporarily offline — showing last known position." Include ETA confidence reduced.
    *   Start reconnection window timer (configurable, e.g., 30s–180s depending on region and local expectations).
*   **Safety & operations:**
    *   If driver offline but trip is ongoing, rider can call agent (via masked number), or contact support. Backend may automatically place recorded route portion into "pending reconciliation" until device uploads buffered data.
    *   If agent remains offline beyond reconnection window and there’s evidence of abandonment (no movement, no buffered uploads on reconnect), Orchestrator can trigger contingency: attempt contact, request agent to call, and if unresolved, escalate to admin and consider system cancellation/auto-complete with compensation rules.
*   **Data consistency:**
    *   When agent rejoins, device uploads buffered events with `seqNumbers` and original timestamps; server reconciles ordered-by-seq, validates for jumps, and fills gaps.
    *   If server receives events with timestamps that conflict (e.g., client clock skew), rely on server `receiveTime` ordering and `seqNumbers`.

### GPS data freezes or shows sudden jumps
*   **Detection:**
    *   Location Service runs anomaly detection: zero movement beyond expected given speed vs time (freeze), or sudden large displacement inconsistent with speed/known max possible velocity (jump).
    *   Use `accuracy_meters` and provider to weight trust.
*   **Handling:**
    *   **For freezes:** treat as stale data; extrapolate position using last valid speed/heading for short durations with low confidence, tag path with "extrapolated" flag.
    *   **For jumps:** reject improbable point if it violates physical constraints (e.g., teleportation). Keep previous valid point and request a fresh fix from device. Store rejected points as anomalies for later audit.
    *   Notify rider minimally: "Location accuracy low" rather than noisy warnings.
*   **Impact on billing and trip state:** Use validated path for distance/fare computations; exclude or smooth rejected/jumped points. If anomalies result in ambiguous distance, mark trip for manual review or apply conservative charging rules.
*   **Feedback to agent:** agent app may prompt user to allow high-accuracy GPS or check device settings. Provide a retry mechanism and request agent to pull over if safety-critical.

### Customer app is backgrounded or killed
*   **Detection:** Client-to-server socket closes; no subscriber for location updates, but backend must still continue to send agent updates.
*   **Handling:**
    *   Rider receives push notifications for critical events (agent assigned, arrival, cancellation, trip completed). Backend also keeps a canonical timeline of locations regardless of client.
    *   On re-open, rider app should request a delta/refresh; server returns last-known agent path and current state.
    *   For tracking privacy: if customer backgrounded, do not send continuous location requests to driver or rely on customer location unless rider allowed background location. But agent tracking of driver continues regardless.
*   **Data consistency:** No data loss of trip telemetry; agent telemetry stored server-side. Customer's absence doesn't affect trip lifecycle except potentially inability to confirm arrival quickly; implement fallback confirmation methods (driver photo, OTP).

### Socket disconnects and reconnects
*   **Device behavior:** On disconnect, attempt exponential backoff reconnect. Buffer outgoing location messages with `seqNumbers` and limited buffer size. Provide a periodic but low-overhead heartbeat via push to indicate connectivity problem.
*   **Server behavior:** Mark socket as disconnected but do not change ride state immediately; start a disconnect timer for critical roles. On reconnect with same session credentials, allow resuming with sequence reconciliation. If reconnection occurs on a new IP/session, verify cryptographic credentials and optionally re-authorize.
*   **Reconciliation:** On reconnect, device publishes buffered events with `seqNumbers`; server validates monotonicity and applies any missing intermediate events. If gaps exist in `seqNumbers` beyond buffer, mark missing segments and flag for possible manual review.
*   **TTL and escalation:** After N unsuccessful reconnect attempts or prolonged disconnect beyond policy, trigger fallback flows (SMS, voice call, assign backup agent, or flag admin).

### General consistency & auditing
*   All events are persisted with `causationId` and `actor`; audit logs kept for a retention window for dispute resolution.
*   **Versioning:** `rideVersion` increments at every accepted transition and each authoritative change. If client tries a stale update, server rejects with version error and instructs client to refresh.
*   **Alerts & manual intervention:** Admin dashboards show flagged rides (disconnects > threshold, suspicious GPS, or long idle) for human follow-up.

---

## Part 4 — Cancellation + Tracking Interaction

Two scenarios with required behavior: maintain robust tracking/audit while ensuring consistent records and clear admin visibility.

### Scenario A: Customer cancels while agent is moving to pickup

**Flow:**
*   Customer initiates cancellation; Orchestrator validates cancellation policy (timing, free/fee, promotions).
*   Orchestrator atomically attempts transition: **agent Assigned -> Cancelled (Customer)**. If agent has already toggled Arrived or Trip Started, follow those state rules (arrival may allow cancellation with different fee).
*   Backend notifies agent of cancellation immediately (socket + push + SMS fallback).
*   If agent was en route and connection lost briefly, cancellation still accepted by Orchestrator as long as transition validation passes (versioning ensures no conflicting Trip Started concurrently).

**What happens to tracking:**
*   Real-time tracking for that ride is stopped for rider updates: server stops forwarding agent location updates to the cancelled rider.
*   For the agent, the app still continues to send location (presence) but the ride-specific high-frequency tracking stream is terminated and any buffered ride-specific locations are tagged as post-cancellation and stored separately.
*   Server continues to store last-known location and recent trace (configurable retention for cancelled rides, kept for dispute/safety).

**What data is stored:**
*   **Cancellation record:** who cancelled, timestamp, reason code, policy applied (fee/no-fee), and `causationId`.
*   **Trip audit:** store the last N minutes of location trace (both agent and rider if available), event timeline (assignment time, accepted time, cancel time), messages/notifications sent, and any reconnection/retry logs.
*   **Financial metadata:** if cancellation fee applies, store fee calculation, charge authorization result, and refund/charge events.
*   **Safety logs:** if cancellation occurred during suspicious conditions (e.g., agent offline, rapid GPS jumps), flag and store anomaly details.

**How the ride appears in admin view:**
*   Admin sees ride as **Cancelled (Customer)** with metadata: assigned agent, times (request, assign, cancel), last-known locations with timestamps, and a small map showing the final known agent position and recent path segment.
*   Admin can view raw audit trail and set filters (e.g., cancellations while en route). If flagged, ride appears in a "review needed" queue.

### Scenario B: agent cancels during pickup (agent cancels after being assigned or after toggling Arrived)

**Flow:**
*   agent requests cancel (driver app) or is auto-cancelled by system (e.g., fails arrival ack, safety stop).
*   Orchestrator validates allowed transition: **agent Assigned/agent Arrived -> Cancelled (agent)** and enforces any agent-side penalties/policy.
*   System notifies rider immediately; Matching Service may attempt rapid re-dispatch to another agent.
*   If agent was near pickup and rider already boarded (ambiguous), Orchestrator may refuse agent cancellation and escalate to admin or request confirmation (policy-dependent). If rider boarded, better to require Trip Started instead of cancellation.

**What happens to tracking:**
*   The ride-specific tracking channel is terminated for the rider (they are notified and provided with alternatives). agent app stops sending trip-scoped updates; agent may remain online for new assignments with standard presence pings.
*   The server stores the agent's last-known position and the trace leading up to cancellation.

**What data is stored:**
*   Cancellation record with `actor=agent`, reason (driver-initiated, emergency, device offline), timestamp, any penalty or notes.
*   Full recent trace for audit: last-known GPS points, sequence numbers, and connection events (socket disconnects, upload of buffered data).
*   Messages to rider and agent, and any reassign attempts.

**How the ride appears in admin view:**
*   Admin sees **Cancelled (agent)** with agent-supplied reason (if provided), times, last-known location, and any matching logs. If cancellation falls into suspicious patterns (high cancellation rate for this agent), flag for QA or enforcement.

### Cross-cutting: Data retention, privacy, and dispute resolution
*   Retain sufficient telemetry and audit logs for dispute windows (regionally compliant, e.g., 30–90 days), but aggregate/anonymize older trajectories.
*   For privacy, do not surface precise historical agent locations beyond necessary for disputes; redact where policy requires.
*   Expose exportable logs for manual investigations: sequence of events, raw location stream, socket/session logs, and notification history.

### Edge cases and policies tying cancellations & tracking together
*   **Late cancellation by rider after agent arrival:** treat as special—store arrival confirmation evidence (arrival timestamp, last-known location, any photo/OTP), apply arrival cancellation fees as per policy.
*   **agent cancels claiming offline/technical issues:** require evidence where practical (e.g., socket logs showing disconnect, device telemetry). If agent later uploads buffered data showing they were active, flag inconsistency.
*   **Forced cancellation by system due to safety** (e.g., device reports crash, severe GPS anomaly, or fraudulent behavior): produce rich audit bundle for safety team, preserve full recent traces, and prevent auto-reassignment to the same agent until cleared.

---

## Summary — Operational rules & best-practice checklist

*   **Authoritative state:** Orchestrator service enforces transitions with optimistic locking and idempotency.
*   **Location streams:** prioritized channels (socket -> HTTP -> push), seqNumbers, client timestamps, and server-side validation/filtering.
*   **Buffer-and-forward:** client buffers updates during disconnects; server reconciles by `seqNumber` and timestamps, preserving ordering.
*   **Failure windows:** configurable timers for reconnection and reassignment to balance user experience and safety.
*   **Anomaly handling:** detect freezes/jumps and either smooth, reject, or extrapolate with confidence flags; flag ambiguous trips for manual review.
*   **Cancellation policy:** clearly record actor, reason, time, evidence (arrival proof), and recent location trace; stop forwarding tracking to rider post-cancellation while retaining traces for audit.
*   **Admin visibility:** show state, timestamps, last-known locations, and recent trace snippet plus a link to full audit for flagged rides.