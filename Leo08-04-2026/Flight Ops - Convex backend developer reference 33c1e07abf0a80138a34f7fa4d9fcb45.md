# Flight Ops - Convex backend developer reference

This document contains the complete, production-ready backend code for FlightOps MVP, broken down into logical files. It includes Clerk authentication, data integrity guards, and hybrid sync error handling.

## 1. Database Schema

Defines tables, strict types, and query indexes.

```tsx
// convex/schema.ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  schools: defineTable({
    name: v.string(),
    timezone: v.string(),
    operatingHours: v.any(),
    settings: v.any(),
  }),

  staff: defineTable({
    schoolId: v.id("schools"),
    clerkId: v.string(), // Links to Clerk external auth
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
    caSyncStatus: v.union(v.literal("SYNCED"), v.literal("PENDING"), v.literal("FAILED")),
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

  bookings: defineTable({
    schoolId: v.id("schools"),
    caBookingId: v.optional(v.string()), // Nullable for local walk-ins
    caPilotId: v.string(),
    caInstructorId: v.optional(v.string()),
    requestedModelId: v.optional(v.id("aircraftModels")),
    requestedGroupId: v.optional(v.id("aircraftGroups")),
    assignedAircraftId: v.optional(v.id("aircraft")),
    startTime: v.number(),
    endTime: v.number(),
    status: v.string(), // CONFIRMED, CANCELLED, IN_PROGRESS
  })
    .index("by_school", ["schoolId"])
    .index("by_time", ["startTime"]),

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

  squawks: defineTable({
    schoolId: v.id("schools"),
    aircraftId: v.id("aircraft"),
    reporterName: v.string(), // Flat string for CA Pilots or FO Staff
    description: v.string(),
    severity: v.union(v.literal("LOW"), v.literal("MEDIUM"), v.literal("HIGH"), v.literal("GROUNDED")),
    status: v.union(v.literal("OPEN"), v.literal("IN_REVIEW"), v.literal("RESOLVED"), v.literal("DEFERRED")),
    resolutionNotes: v.optional(v.string()),
  }).index("by_aircraft_and_status", ["aircraftId", "status"]),

  aircraftDowntimes: defineTable({
    schoolId: v.id("schools"),
    aircraftId: v.id("aircraft"),
    squawkId: v.optional(v.id("squawks")),
    startTime: v.number(),
    endTime: v.optional(v.number()), // Null means grounded indefinitely
    reason: v.string(),
    createdByStaffId: v.id("staff"),
  }).index("by_aircraft", ["aircraftId"]),

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
  }).index("by_pilot", ["caPilotId"]),
});
```

---

## 2. Inbound Webhooks (CA → FO)

Receives bookings and cancellations from the central platform.

```tsx
// convex/http.ts
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";
import { internal } from "./_generated/api";

const http = httpRouter();

// 1. Inbound Bookings
http.route({
  path: "/webhooks/bookings",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const payload = await request.json();
    await ctx.runMutation(internal.bookings.upsertFromCA, {
      caBookingId: payload.id,
      schoolId: payload.schoolId,
      caPilotId: payload.pilotId,
      status: payload.status, 
      startTime: payload.startTime,
      endTime: payload.endTime,
      requestedModelId: payload.requestedModelId,
      assignedAircraftId: payload.assignedAircraftId
    });
    return new Response(null, { status: 200 });
  }),
});

// 2. Inbound Squawks
http.route({
  path: "/webhooks/squawks",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const payload = await request.json();
    await ctx.runMutation(internal.maintenance.ingestSquawk, {
      schoolId: payload.schoolId,
      aircraftId: payload.aircraftId,
      reporterName: payload.pilotName, // Sent directly from CA
      description: payload.description,
      severity: payload.severity,
    });
    return new Response(null, { status: 200 });
  }),
});

// 3. Inbound Pilot Checkouts
http.route({
  path: "/webhooks/checkouts",
  method: "POST",
  handler: httpAction(async (ctx, request) => {
    const payload = await request.json();
    await ctx.runMutation(internal.compliance.ingestCheckouts, {
      schoolId: payload.schoolId,
      caPilotId: payload.pilotId,
      requirements: payload.validRequirementIds, // Array of cleared IDs
    });
    return new Response(null, { status: 200 });
  }),
});

export default http;
```

Receives bookings and cancellations from the central platform.

## 3. Outbound Integrations (FO → CA)

Computes availability and pushes changes with exponential backoff logic.

```tsx
// convex/caIntegration.ts
import { internalAction } from "./_generated/server";
import { v } from "convex/values";
import { internal } from "./_generated/api";

export const pushAvailability = internalAction({
  args: { 
    aircraftId: v.id("aircraft"),
    schoolId: v.id("schools"),
    retryCount: v.optional(v.number()), 
  },
  handler: async (ctx, args) => {
    const attempts = args.retryCount ?? 0;
    const maxRetries = 3;

    const newAvailabilitySlots = await ctx.runQuery(internal.fleet.calculateSlots, {
      aircraftId: args.aircraftId
    });

    try {
      const response = await fetch("[https://api.ca-system.com/webhooks/availability](https://api.ca-system.com/webhooks/availability)", {
        method: "POST",
        headers: { 
          "Content-Type": "application/json",
          "Authorization": `Bearer ${process.env.CA_API_KEY}`
        },
        body: JSON.stringify({
          aircraftId: args.aircraftId,
          schoolId: args.schoolId,
          slots: newAvailabilitySlots
        }),
      });

      if (!response.ok) throw new Error(`Status ${response.status}`);

      await ctx.runMutation(internal.internalUtils.updateSyncStatus, {
        aircraftId: args.aircraftId,
        status: "SYNCED",
      });

    } catch (error) {
      if (attempts < maxRetries) {
        await ctx.scheduler.runAfter(30000, internal.caIntegration.pushAvailability, {
          aircraftId: args.aircraftId,
          schoolId: args.schoolId,
          retryCount: attempts + 1,
        });
      } else {
        await ctx.runMutation(internal.internalUtils.updateSyncStatus, {
          aircraftId: args.aircraftId,
          status: "FAILED", // Triggers UI warning banner
        });
      }
    }
  },
});
```

## 4. Compliance Sync (FO → CA)

Syncs local checkout overrides back to CA.

```tsx
// convex/compliance.ts
import { internalAction } from "./_generated/server";
import { v } from "convex/values";
import { internal } from "./_generated/api";

export const pushCheckoutToCA = internalAction({
  args: { checkoutId: v.id("pilotCheckouts"), schoolId: v.id("schools") },
  handler: async (ctx, args) => {
    const checkoutRecord = await ctx.runQuery(internal.compliance.getCheckout, { id: args.checkoutId });
    
    try {
      const response = await fetch("[https://api.ca-system.com/webhooks/checkouts](https://api.ca-system.com/webhooks/checkouts)", {
        method: "POST",
        headers: { "Content-Type": "application/json", "Authorization": `Bearer ${process.env.CA_API_KEY}` },
        body: JSON.stringify(checkoutRecord),
      });

      if (!response.ok) throw new Error("CA Sync Failed");
      await ctx.runMutation(internal.compliance.updateCheckoutSyncStatus, { id: args.checkoutId, status: "SYNCED" });
    } catch (error) {
      console.error("Failed to push checkout to CA", error);
      await ctx.runMutation(internal.compliance.updateCheckoutSyncStatus, { id: args.checkoutId, status: "FAILED" });
    }
  }
});
```

## 5. Internal Utilities

```tsx
// convex/internalUtils.ts
import { internalMutation } from "./_generated/server";
import { v } from "convex/values";

export const updateSyncStatus = internalMutation({
  args: {
    aircraftId: v.id("aircraft"),
    status: v.union(v.literal("SYNCED"), v.literal("FAILED")),
  },
  handler: async (ctx, args) => {
    await ctx.db.patch(args.aircraftId, {
      caSyncStatus: args.status,
    });
  },
});
```

## 6. Dispatch Workflow

Handles physical airplane checkout and return, strictly enforcing Auth, Compliance, and data integrity.

```tsx
// convex/dispatch.ts
import { mutation } from "./_generated/server";
import { v } from "convex/values";
import { internal } from "./_generated/api";

export const checkOut = mutation({
  args: {
    bookingId: v.id("bookings"),
    aircraftId: v.id("aircraft"),
    caPilotId: v.optional(v.string()),       // Updated for local walk-ins
    localWalkInName: v.optional(v.string()), // Updated for local walk-ins
    hobbsStart: v.number(),
    tachStart: v.number(),
  },
  handler: async (ctx, args) => {
    // 1. Auth & Context
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");
    
    const staff = await ctx.db.query("staff").withIndex("by_clerk", (q) => q.eq("clerkId", identity.subject)).first();
    if (!staff) throw new Error("Staff profile not found");

    // 2. Double Dispatch Guard
    const activeDispatch = await ctx.db.query("dispatchRecords").withIndex("by_aircraft_and_status", (q) => q.eq("aircraftId", args.aircraftId).eq("status", "OUT")).first();
    if (activeDispatch) throw new Error("CRITICAL: Aircraft is already checked out!");

    // 3. Strict Compliance Check (Bypassed for local walk-ins, enforced for CA Pilots)
    if (args.caPilotId) {
      const aircraftData = await ctx.db.get(args.aircraftId);
      if (!aircraftData) throw new Error("Aircraft not found");

      const requirements = await ctx.db.query("checkoutRequirements").withIndex("by_school", (q) => q.eq("schoolId", staff.schoolId)).filter((q) => q.or(q.eq(q.field("targetAircraftId"), args.aircraftId), q.eq(q.field("targetModelId"), aircraftData.modelId))).collect();

      if (requirements.length > 0) {
        const pilotCheckouts = await ctx.db.query("pilotCheckouts").withIndex("by_pilot", (q) => q.eq("caPilotId", args.caPilotId)).filter((q) => q.eq(q.field("schoolId"), staff.schoolId)).collect();

        for (const req of requirements) {
          const hasValidCheckout = pilotCheckouts.some(pc => pc.requirementId === req._id && (!pc.expirationDate || pc.expirationDate > Date.now()));
          if (!hasValidCheckout) throw new Error(`Pilot lacks a valid checkout for requirement: ${req.name}`);
        }
      }
    } else if (!args.localWalkInName) {
      throw new Error("Dispatch requires either a valid CA Pilot ID or a Local Walk-In Name.");
    }

    // 4. Create Dispatch Record
    const recordId = await ctx.db.insert("dispatchRecords", {
      schoolId: staff.schoolId,
      bookingId: args.bookingId,
      aircraftId: args.aircraftId,
      dispatchedByStaffId: staff._id,
      hobbsStart: args.hobbsStart,
      tachStart: args.tachStart,
      checkoutTime: Date.now(),
      status: "OUT",
    });

    // 5. Trigger CA Sync
    await ctx.scheduler.runAfter(0, internal.caIntegration.pushAvailability, {
      aircraftId: args.aircraftId,
      schoolId: staff.schoolId,
    });

    return recordId;
  },
});

export const checkIn = mutation({
  args: {
    dispatchRecordId: v.id("dispatchRecords"),
    aircraftId: v.id("aircraft"),
    hobbsEnd: v.number(),
    tachEnd: v.number(),
    fuelAddedGal: v.optional(v.number()),
    fuelType: v.optional(v.string()),
    newSquawks: v.optional(v.array(v.object({
      description: v.string(),
      severity: v.union(v.literal("LOW"), v.literal("MEDIUM"), v.literal("HIGH"), v.literal("GROUNDED"))
    })))
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");
    
    const staff = await ctx.db.query("staff").withIndex("by_clerk", (q) => q.eq("clerkId", identity.subject)).first();
    if (!staff) throw new Error("Staff profile not found");

    const record = await ctx.db.get(args.dispatchRecordId);
    if (!record) throw new Error("Dispatch record not found");

    // 2. Integrity Guards (Prevent rolling back meters)
    if (args.hobbsEnd < record.hobbsStart) throw new Error("Ending Hobbs cannot be less than Start Hobbs.");
    if (args.tachEnd < record.tachStart) throw new Error("Ending Tach cannot be less than Start Tach.");

    // 3. Close out record
    await ctx.db.patch(args.dispatchRecordId, {
      checkedInByStaffId: staff._id,
      hobbsEnd: args.hobbsEnd,
      tachEnd: args.tachEnd,
      fuelAddedGal: args.fuelAddedGal,
      fuelType: args.fuelType,
      checkinTime: Date.now(),
      status: "COMPLETED",
    });

    // 4. Update physical aircraft meters
    await ctx.db.patch(args.aircraftId, {
      currentHobbs: args.hobbsEnd,
      currentTach: args.tachEnd,
    });

    // 5. Process new squawks
    if (args.newSquawks && args.newSquawks.length > 0) {
      for (const squawk of args.newSquawks) {
        await ctx.db.insert("squawks", {
          schoolId: staff.schoolId,
          aircraftId: args.aircraftId,
          reporterName: staff.firstName,
          description: squawk.description,
          severity: squawk.severity,
          status: "OPEN"
        });
      }
    }

    // 6. Trigger CA Sync
    await ctx.scheduler.runAfter(0, internal.caIntegration.pushAvailability, {
      aircraftId: args.aircraftId,
      schoolId: staff.schoolId,
    });
  },
});
```

## 7. Maintenance & Dashboard Ops

Handles squawks and fleet groundings directly from the dispatch UI.

```tsx
// convex/maintenance.ts
import { mutation, query } from "./_generated/server";
import { v } from "convex/values";
import { internal } from "./_generated/api";

export const getActiveSquawks = query({
  args: { schoolId: v.id("schools") },
  handler: async (ctx, args) => {
    return await ctx.db
      .query("squawks")
      .filter((q) => 
        q.and(
          q.eq(q.field("schoolId"), args.schoolId),
          q.or(
            q.eq(q.field("status"), "OPEN"),
            q.eq(q.field("status"), "IN_REVIEW")
          )
        )
      )
      .collect();
  },
});

export const updateSquawkStatus = mutation({
  args: {
    squawkId: v.id("squawks"),
    status: v.union(v.literal("RESOLVED"), v.literal("DEFERRED")),
    resolutionNotes: v.optional(v.string()),
  },
  handler: async (ctx, args) => {
    await ctx.db.patch(args.squawkId, {
      status: args.status,
      resolutionNotes: args.resolutionNotes,
    });
  },
});

export const groundAircraft = mutation({
  args: {
    squawkId: v.id("squawks"),
    aircraftId: v.id("aircraft"),
    reason: v.string(),
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");
    
    const staff = await ctx.db.query("staff").withIndex("by_clerk", (q) => q.eq("clerkId", identity.subject)).first();
    if (!staff) throw new Error("Staff profile not found");

    await ctx.db.patch(args.squawkId, { status: "IN_REVIEW" });

    await ctx.db.insert("aircraftDowntimes", {
      schoolId: staff.schoolId,
      aircraftId: args.aircraftId,
      squawkId: args.squawkId,
      startTime: Date.now(), 
      reason: args.reason,
      createdByStaffId: staff._id,
    });

    await ctx.db.patch(args.aircraftId, { caSyncStatus: "PENDING" });

    await ctx.scheduler.runAfter(0, internal.caIntegration.pushAvailability, {
      aircraftId: args.aircraftId,
      schoolId: staff.schoolId,
      retryCount: 0,
    });
  },
});

export const ungroundAircraft = mutation({
  args: {
    aircraftId: v.id("aircraft"),
    squawkId: v.id("squawks"),
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Unauthorized");

    const staff = await ctx.db.query("staff").withIndex("by_clerk", (q) => q.eq("clerkId", identity.subject)).first();
    if (!staff) throw new Error("Staff profile not found");

    // 1. Mark squawk as resolved
    await ctx.db.patch(args.squawkId, { status: "RESOLVED" });

    // 2. Find the active downtime and close it
    const activeDowntime = await ctx.db
      .query("aircraftDowntimes")
      .withIndex("by_aircraft", (q) => q.eq("aircraftId", args.aircraftId))
      .filter((q) => q.eq(q.field("endTime"), undefined))
      .first();

    if (activeDowntime) {
      await ctx.db.patch(activeDowntime._id, { endTime: Date.now() });
    }

    // 3. Push restored availability to CA
    await ctx.db.patch(args.aircraftId, { caSyncStatus: "PENDING" });
    await ctx.scheduler.runAfter(0, internal.caIntegration.pushAvailability, {
      aircraftId: args.aircraftId,
      schoolId: staff.schoolId,
    });
  },
});

// Used by the CA Webhook
export const ingestSquawk = internalMutation({
  args: {
    schoolId: v.id("schools"),
    aircraftId: v.id("aircraft"),
    reporterName: v.string(),
    description: v.string(),
    severity: v.union(v.literal("LOW"), v.literal("MEDIUM"), v.literal("HIGH"), v.literal("GROUNDED")),
  },
  handler: async (ctx, args) => {
    await ctx.db.insert("squawks", {
      schoolId: args.schoolId,
      aircraftId: args.aircraftId,
      reporterName: args.reporterName,
      description: args.description,
      severity: args.severity,
      status: "OPEN",
    });
  }
});
```

## 8. Bookings

Internal mutation to process inbound webhooks and handle auto-assignment.

```tsx
// convex/bookings.ts
import { internalMutation } from "./_generated/server";
import { v } from "convex/values";

export const upsertFromCA = internalMutation({
  args: {
    caBookingId: v.string(),
    schoolId: v.id("schools"),
    caPilotId: v.string(),
    status: v.string(),
    startTime: v.number(),
    endTime: v.number(),
    requestedModelId: v.optional(v.id("aircraftModels")),
    assignedAircraftId: v.optional(v.id("aircraft")),
  },
  handler: async (ctx, args) => {
    let finalAircraftId = args.assignedAircraftId;

    // AUTO-ASSIGNMENT LOGIC: If CA booked a model but no specific tail number
    if (!finalAircraftId && args.requestedModelId) {
      const availablePlanes = await ctx.db
        .query("aircraft")
        .withIndex("by_school", q => q.eq("schoolId", args.schoolId))
        .filter(q => q.eq(q.field("modelId"), args.requestedModelId))
        .collect();
      
      // Simple MVP auto-assignment: just pick the first one matching the model
      // (In production, you'd run a slot check here to find one that is actually free)
      if (availablePlanes.length > 0) {
        finalAircraftId = availablePlanes[0]._id;
      }
    }

    // Check if booking exists
    const existing = await ctx.db
      .query("bookings")
      .filter(q => q.eq(q.field("caBookingId"), args.caBookingId))
      .first();

    if (existing) {
      await ctx.db.patch(existing._id, {
        status: args.status,
        startTime: args.startTime,
        endTime: args.endTime,
        assignedAircraftId: finalAircraftId,
      });
    } else {
      await ctx.db.insert("bookings", {
        schoolId: args.schoolId,
        caBookingId: args.caBookingId,
        caPilotId: args.caPilotId,
        status: args.status,
        startTime: args.startTime,
        endTime: args.endTime,
        requestedModelId: args.requestedModelId,
        assignedAircraftId: finalAircraftId,
      });
    }
  }
});
```

## 9. Fleet

*Contains the core availability algorithm and interval subtraction math.*

```tsx
// convex/fleet.ts
import { internalQuery } from "./_generated/server";
import { v } from "convex/values";

// Helper Function: The core interval subtraction math
function subtractBusyIntervals(availableSlots: {start: number, end: number}[], busySlots: {start: number, end: number}[]) {
  let currentAvailable = [...availableSlots];
  
  for (const busy of busySlots) {
    let nextAvailable = [];
    
    for (const slot of currentAvailable) {
      // Case 1: Busy completely covers the slot -> discard slot
      if (busy.start <= slot.start && busy.end >= slot.end) { continue; }
      
      // Case 2: Busy overlaps the start of the slot
      if (busy.start <= slot.start && busy.end > slot.start && busy.end < slot.end) {
        nextAvailable.push({ start: busy.end, end: slot.end });
      }
      // Case 3: Busy overlaps the end of the slot
      else if (busy.start > slot.start && busy.start < slot.end && busy.end >= slot.end) {
        nextAvailable.push({ start: slot.start, end: busy.start });
      }
      // Case 4: Busy falls strictly inside the slot -> split the slot in two
      else if (busy.start > slot.start && busy.end < slot.end) {
        nextAvailable.push({ start: slot.start, end: busy.start });
        nextAvailable.push({ start: busy.end, end: slot.end });
      }
      // Case 5: No overlap at all -> keep slot as is
      else {
        nextAvailable.push(slot);
      }
    }
    currentAvailable = nextAvailable;
  }
  return currentAvailable;
}

export const calculateSlots = internalQuery({
  args: {
    aircraftId: v.id("aircraft"),
    lookaheadDays: v.optional(v.number()), 
  },
  handler: async (ctx, args) => {
    const lookahead = args.lookaheadDays ?? 14;
    const now = Date.now();
    const horizon = now + (lookahead * 24 * 60 * 60 * 1000);

    const aircraft = await ctx.db.get(args.aircraftId);
    if (!aircraft) throw new Error("Aircraft not found");

    const school = await ctx.db.get(aircraft.schoolId);
    if (!school) throw new Error("School not found");
    
    // 1. Fetch all overlapping ACTIVE bookings
    const activeBookings = await ctx.db
      .query("bookings")
      .withIndex("by_school", (q) => q.eq("schoolId", aircraft.schoolId))
      .filter((q) => q.and(
        q.eq(q.field("assignedAircraftId"), args.aircraftId),
        q.neq(q.field("status"), "CANCELLED"),
        q.gte(q.field("endTime"), now)
      ))
      .collect();

    // 2. Fetch all overlapping DOWNTIMES (Maintenance)
    const activeDowntimes = await ctx.db
      .query("aircraftDowntimes")
      .withIndex("by_aircraft", (q) => q.eq("aircraftId", args.aircraftId))
      .filter((q) => q.or(
        q.eq(q.field("endTime"), undefined), // Grounded indefinitely
        q.gte(q.field("endTime"), now)
      ))
      .collect();

    // 3. Build array of busy intervals
    const busyIntervals = [
      ...activeBookings.map(b => ({ start: b.startTime, end: b.endTime })),
      ...activeDowntimes.map(d => ({ start: d.startTime, end: d.endTime ?? horizon }))
    ];

    // 4. Generate standard operating hours intervals for the next `lookahead` days
    // ⚠️ CRITICAL NOTE FOR UI DEVELOPER:
    // Convex servers run in UTC. The `school.operatingHours` are stored as local time strings (e.g., "08:00").
    // You MUST use `date-fns-tz` to map these strings to the `school.timezone` (e.g., "America/New_York")
    // before converting them to Unix epoch timestamps. 
    // Example: If NY is open at 08:00 EST, that is 13:00 UTC. Do NOT use standard `new Date().setHours()`.
    
    // Placeholder array until date-fns-tz interval mapping is implemented based on the warning above
    let baseOperatingIntervals = [{ start: now, end: horizon }]; 

    // 5. Apply the subtraction algorithm
    return subtractBusyIntervals(baseOperatingIntervals, busyIntervals);
  },
});
```