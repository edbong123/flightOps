# Flight Ops - Kickoff Prompts

Use these prompts to kick off your frontend generation in v0, Cursor, or your preferred AI coding tool. The secret to a great result is starting with the static UI first, and then wiring it to the backend second.

## Prompt 1: Initial UI Generation (For v0 / Claude)

*Copy and paste this into v0.dev or Claude to generate the React components and layout.*

> You are an expert full-stack developer specializing in Next.js (App Router), React, Tailwind CSS, shadcn/ui, Clerk Authentication, and Convex.
> 
> 
> I am building the MVP for "FlightOps," an internal operational dashboard for flight schools (think of it like a hotel property management system, but for airplanes). FlightOps handles fleet status, aircraft dispatching (check-out/check-in), and maintenance groundings. It syncs data with an external centralized booking platform (CA) and handles "walk-in" students via a unified local booking flow.
> 
> **YOUR TASK:**
> Please build the main "Dispatcher Dashboard" UI. This is the operational command center for the logged-in staff member.
> 
> **UI / UX REQUIREMENTS:**
> 
> 1. **Layout:** A clean, dense, operational layout. Create a sidebar navigation (Dashboard, Fleet, Dispatch, Maintenance, Settings) and a main content area. Assume the user is securely logged in via Clerk (show a UserProfile or SignOut stub in the top right or bottom of the nav).
> 2. **Components:** Use shadcn/ui components (Cards, Data Tables, Badges, Dialogs, Buttons) and Lucide React icons.
> 3. **The Warning Banner:** At the top of the dashboard, include a conditionally rendered alert banner (using destructive/red colors) that says: "⚠️ System Alert: Tail N12345 was grounded locally, but the sync to the CA Booking Platform failed. Please verify in CA manually."
> 4. **Key Dashboard Sections:**
>     - ***Active Flights (Check-in/Out):** A table or card grid showing flights scheduled for today* (representing both CA syncs and local walk-ins)*. Show the Pilot Name (use `localWalkInName` if `caPilotId` is missing), Aircraft Tail,* Departure Time, and a primary action button ("Check-out" or "Check-in").
>     - **Fleet Status:** A quick visual summary of available vs. grounded aircraft.
>     - **Maintenance Queue (Squawks):** A list of active squawks (discrepancies). Show the Tail Number, Severity (Low, Medium, High, Grounded), Description, `reporterName` (e.g., "Reported by John D."), and a "Review" button.
> 
> **BACKEND CONTEXT (CONVEX):**
> To help you structure the mock data and component props correctly, here is a simplified version of my Convex database schema. Please structure your frontend state and mock data to mirror this exactly:
> 
> - `staff`: { clerkId: string, role: "ADMIN" | "DISPATCHER" | "INSTRUCTOR" | "MECHANIC" }
> - `aircraft`: { tailNumber: string, currentHobbs: number, caSyncStatus: "SYNCED" | "PENDING" | "FAILED" }
> - `dispatchRecords`: { aircraftId: string, hobbsStart: number, status: "OUT" | "COMPLETED" }
> - `squawks`: { aircraftId: string, description: string, severity: "LOW" | "MEDIUM" | "HIGH" | "GROUNDED", status: "OPEN" | "IN_REVIEW", reporterName: string }
> 
> Please generate the complete, responsive Next.js page for this dashboard with rich mock data so I can see how it looks and functions. Make it look professional, modern, and highly utilitarian.
> 

---

## Prompt 2: Backend Integration (For Cursor / Copilot)

*Once you have the generated UI code inside your local IDE, open the file and use this prompt to connect it to your actual Convex backend.*

> You are an expert Next.js and Convex developer.
> 
> 
> I have this UI component for the "Dispatcher Dashboard". It currently looks great but relies entirely on static, hardcoded mock data.
> 
> **YOUR TASK:**
> I need you to "wire up" this component so it talks directly to my live Convex backend and Clerk authentication. Please rewrite this component to replace all the mock data and empty button clicks with actual Convex hooks.
> 
> **SPECIFIC INSTRUCTIONS:**
> 
> 1. **Auth Context:** Import `useUser` from `@clerk/nextjs` to get the logged-in user. Use their identity to securely interact with the backend.
> 2. **Queries:** Import `useQuery` from `convex/react` and use it to fetch the real data.
>     - Replace the maintenance queue mock data by calling the `getActiveSquawks` query (from `@convex/maintenance.ts`).
>     - Call a query to find aircraft where `caSyncStatus === "FAILED"` to conditionally trigger the warning banner.
> 3. **Mutations:** Import `useMutation` from `convex/react`.
>     - Wire up the "Check-out" and "Check-in" buttons to call the `checkOut` and `checkIn` mutations (from `@convex/dispatch.ts`).
>     - Wire up the "Ground Aircraft" button to call `groundAircraft` (from `@convex/maintenance.ts`).
> 4. **State Management:** Add loading states. If `useQuery` returns `undefined`, show a loading skeleton or spinner instead of crashing the page.
> 5. **Error Handling:** My Convex mutations have strict data integrity guards (e.g., they will throw an error if an aircraft is already checked out, or if meter readings are lower than the start). Wrap your mutation calls in `try/catch` blocks and use `toast` from shadcn to display these errors safely to the user.
> 
> Please review my Convex backend files (schema, dispatch, maintenance) to ensure you are passing the exact correct arguments to the mutations. Output the fully refactored React component.
> 

---

### 💡 Pro-Tips for the Build Phase:

- **Iterate locally:** Once the initial dashboard is generated, highlight specific parts of the UI (like the "Check-out" button) and ask the AI to: *"Build the Dialog/Modal that opens when I click this, including form inputs for Hobbs and Tach meters."*
- **Keep it strictly UI first:** Always ensure you are happy with the static visual design before applying Prompt 2. Changing the visual layout is much harder once the complex database hooks are wired in.