# Flight Ops - Bootstrapping & Core Algorithms

---

This document contains the initialization scripts required to start the FlightOps application from a blank database, as well as the complex interval-math logic required to compute outbound aircraft availability.

## 1. Environment Setup

To run this Convex + Next.js + Clerk stack, the following environment variables are required in `.env.local`:

```markdown
Clerk Auth
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

Convex
NEXT_PUBLIC_CONVEX_URL=[https://your-convex-url.convex.cloud](https://your-convex-url.convex.cloud/)

FlightOps Integrations
CA_API_KEY=your_secure_webhook_key
```

## 2. Database Bootstrap (Seed Script)

Because the application relies on strict Clerk authentication and requires a user's `clerkId` to exist in the `staff` table before they can do anything, you must seed the database to bypass the "Unauthorized" lock on day one.

**Instructions for AI / Developer:** Create this mutation and run it once from the Convex CLI/Dashboard, passing in your personal Clerk User ID.

```tsx
// convex/init.ts
import { mutation } from "./_generated/server";
import { v } from "convex/values";

export const seedInitialAdmin = mutation({
  args: {
    clerkId: v.string(), // Pass your Clerk User ID here
    firstName: v.string(),
    email: v.string(),
  },
  handler: async (ctx, args) => {
    // 1. Create the foundational School
    const schoolId = await ctx.db.insert("schools", {
      name: "FlightOps Alpha Academy",
      timezone: "America/New_York",
      operatingHours: [
        { dayOfWeek: 1, open: "08:00", close: "18:00" },
        { dayOfWeek: 2, open: "08:00", close: "18:00" },
        { dayOfWeek: 3, open: "08:00", close: "18:00" },
        { dayOfWeek: 4, open: "08:00", close: "18:00" },
        { dayOfWeek: 5, open: "08:00", close: "18:00" },
        { dayOfWeek: 6, open: "09:00", close: "15:00" },
        { dayOfWeek: 0, open: "09:00", close: "15:00" }
      ],
      settings: {},
    });

    // 2. Create your Admin Staff profile
    const staffId = await ctx.db.insert("staff", {
      schoolId: schoolId,
      clerkId: args.clerkId,
      firstName: args.firstName,
      lastName: "Admin",
      email: args.email,
      role: "ADMIN",
    });

    // 3. Create a test Aircraft
    const makeId = await ctx.db.insert("aircraftMakes", { name: "Cessna" });
    const modelId = await ctx.db.insert("aircraftModels", { makeId, name: "172 Skyhawk" });

    await ctx.db.insert("aircraft", {
      schoolId: schoolId,
      modelId: modelId,
      tailNumber: "N12345",
      currentHobbs: 1200.5,
      currentTach: 1150.0,
      registration: "Active",
      equipmentFlags: { ifr: true },
      caSyncStatus: "SYNCED",
    });

// 4. Push initial availability to CA so the plane becomes bookable
    await ctx.scheduler.runAfter(0, internal.caIntegration.pushAvailability, {
      aircraftId: args.aircraftId,
      schoolId: schoolId,
    });

    return { success: true, schoolId, staffId };
  },
});
```

## 3. Core Algorithm: `calculateSlots`

This is the most complex query in the application. It fetches the raw operational data and calculates exact bookable time windows by subtracting downtime and active bookings from the school's operating hours.

**Instructions for AI / Developer:**
Implement the following logic structure. Ensure the `subtractBusyIntervals` helper function is implemented exactly as mapped out below to avoid overlapping timeline bugs.

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
    // (Developer: Implement loop using date-fns to map `school.operatingHours` array 
    // to actual Date.now timestamps for each day, pushing to `baseOperatingIntervals`)
    // ⚠️ CRITICAL INSTRUCTION FOR AI/DEVELOPER:
    // Convex servers run in UTC. The `school.operatingHours` are stored as local time strings (e.g., "08:00").
    // You MUST use `date-fns-tz` to map these strings to the `school.timezone` (e.g., "America/New_York")
    // before converting them to Unix epoch timestamps. 
    // Example: If NY is open at 08:00 EST, that is 13:00 UTC. Do NOT use standard `new Date().setHours()`.
    let baseOperatingIntervals = []; // e.g., [{ start: 1700000000, end: 1700036000 }]

    // 5. Apply the subtraction algorithm
    return subtractBusyIntervals(baseOperatingIntervals, busyIntervals);
  },
});
```