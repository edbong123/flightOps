# Flight Ops MVP - User Stories

Here are the user stories for the FlightOps MVP, broken down by the core domains.

## 1. Admin / Setup & Security

- **As a Staff Member**, I want to securely log in using Clerk, so that I can only access my specific flight school's operational data.
- **As a School Admin**, I want to define my school's timezone and standard operating hours, so that aircraft availability is only pushed to CA for times we are actually open.
- **As a School Admin**, I want to create staff profiles and assign them specific roles (Admin, Dispatcher, Instructor, Mechanic), so that employees only have access to the tools they need.
- **As a School Admin**, I want to add individual aircraft to the system with their tail numbers, models, and initial Hobbs/Tach meter readings, so that we have an accurate baseline for our fleet.
- **As a School Admin**, I want to organize aircraft into specific groups (e.g., "Training Fleet", "IFR Capable"), so that CA can display them accurately in group-based booking flows.

## 2. CA Integration & Scheduling

- **As the FlightOps System**, I want to automatically push granular, per-aircraft availability slots to CA whenever our schedule or downtimes change, so that CA always displays an accurate picture of what can be booked.
- **As the FlightOps System**, I want to expose a secure webhook endpoint to receive confirmed, updated, or cancelled bookings from CA in real-time.
- **As a Dispatcher**, I want the system to automatically receive and display confirmed bookings from CA, so that I have a reliable daily schedule of who is arriving to fly.
- **As a Dispatcher**, I want to create a booking directly in the FlightOps system for a "walk-in" student, so they follow the exact same operational flow as a CA-booked student without requiring a separate workflow.
- **As a Mechanic**, I want pilot-reported squawks to sync directly from CA into our local queue, so that I can evaluate discrepancies without having to log into a separate booking platform.

## 3. Dispatch (The Operational Core)

- **As a Dispatcher**, I want to assign a specific aircraft tail number to a booking at the time of dispatch, so that group-level bookings from CA are resolved to a physical plane.
- **As a Dispatcher**, I want the system to hard-block me from checking out an aircraft that is already dispatched, preventing dangerous double-bookings.
- **As a Dispatcher**, I want to record the starting Hobbs and Tach times when checking out an aircraft, so that flight telemetry is perfectly tracked.
- **As a Dispatcher**, I want to record the ending Hobbs, Tach, and any fuel added when a pilot returns, so that the flight record is closed out and ready for future billing.
- **As a Dispatcher**, I want the system to prevent me from entering an ending Hobbs/Tach meter reading that is lower than the starting reading, protecting data integrity.
- **As an Instructor**, I want the permission to dispatch and check in my own flights, so that I am not bottlenecked waiting for front-desk staff during busy hours.

## 4. Maintenance (Lite)

- **As a Mechanic**, I want to review incoming squawks (clearly labeled with the reporter's name) and update their status (Open, In Review, Resolved, Deferred), so that the entire staff knows the current state of reported issues.
- **As a Mechanic**, I want to create a "downtime" record to ground an aircraft, so that the system immediately recalculates availability and removes the plane from CA's bookable slots.
- **As a Dispatcher**, I want a clear visual indicator on the dashboard if an aircraft is grounded, so that I do not accidentally attempt to dispatch an unairworthy plane.
- **As a Dispatcher**, I want to see a critical warning banner if an aircraft's availability fails to sync to CA (Hybrid Sync Warning), so I can manually verify its status and avoid silent booking failures.

## 5. Compliance (Lite)

- **As a School Admin**, I want to define specific checkout requirements (e.g., "C172 Checkout") for our aircraft models, so that we establish clear safety and insurance baselines.
- **As the FlightOps System**, I want to automatically ingest a pilot's valid checkouts from CA during the booking sync, so dispatchers always have up-to-date compliance data.
- **As an Instructor**, I want to manually mark a pilot's profile as "checked out" for a specific aircraft model as a local override, so that the system knows they are cleared to fly it even if CA sync is delayed.
- **As a Dispatcher**, I want the system to hard-block the check-out process if a pilot does not have a valid checkout for the assigned aircraft, so that we never accidentally violate our compliance policies.