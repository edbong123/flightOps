# Flight Ops - Data Model and Convex Schema

Our schema translates the traditional relational model into a reactive, TypeScript-first, document-based architecture using Convex. It includes strict Clerk Auth integration, data integrity constraints, and hybrid sync error handling.

## 1. Domain Entity Map

### Admin / Setup (The Foundation)

| Entity | Core Fields | Relationships / Notes |
| --- | --- | --- |
| **School** | `id`, `name`, `timezone`, `operatingHours`, `settings` | Root tenant. `operatingHours` is an array of day-specific open/close rules. |
| **Staff** | `id`, `schoolId`, `clerkId`, `firstName`, `lastName`, `email`, `role`, `payRate` | `role`: ADMIN, DISPATCHER, INSTRUCTOR, MECHANIC. `clerkId` links to external auth. |
| **Aircraft** | `id`, `schoolId`, `modelId`, `homeAirportId`, `tailNumber`, `currentHobbs`, `currentTach`, `caSyncStatus` | The core machine. `caSyncStatus` tracks CA webhook success/failure. |
| **AircraftGroup** | `id`, `schoolId`, `name` | e.g., "Training Fleet". |

### CA Integration (Bookings Cache)

| Entity | Core Fields | Relationships / Notes |
| --- | --- | --- |
| **Booking** | `id`, `schoolId`, `caBookingId` (Nullable), `caPilotId`, `assignedAircraftId` (Nullable), `startTime`, `endTime`, `status` | Synced from CA. Walk-ins are handled by creating a local booking with a null `caBookingId`. |

### Dispatch (The Operational Core)

| Entity | Core Fields | Relationships / Notes |
| --- | --- | --- |
| **DispatchRecord** | `id`, `schoolId`, `bookingId`, `aircraftId`, `dispatchedByStaffId`, `checkedInByStaffId`, `hobbsStart`, `hobbsEnd`, `tachStart`, `tachEnd`, `status` | `status`: OUT, COMPLETED. Strict mutations prevent meter rollbacks and double-dispatching. |

### Maintenance & Compliance

| Entity | Core Fields | Relationships / Notes |
| --- | --- | --- |
| **Squawk** | `id`, `schoolId`, `aircraftId`, `reporterName`, `description`, `severity`, `status` | `reporterName` is a flat string to seamlessly support both CA Pilots (via webhook) and local FO Staff. |
| **AircraftDowntime** | `id`, `schoolId`, `aircraftId`, `squawkId`, `startTime`, `endTime` (Nullable), `reason` | Null `endTime` means grounded indefinitely. Drives CA availability. |
| **PilotCheckout** | `id`, `schoolId`, `caPilotId`, `requirementId`, `origin`, `expirationDate`, `caSyncStatus` | `origin`: CA_SYNC or LOCAL_OVERRIDE. Tracks if the local override synced back to CA. |

---

## 2. Convex Database Schema (`convex/schema.ts`)

```tsx
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  // ==========================================
  // 1. ADMIN / SETUP
  // ==========================================
  schools: defineTable({
    name: v.string(),
    timezone: v.string(),
    // Strict flexible hours: 0 = Sunday, 1 = Monday, etc.
    operatingHours: v.array(v.object({
      dayOfWeek: v.number(),
      open: v.string(),  // e.g., "08:00"
      close: v.string()  // e.g., "18:00"
    })),
    settings: v.any(),
  }),

  staff: defineTable({
    schoolId: v.id("schools"),
    clerkId: v.string(),
    firstName: v.string(),
    lastName: v.string(),
    email: v.string(),
    role: v.union(
      v.literal("ADMIN"),
      v.literal("DISPATCHER"),
      v.literal("INSTRUCTOR"),
      v.literal("MECHANIC")
    ),
    payRate: v.optional(v.number()),
  })
    .index("by_school", ["schoolId"])
    .index("by_clerk", ["clerkId"]),

  airports: defineTable({
    identifier: v.string(),
    name: v.string(),
  }),

  aircraftMakes: defineTable({
    name: v.string(),
  }),

  aircraftModels: defineTable({
    makeId: v.id("aircraftMakes"),
    name: v.string(),
  }),

  aircraft: defineTable({
    schoolId: v.id("schools"),
    modelId: v.id("aircraftModels"),
    homeAirportId: v.optional(v.id("airports")),
    tailNumber: v.string(),
    currentHobbs: v.number(),
    currentTach: v.number(),
    registration: v.string(),
    equipmentFlags: v.any(),
    caSyncStatus: v.union(
      v.literal("SYNCED"),
      v.literal("PENDING"),
      v.literal("FAILED")
    ),
  })
    .index("by_school", ["schoolId"])
    .index("by_sync_status", ["caSyncStatus"]),

  aircraftGroups: defineTable({
    schoolId: v.id("schools"),
    name: v.string(),
  }).index("by_school", ["schoolId"]),

  aircraftGroupMembers: defineTable({
    aircraftId: v.id("aircraft"),
    groupId: v.id("aircraftGroups"),
  })
    .index("by_aircraft", ["aircraftId"])
    .index("by_group", ["groupId"]),

	// ==========================================
  // 2. CA INTEGRATION (Bookings Cache)
  // ==========================================
  bookings: defineTable({
    schoolId: v.id("schools"),
    caBookingId: v.optional(v.string()), 
    caPilotId: v.optional(v.string()),      
    localWalkInName: v.optional(v.string()), 
    caInstructorId: v.optional(v.string()),
    requestedModelId: v.optional(v.id("aircraftModels")),
    requestedGroupId: v.optional(v.id("aircraftGroups")),
    assignedAircraftId: v.optional(v.id("aircraft")),
    startTime: v.number(),
    endTime: v.number(),
    status: v.string(),
  })
    .index("by_school", ["schoolId"])
    .index("by_time", ["startTime"]),

  // ==========================================
  // 3. DISPATCH
  // ==========================================
  dispatchRecords: defineTable({
    schoolId: v.id("schools"),
    bookingId: v.id("bookings"),
    aircraftId: v.id("aircraft"),
    dispatchedByStaffId: v.id("staff"),
    checkedInByStaffId: v.optional(v.id("staff")),
    hobbsStart: v.number(),
    hobbsEnd: v.optional(v.number()),
    tachStart: v.number(),
    tachEnd: v.optional(v.number()),
    fuelAddedGal: v.optional(v.number()),
    fuelType: v.optional(v.string()),
    checkoutTime: v.number(),
    checkinTime: v.optional(v.number()),
    status: v.union(v.literal("OUT"), v.literal("COMPLETED")),
  })
    .index("by_school", ["schoolId"])
    .index("by_aircraft_and_status", ["aircraftId", "status"]),

  // ==========================================
  // 4. MAINTENANCE LITE
  // ==========================================
  squawks: defineTable({
    schoolId: v.id("schools"),
    aircraftId: v.id("aircraft"),
    reporterName: v.string(),
    description: v.string(),
    severity: v.union(v.literal("LOW"), v.literal("MEDIUM"), v.literal("HIGH"), v.literal("GROUNDED")),
    status: v.union(
      v.literal("OPEN"),
      v.literal("IN_REVIEW"),
      v.literal("RESOLVED"),
      v.literal("DEFERRED")
    ),
    resolutionNotes: v.optional(v.string()),
  }).index("by_aircraft_and_status", ["aircraftId", "status"]),

  aircraftDowntimes: defineTable({
    schoolId: v.id("schools"),
    aircraftId: v.id("aircraft"),
    squawkId: v.optional(v.id("squawks")),
    startTime: v.number(),
    endTime: v.optional(v.number()),
    reason: v.string(),
    createdByStaffId: v.id("staff"),
  }).index("by_aircraft", ["aircraftId"]),

  // ==========================================
  // 5. COMPLIANCE LITE
  // ==========================================
  checkoutRequirements: defineTable({
    schoolId: v.id("schools"),
    name: v.string(),
    description: v.string(),
    targetModelId: v.optional(v.id("aircraftModels")),
    targetAircraftId: v.optional(v.id("aircraft")),
  }).index("by_school", ["schoolId"]),

  pilotCheckouts: defineTable({
    schoolId: v.id("schools"),
    caPilotId: v.string(),
    requirementId: v.id("checkoutRequirements"),
    grantedByStaffId: v.optional(v.id("staff")),
    origin: v.union(v.literal("CA_SYNC"), v.literal("LOCAL_OVERRIDE")),
    expirationDate: v.optional(v.number()),
    caSyncStatus: v.optional(v.union(v.literal("SYNCED"), v.literal("PENDING"), v.literal("FAILED"))),
  }).index("by_pilot", ["caPilotId"]),
});
```