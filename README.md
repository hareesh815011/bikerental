# RideTrack – Smart Bike Rental & Tracking System

RideTrack is a responsive bike-rental dashboard prototype built for an India-focused rental fleet. The current release demonstrates the customer experience with realistic sample data and a full-stack-ready application scaffold.

> **Implementation status:** The customer-facing workflows are implemented with browser-local sample data. Authentication infrastructure and the core user table are scaffolded. Domain APIs, persistent bike and booking records, payments, KYC integrations, GPS ingestion, and admin operations remain planned production work.

## Features currently demonstrated

- Customer overview dashboard with active rental metrics and schedule
- 32-bike sample fleet across electric, street, scooter, cruiser, sport, adventure, premium, and commuter categories
- Bike search, category chips, location/type/sort controls, and responsive bike cards
- Booking configuration modal with pickup, duration, estimated total, and KYC eligibility messaging
- Booking history table with active, confirmed, upcoming, and completed states
- Live tracking view with illustrative route map, GPS state, speed, distance, battery, and return estimate
- Emergency action with user-facing support guidance
- KYC centre with masked Aadhaar display, licence status, progress state, and privacy messaging
- Notification popover, support entry points, responsive mobile navigation, and reduced-motion support

## Technology stack

- React 19 and TypeScript
- Vite for frontend development and production bundling
- Tailwind CSS 4 with project-specific CSS design tokens
- Radix/shadcn-style UI primitives
- Lucide React icons
- Sonner for toast notifications
- Node.js, Express, and tRPC server scaffold
- Drizzle ORM with MySQL/TiDB configuration
- Manus OAuth session infrastructure
- Vitest for unit tests

## Project structure

```text
client/
  src/
    App.tsx                       Application providers and root component
    pages/Home.tsx                Dashboard prototype and sample workflows
    components/ErrorBoundary.tsx  Application-level React error boundary
    components/ui/                Reusable UI primitives
    contexts/ThemeContext.tsx     Theme provider
    index.css                     Design tokens and responsive styling
  public/                         Small static configuration assets

drizzle/
  schema.ts                       Drizzle database schema
  relations.ts                    Drizzle relation definitions
  migrations/                     Generated database migrations

server/
  routers.ts                      tRPC application router
  db.ts                           Database connection and user helpers
  _core/                          Auth, context, OAuth, Vite, and runtime plumbing
  auth.logout.test.ts             Example server-side unit test

shared/
  const.ts                        Shared application constants
  types.ts                        Shared types

RideTrack_Project_Report.md       Product and implementation report
```

## Local development

From the project root:

```bash
pnpm install
pnpm dev
```

The development server runs through `server/_core/index.ts` and serves the Vite frontend. Project environment variables are supplied by the WebDev runtime. Do not commit `.env` files or copy secrets into documentation.

## Validation commands

```bash
# TypeScript compilation without emitting files
pnpm check

# Production frontend and server build
pnpm build

# Unit test suite
pnpm test

# Generate and apply Drizzle migrations when the schema changes
pnpm db:push
```

The current project has passed `pnpm check` and `pnpm build`. The build may report a bundle-size advisory for the main frontend chunk; this is an optimization warning rather than a build failure.

## Unit testing guide

Tests use Vitest and should be placed next to the server behavior they cover or in a clearly named `*.test.ts` file. The existing `server/auth.logout.test.ts` is the reference pattern.

### Test structure

1. Import `describe`, `it`, and `expect` from `vitest`.
2. Create a small test context rather than starting the HTTP server.
3. Build a caller with `appRouter.createCaller(ctx)`.
4. Invoke the tRPC procedure directly.
5. Assert both the returned value and important side effects.
6. Keep tests deterministic and independent of external services.

Example test flow:

```ts
const { ctx, clearedCookies } = createAuthContext();
const caller = appRouter.createCaller(ctx);
const result = await caller.auth.logout();

expect(result).toEqual({ success: true });
expect(clearedCookies).toHaveLength(1);
```

### Recommended coverage as domain APIs are added

- **Bike availability:** unavailable or maintenance bikes cannot be booked.
- **Booking conflicts:** overlapping bookings for the same bike are rejected.
- **Authorization:** riders cannot access another rider’s booking, KYC documents, or location history.
- **KYC gating:** booking confirmation is rejected until required verification is approved.
- **Payment callbacks:** duplicate provider callbacks are idempotent.
- **Emergency reports:** authenticated reports retain booking, bike, rider, and timestamp associations.
- **Admin procedures:** non-admin users receive a forbidden response.
- **Privacy:** Aadhaar and driving-licence identifiers are masked in user-facing responses.

Tests should use factories for users, bikes, bookings, and KYC records once those tables exist. Database integration tests should run against an isolated test database and should clean up records after each test or use transaction rollback.

## Error-boundary behavior

`client/src/components/ErrorBoundary.tsx` wraps the complete application in `App.tsx`. It catches errors thrown during rendering, lifecycle methods, and child component constructors. When an error is caught, React calls `getDerivedStateFromError`, which switches the component into a safe fallback state.

The fallback UI currently:

- Shows a clear unexpected-error message.
- Displays the captured stack for debugging in the current development-oriented prototype.
- Provides a Reload Page action that resets the browser application state.

The error boundary does **not** automatically catch errors from event handlers, asynchronous callbacks, or rejected tRPC requests. Those paths must be handled locally with mutation error callbacks, query error states, and toast or inline messaging.

### Production hardening recommendations

- Hide stack traces from end users in production builds.
- Add an `onError` reporting hook for a monitoring service.
- Include a correlation ID in reported errors.
- Add route-level boundaries so one feature page does not blank the whole application.
- Provide a recovery action that returns to a stable route instead of only reloading.
- Test fallback rendering by mounting a component that intentionally throws.

## API documentation

The backend uses tRPC, so the router is the type-safe API contract. The client should call procedures through the generated `trpc` hooks rather than introducing ad hoc `fetch` or Axios wrappers.

### Current procedures

| Procedure | Type | Access | Purpose |
|---|---|---|---|
| `auth.me` | Query | Public context | Returns the current authenticated user or `null`/`undefined` when no user is present. |
| `auth.logout` | Mutation | Public context | Clears the configured session cookie and returns `{ success: true }`. |
| `system.*` | Router | Runtime-defined | Framework and platform procedures provided by the server scaffold. |

The current application router is intentionally small. Domain procedures should be added as feature routers rather than expanding one large file.

### Planned domain routers

| Router | Representative procedures | Access expectation |
|---|---|---|
| `bikes` | `list`, `getById`, `availability` | Public read; admin write |
| `bookings` | `create`, `getMine`, `cancel`, `updateStatus` | Authenticated; admin status changes |
| `kyc` | `getMine`, `submit`, `review` | Authenticated; authorised staff review |
| `payments` | `createIntent`, `confirm`, `getInvoice` | Authenticated; provider webhook protected |
| `tracking` | `getActive`, `getTelemetry`, `history` | Rider-own booking or admin |
| `emergencies` | `create`, `getMine`, `updateStatus` | Authenticated; support/admin updates |
| `admin` | `metrics`, `customers`, `maintenance` | Admin only |

Every protected procedure should validate its input with Zod, load records through server-side helpers, and enforce ownership or role checks before returning data.

## Database schema

The current Drizzle schema contains the authentication-backed `users` table:

| Column | Type/constraint | Purpose |
|---|---|---|
| `id` | Auto-increment integer primary key | Internal user identifier |
| `openId` | Required unique varchar | Manus OAuth identity |
| `name` | Nullable text | Display name |
| `email` | Nullable varchar | Contact email |
| `loginMethod` | Nullable varchar | Authentication method metadata |
| `role` | `user` or `admin` enum | Role-based access foundation |
| `createdAt` | Required timestamp | Account creation time |
| `updatedAt` | Required timestamp | Last account update |
| `lastSignedIn` | Required timestamp | Last successful sign-in |

The planned relational model should add:

- `customer_profiles` for rental profile and contact details
- `bikes` for fleet identity, specifications, pricing, status, and images
- `bike_locations` for current and historical coordinates
- `bike_telemetry` for speed, distance, battery, GPS, and engine readings
- `bookings` for rental periods, status, pickup/return details, and totals
- `payments` for provider references, status, amount, invoice, and timestamps
- `kyc_records` and `driving_licence_records` for minimised verification data
- `emergency_reports` for incident type, location, evidence, and resolution status
- `feedback` for ratings and post-rental comments
- `maintenance_records` for service dates, due dates, and work history
- `notifications` for user-facing delivery state and event type
- `support_tickets` for non-emergency customer support
- `audit_logs` for sensitive access and state changes

Recommended indexes include bike status plus location, booking bike plus date range, booking rider plus created time, KYC status, emergency status, and telemetry bike plus recorded time. Document bytes should be stored in private object storage; database rows should retain only metadata and secure object references.

## Security notes

The interface masks sensitive identity information, but production privacy controls must be enforced by the server. Do not return full Aadhaar numbers, raw payment credentials, CVV values, or unrestricted document URLs. Use role checks, ownership checks, input validation, upload restrictions, audit logging, and short-lived signed URLs.

## Current limitations and next steps

The current frontend uses browser-local sample data. The next implementation milestone is to add the domain schema and typed procedures, then connect booking persistence, Razorpay or another compliant payment provider, secure KYC storage, GPS/WebSocket updates, and an admin workspace.

## Documentation maintenance

Update this README whenever any of the following changes:

- A tRPC procedure is added, renamed, or changes access rules.
- A database table, index, enum, or sensitive field changes.
- A new test category or error-handling policy is introduced.
- Authentication, storage, payment, tracking, or notification integrations are connected.
- The customer flow or admin workflow changes.
