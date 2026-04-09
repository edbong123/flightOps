# Product Requirements Document (PRD): FlightOps MVP

## 1. Executive Summary

FlightOps (FO) is the operational source of truth for flight schools. If CA is "[Booking.com](http://booking.com/)" (owning pilot identity, discovery, and booking), FO is "Cloudbeds" (the localized property management system).

Each flight school runs its own instance of FO to manage fleet status, handle ground operations, dispatch aircraft, and track maintenance. FO pushes aircraft availability up to CA and ingests bookings and pilot-reported issues (squawks) from CA.

**MVP Objective:** Enable a flight school to securely log in staff via Clerk, set up their fleet, receive bookings from CA (or create walk-ins locally), dispatch aircraft with strict safety guards, capture flight telemetry (Hobbs/Tach), handle squawks/groundings, and verify pilot checkout status prior to flight.

## 2. System Boundaries & Ownership

A strict boundary exists between what CA handles (discovery/people) and what FO handles (operations/machines). CA never knows *why* an aircraft is down; it only knows *if* it is bookable.

| Domain | Owner | Description |
| --- | --- | --- |
| **Identity & Bookings** | **CA** | Pilot certs, instructor profiles, endorsements, booking flow, discovery. |
| **Fleet & State** | **FO** | Aircraft details, Hobbs/Tach, registration, operational availability. |
| **Dispatch & Ops** | **FO** | Check-out, check-in, flight closeout, pilot checkout verification. |
| **Maintenance** | **FO** | Squawk review, downtime scheduling, grounding decisions. |

### 2.1 The Integration Surface (Data Flows)

The systems interact via three specific data flows over the Convex backend:

- **FO → CA (Availability Slots):** FO pushes net operational availability per aircraft to CA.
    - *Calculation Logic:* Availability is computed dynamically by taking the school's base operating hours and subtracting all active `AircraftDowntime` records and active `Bookings`.
    - *Hybrid Sync Warning:* If the HTTP push to CA fails after automated retries, FO flags the aircraft's sync status as `FAILED` and displays a critical warning banner to the Dispatcher to prevent silent booking failures.
- **CA → FO (Bookings):** CA syncs confirmed, cancelled, and modified reservations via inbound Convex webhooks. FO uses these to drive the local dispatch workflow.
    - *Note:* Walk-in flights are processed by FO staff creating a standard booking locally on behalf of the student, maintaining a single unified dispatch pipeline.
- **CA → FO (Squawks):** Pilots report discrepancies via CA, which flow into FO as an intake channel. Squawks capture a flat `reporterName` so FO doesn't need a complex pilot identity table.

## 3. MVP Domain Scope

### 3.1 Admin & Setup (The Foundation)

- **Entities:** `Schools`, `Staff` (linked securely via external `clerkId`), `Airports`, `AircraftMakes`, `AircraftModels`, `Aircraft`, `AircraftGroups`, `AircraftGroupMembers`.
- **Staff Roles:** Admin, Dispatcher, Instructor, Mechanic.

### 3.2 Dispatch (The Operational Core)

- **Entities:** `DispatchRecords`
- **Workflow:**
    1. Booking syncs from CA (or is created locally for a walk-in).
    2. Pilot arrives; system verifies pilot is checked-out on the aircraft.
    3. Staff records initial Hobbs/Tach and releases the flight. *Safety Guard: System strictly prevents checking out an aircraft that is already dispatched.*
    4. Pilot returns; staff records ending Hobbs/Tach, fuel added, and logs any squawks. *Safety Guard: Ending meters must be ≥ starting values.*

### 3.3 Maintenance Lite

- **Entities:** `Squawks`, `AircraftDowntime`
- **Workflow:** Squawk lands in FO review queue. Staff decides to ground the aircraft or defer. If grounded, FO creates an `AircraftDowntime` record and automatically triggers the background action to push updated, reduced availability to CA.

### 3.4 Compliance Lite

- **Entities:** `CheckoutRequirements`, `PilotCheckouts`
- **Workflow:** Admins define requirements. Staff (or CA syncs) mark pilots as checked out. System hard-blocks dispatch release if the pilot lacks a valid checkout.

## 4. Out of Scope for MVP

The following features are deferred to post-MVP development:

- **Full Maintenance:** Work orders, AD tracking, 100-hour projections, parts inventory.
- **Full Compliance:** Medical tracking, flight reviews (BFR), currency rules engine, endorsement expirations.
- **Billing:** Invoicing, payments, leasebacks, block accounts (requires finalized business logic).
- **Reporting:** Analytics and dashboards.
- **Training Management:** Syllabi and lesson tracking (handled via separate system/module).

## 5. Build & Execution Strategy

Development should be sequenced to unblock the core operational loop as quickly as possible.

1. **Phase 1: Admin / Setup:** Build the foundation (Schools, Staff, Fleet hierarchy, Clerk Auth).
2. **Phase 2: CA Integration:** Establish the 3 integration flows (receive bookings, receive squawks, push availability). *(Can run parallel to Phase 3).*
3. **Phase 3: Dispatch:** Build the Check-out / Check-in workflow and data capture mechanisms.
4. **Phase 4: Maintenance Lite:** Build the squawk review queue, grounding logic, and downtime management.
5. **Phase 5: Compliance Lite:** Build the pre-dispatch pilot checkout verification. *(Gated by Phase 3).*