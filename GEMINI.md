# Restaurant ERP & POS Order Engine - System Architecture Documentation

This document serves as the single source of truth for the codebase structure, database models, multi-tenant isolation, server-side pricing integrity, shift-window constraints, and testing protocols.

---

## 1. Folder Structure & Naming Conventions

### Directory Layout

```
restaurant-erp/
├── client/                            # React 18 + Vite KDS Frontend
│   ├── src/
│   │   ├── components/
│   │   │   ├── KdsHeader.tsx          # Station selector, clock, live counter, connection pill
│   │   │   ├── KdsTicketCard.tsx      # Ticket card (dynamic aging headers, modifiers, touch bump)
│   │   │   ├── KdsTicketGrid.tsx      # Responsive auto-wrapping ticket container
│   │   │   ├── OfflineStatusBanner.tsx# "Offline Mode — Queued (N)" dynamic alert banner
│   │   │   ├── StationBadge.tsx       # Prep station color badge (GRILL, BAR, FRY, etc.)
│   │   │   └── TicketTimer.tsx        # Live 1-second interval timer
│   │   ├── hooks/
│   │   │   ├── useElapsedTime.ts      # Elapsed duration hook (<10m normal, 10-15m yellow, >15m red)
│   │   │   ├── useKdsSocket.ts        # Socket.io connection, room join, reconnect, optimistic bump
│   │   │   └── useOfflineQueue.ts     # Offline queue state, connectivity detector, and sync trigger
│   │   ├── offline/
│   │   │   ├── indexedDb.ts           # IndexedDB persistence (menu, taxes, staff PINs, queue, print jobs)
│   │   │   ├── localTicketPrinter.ts  # Local 80mm ESC/POS prep ticket formatting during WAN outages
│   │   │   ├── offlineQueueManager.ts # Queue manager with UUID idempotency and offline total calculations
│   │   │   ├── pinVerifier.ts         # Isomorphic PBKDF2 PIN verification (Web Crypto + Node)
│   │   │   └── syncWorker.ts          # Automatic background sync worker with chronological replay
│   │   ├── styles/
│   │   │   └── kds.css                # Dark-mode, anti-glare kitchen UI styles
│   │   ├── types/
│   │   │   └── kds.ts                 # Strongly-typed ticket and socket interfaces
│   │   ├── App.tsx                    # KDS application root coordinator
│   │   └── main.tsx                   # React DOM entrypoint
│   ├── package.json                   # Client dependencies (React 18, socket.io-client, lucide-react)
│   ├── tsconfig.json                  # Strict client TypeScript configuration
│   └── vite.config.ts                 # Vite bundler configuration
├── prisma/
│   ├── migrations/
│   │   ├── 20261116065754_harden_multitenant_orders/
│   │   │   └── migration.sql          # Multi-tenant hardened PostgreSQL migration
│   │   ├── 20261116071207_add_inventory_bom/
│   │   │   └── migration.sql          # Recipe BOM & Inventory PostgreSQL migration
│   │   └── 20261116073632_add_payments_hardware/
│   │       └── migration.sql          # Payments, Refunds & Print Jobs PostgreSQL migration
│   └── schema.prisma                  # Prisma ORM schema and database mappings
├── src/
│   │   ├── controllers/
│   │   │   ├── OrderController.ts         # Route handlers for create, settle, and get order
│   │   │   ├── InventoryController.ts     # Admin handlers for ingredients, recipes, and POs
│   │   │   ├── AnalyticsPayrollTaxController.ts # Food-cost, attendance, payroll CSV, and tax handlers
│   │   │   ├── PaymentController.ts       # Stripe Terminal intents, webhooks, tips, and split-checks
│   │   │   ├── ManagerOverrideController.ts # PIN-protected voids, comps, discounts, and refunds
│   │   │   ├── ThermalPrinterController.ts# 80mm ESC/POS receipts, prep tickets, and hardware queue
│   │   │   ├── PosSyncController.ts       # POS bootstrap endpoint for menu, taxes, and staff caching
│   │   │   └── SaasBillingOnboardingController.ts # Onboarding, Stripe billing webhooks, CSV menu importer
│   │   ├── middleware/
│   │   │   ├── auth.ts                    # JWT verification, RBAC, shift-window enforcement
│   │   │   ├── idempotency.ts             # Distributed UUID-based idempotency lock
│   │   │   └── subscription.ts            # Active SaaS subscription enforcement (blocks suspended tenants)
│   │   ├── routes/
│   │   │   ├── orderRoutes.ts             # POST /orders, POST /orders/:id/settle, GET /orders/:id
│   │   │   ├── inventoryRoutes.ts         # /admin/ingredients, /admin/recipes, /admin/purchase-orders
│   │   │   ├── analyticsPayrollTaxRoutes.ts # /analytics/food-cost, /payroll, /tax-rules, /orders/:id/receipt
│   │   │   ├── paymentHardwareRoutes.ts   # /payments, /refunds, /manager-override, /printers
│   │   │   └── saasBillingRoutes.ts       # /onboarding/register, /billing/webhook, /menu/import
│   │   ├── services/
│   │   │   ├── OrderEngineService.ts      # Authoritative price recalculation, order creation & settling
│   │   │   ├── InventoryService.ts        # BOM deduction, SELECT FOR UPDATE locking, draft PO trigger
│   │   │   ├── FoodCostAnalyticsService.ts # Daily net revenue, theoretical food cost, variance
│   │   │   ├── AttendancePayrollService.ts # Haversine geofence validation, overtime calculation, CSV export
│   │   │   ├── TaxEInvoicingService.ts    # Multi-rule tax engine, sequential fiscal numbering, UBL XML receipts
│   │   │   ├── PaymentService.ts          # Stripe Terminal intents, penny-accurate split check math
│   │   │   ├── ManagerOverrideService.ts  # Audit-logged manager PIN overrides
│   │   │   ├── ThermalPrinterService.ts   # ESC/POS byte streaming and queue lifecycle
│   │   │   ├── SubscriptionBillingService.ts # Stripe webhook processing, tier limits (STARTER/GROWTH/ENTERPRISE)
│   │   │   ├── OnboardingService.ts       # Atomic multi-step onboarding transaction (Tenant, Loc, Admin, Tax)
│   │   │   └── MenuImportService.ts       # CSV preview validation & atomic batch commit transaction
│   │   ├── tenantPrisma.ts            # Prisma client extension enforcing tenant isolation
│   │   └── enableRls.ts               # PostgreSQL Row-Level Security (RLS) enforcer
│   ├── middleware/
│   │   ├── auth.ts                    # JWT auth, RBAC, and shift window constraint
│   │   └── idempotency.ts             # Distributed idempotency key middleware
│   ├── routes/
│   │   ├── orderRoutes.ts             # Express router definitions for POS orders & bootstrap
│   │   ├── inventoryRoutes.ts         # Express router for admin inventory & BOM
│   │   ├── analyticsPayrollTaxRoutes.ts # Express router for analytics, payroll & e-invoicing
│   │   └── paymentHardwareRoutes.ts   # Express router for payments, overrides & thermal printers
│   ├── services/
│   │   ├── OrderEngineService.ts      # Core business logic: pricing, BOM deduction, KDS routing
│   │   ├── InventoryService.ts        # Inventory, recipe linking, PO approvals and receipts
│   │   ├── FoodCostAnalyticsService.ts# Theoretical BOM costing, revenue, and stock variance
│   │   ├── AttendancePayrollService.ts# Haversine GPS geofence, overtime, and CSV payroll export
│   │   ├── TaxEInvoicingService.ts    # Configurable tax engine, sequential fiscal receipts & UBL XML
│   │   ├── PaymentService.ts          # Penny-accurate split checks, Stripe Terminal & webhooks
│   │   ├── ManagerOverrideService.ts  # Manager PIN verification, audit overrides, voids/comps/refunds
│   │   └── ThermalPrinterService.ts   # 80mm ESC/POS formatting & local hardware print queue
│   ├── utils/
│   │   └── pinHasher.ts               # PBKDF2/scrypt cryptographic manager PIN hashing & timing-safe verify
│   ├── validation/
│   │   ├── orderSchemas.ts            # Zod validation schemas for POS orders
│   │   ├── inventorySchemas.ts        # Zod validation schemas for inventory, recipes & POs
│   │   ├── analyticsPayrollTaxSchemas.ts # Zod schemas for clock-in/out and tax rules
│   │   ├── paymentsHardwareSchemas.ts # Zod schemas for payments, splits, PIN overrides & print jobs
│   │   └── saasBillingSchemas.ts      # Zod schemas for onboarding, billing webhooks & CSV menu commit
│   └── index.ts                       # Express application entrypoint
├── tests/
│   ├── orderIntegration.test.ts       # Comprehensive integration tests (isolation, pricing, shifts)
│   ├── inventoryBomConcurrency.test.ts# Concurrency test (row-level locking, par levels, POs)
│   ├── analyticsPayrollTax.test.ts    # Analytics, geofence, overtime, tax rules & e-invoicing tests
│   ├── paymentsHardware.test.ts       # Split-checks, webhook idempotency, PIN overrides & ESC/POS tests
│   ├── offlineSync.test.ts            # Offline queue, IndexedDB caching, LAN printing & replay tests
│   ├── saasBillingOnboarding.test.ts  # Multi-step onboarding, Stripe webhooks, CSV menu importer tests
│   ├── crossTenantSecurityAudit.test.ts # Cross-tenant security audit (orders, menus, inventory, websockets)
│   ├── rushHourConcurrency.test.ts    # Rush-hour concurrency (20 simultaneous POS requests, zero negative stock)
│   └── endToEndFlow.test.ts           # Full Playwright E2E browser test (POS -> KDS -> Bump -> BOM -> Analytics)
├── package.json                       # Dependencies and build scripts
├── tsconfig.json                      # TypeScript compiler configuration
└── GEMINI.md                          # Architectural and schema documentation
```

### Naming Conventions

- **Directories**: Lowercase single words or kebab-case (`controllers`, `middleware`, `routes`, `services`, `validation`, `prisma`, `db`, `tests`).
- **Source Files**:
  - TypeScript modules/controllers/services: PascalCase with descriptive suffix (e.g. `OrderController.ts`, `OrderEngineService.ts`).
  - Middleware & Database Extensions: camelCase (e.g. `auth.ts`, `idempotency.ts`, `tenantPrisma.ts`, `orderSchemas.ts`).
  - Routes: camelCase with `Routes` suffix (e.g. `orderRoutes.ts`).
- **Database Mapping**:
  - Prisma Models: PascalCase (e.g. `Tenant`, `MenuItem`, `OrderTicket`, `InventoryItem`).
  - PostgreSQL Tables: snake_case pluralized via `@@map` (e.g. `tenants`, `menu_items`, `order_tickets`, `inventory_items`).
  - Column Names: camelCase in TypeScript/Prisma, mapped to snake_case via `@map` (e.g. `tenantId` -> `tenant_id`, `prepStation` -> `prep_station`).
  - Enums: PascalCase for enum names, UPPER_SNAKE_CASE for values (e.g. `StationType.SALAD_PANTRY`).

---

## 2. Prisma Models, Field Names & Enums

### Enums

| Enum Name | Values |
| :--- | :--- |
| `RoleType` | `ADMIN`, `MANAGER`, `SERVER`, `KITCHEN_STAFF` |
| `ShiftStatus` | `SCHEDULED`, `CLOCKED_IN`, `CLOCKED_OUT`, `NO_SHOW`, `CANCELLED` |
| `OrderStatus` | `PENDING`, `ACCEPTED`, `PREPARING`, `READY_FOR_PICKUP`, `SERVED`, `COMPLETED`, `CANCELLED` |
| `OrderItemStatus`| `PENDING`, `COOKING`, `READY`, `SERVED`, `VOIDED` |
| `StationType` | `GRILL`, `FRY`, `SAUTE`, `BAR`, `SALAD_PANTRY`, `PIZZA`, `PASTRY`, `EXPO` |
| `TicketStatus` | `QUEUED`, `IN_PREPARATION`, `READY`, `RECALLED`, `SERVED`, `VOIDED` |
| `AllergenType` | `DAIRY`, `EGG`, `FISH`, `SHELLFISH`, `TREE_NUTS`, `PEANUTS`, `WHEAT_GLUTEN`, `SOY`, `SESAME` |
| `PricingTierType`| `STANDARD`, `HAPPY_HOUR`, `EVENT_SPECIAL`, `DINE_IN`, `TAKEOUT`, `DELIVERY` |
| `IdempotencyStatus` | `IN_PROGRESS`, `COMPLETED`, `FAILED` |
| `PurchaseOrderStatus` | `DRAFT`, `SUBMITTED`, `APPROVED`, `RECEIVED`, `CANCELLED` |
| `ShiftLogStatus` | `ACTIVE`, `COMPLETED`, `FLAGGED` |
| `PaymentStatus` | `PENDING`, `PROCESSING`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `REFUNDED`, `PARTIALLY_REFUNDED` |
| `PaymentMethod` | `CARD_PRESENT`, `CASH`, `STRIPE_TERMINAL`, `EXTERNAL_CARD` |
| `SplitType` | `FULL`, `EQUAL`, `SEAT`, `ITEM` |
| `RefundStatus` | `PENDING`, `SUCCEEDED`, `FAILED` |
| `PrinterType` | `CUSTOMER_RECEIPT`, `KITCHEN_PREP`, `BAR` |
| `PrintJobStatus` | `QUEUED`, `PROCESSING`, `PRINTED`, `FAILED` |
| `AuditAction` | `CREATE`, `UPDATE`, `DELETE`, `ROLE_ASSIGN`, `ROLE_REVOKE`, `INVENTORY_ADJUST`, `OVERRIDE` |
| `AuditEntityType` | `USER_ROLE`, `INVENTORY_ITEM`, `MENU_ITEM`, `ORDER`, `USER`, `LOCATION`, `INGREDIENT`, `PURCHASE_ORDER`, `RECIPE_ITEM`, `SUPPLIER`, `SHIFT_LOG`, `LOCATION_TAX_RULE`, `FISCAL_SEQUENCE`, `PAYMENT`, `REFUND`, `PRINT_JOB` |

---

### Core Models & Tenant Scoping

1. **`Tenant`** (`tenants`): Root tenant / restaurant enterprise record.
2. **`Location`** (`locations`): Branch/outlet scoped by `tenantId`. Includes `latitude`, `longitude`, and `geofenceRadiusMeters` (default 150m). `@@unique([tenantId, code])`.
3. **`User`** (`users`): Staff member scoped by `tenantId`. Includes `hourlyRate` (default $18.00) and cryptographic `managerPinHash`. `@@unique([tenantId, email])`.
4. **`UserRole`** (`user_roles`): Role assignment scoped by `tenantId` and optional `locationId`. `@@unique([tenantId, userId, role, locationId])`.
5. **`Shift`** (`shifts`): Schedule record scoped by `tenantId`, `locationId`, and `userId`.
6. **`ShiftLog`** (`shift_logs`): Staff clock-in/out records with GPS coordinates, distance to geofence, regular/overtime hours, and gross pay. Scoped by `tenantId` and `locationId`.
7. **`Category`** (`categories`): Menu category scoped by `tenantId`. `@@unique([tenantId, name])`.
8. **`MenuItem`** (`menu_items`): Menu item with station routing, scoped by `tenantId`. `@@unique([tenantId, sku])`.
9. **`MenuItemPriceTier`** (`menu_item_price_tiers`): Pricing tiers scoped by `tenantId`. `@@unique([tenantId, menuItemId, tier])`.
10. **`ModifierGroup`** & **`Modifier`**: Add-ons and options scoped by `tenantId`.
11. **`InventoryItem`** (`inventory_items`): Per-location stock with optimistic lock `version`, scoped by `tenantId`. `@@unique([tenantId, locationId, menuItemId])`.
12. **`Order`** (`orders`): Customer order scoped by `tenantId`. Includes `fiscalReceiptNumber`, `taxBreakdown`, `discountAmount`, `discountPercentage`, `isComped`, `compReason`, `voidReason`. `@@unique([tenantId, locationId, orderNumber])`.
13. **`OrderItem`** (`order_items`): Line items referencing `orderId` and `menuItemId`.
14. **`OrderTicket`** (`order_tickets`): KDS station prep tickets scoped by `tenantId`. `@@unique([tenantId, ticketNumber])`.
15. **`OrderTicketItem`** (`order_ticket_items`): Station line item details with modifier summaries and allergen flags.
16. **`IdempotencyRecord`** (`idempotency_records`): Distributed idempotency store scoped by `tenantId`. `@@unique([tenantId, idempotencyKey])`.
17. **`AuditLog`** (`audit_logs`): Immutable audit trail scoped by `tenantId`.
18. **`Supplier`** (`suppliers`): Food & beverage vendor scoped by `tenantId`. `@@unique([tenantId, name])`.
19. **`Ingredient`** (`ingredients`): Raw inventory items scoped by `tenantId` & `locationId`. `@@unique([tenantId, locationId, sku])`.
20. **`RecipeItem`** (`recipe_items`): Bill of Materials linking `menuItemId` + optional `modifierId` to `ingredientId` & `quantity`.
21. **`PurchaseOrder`** (`purchase_orders`): Replenishment orders scoped by `tenantId` & `locationId`. `@@unique([tenantId, locationId, poNumber])`.
22. **`PurchaseOrderItem`** (`purchase_order_items`): Line items on purchase orders. `@@unique([purchaseOrderId, ingredientId])`.
23. **`LocationTaxRule`** (`location_tax_rules`): Configurable tax splits (e.g. State 4.5%, City 4.375%, Surcharge 1%) scoped by `tenantId` and `locationId`. `@@unique([tenantId, locationId, code])`.
24. **`FiscalSequence`** (`fiscal_sequences`): Atomic sequential counter per location and calendar year for fiscal invoices. `@@unique([tenantId, locationId, fiscalYear])`.
25. **`Payment`** (`payments`): Card and cash settlement records scoped by `tenantId`, `locationId`, and `orderId`. Linked to Stripe Terminal PaymentIntents, tracking amounts, tips, brand, last4, and split details. `@@unique([tenantId, stripePaymentIntentId])`.
26. **`Refund`** (`refunds`): Refund records scoped by `tenantId`, `locationId`, `orderId`, and `paymentId`, tracking refund amounts, reasons, and authorizing manager IDs.
27. **`PrintJob`** (`print_jobs`): ESC/POS raw byte payload queue for 80mm thermal printers, tracking print status, target station, attempts, and error logs. Scoped by `tenantId` and `locationId`.

---

## 3. Commercial Hardening Specifications

### A. Multi-Tenant Isolation: Defense-in-Depth

1. **Prisma Client Extension (`src/db/tenantPrisma.ts`)**:
   - Built using Prisma's `$extends` query lifecycle hooks.
   - Automatically injects `tenantId` into `findFirst`, `findMany`, `create`, `update`, and `delete` operations.
   - Throws `TenantIsolationViolationError` if cross-tenant identifiers are supplied.
2. **PostgreSQL Row-Level Security (RLS) (`src/db/enableRls.ts`)**:
   - `ROW LEVEL SECURITY` enabled across all tables.
   - Policies verify that `"tenant_id" = NULLIF(current_setting('app.current_tenant_id', true), '')::UUID`.
3. **Database-Level Unique Constraints**:
   - Natural keys (`sku`, `code`, `orderNumber`, `ticketNumber`, `idempotencyKey`, `poNumber`) are scoped strictly with composite `[tenantId, ...]` constraints.

### B. Server-Side Price & Tax Integrity

- **Zero-Trust Pricing Architecture**: The client payload only conveys intent (`menuItemId`, `quantity`, `selectedModifierIds`).
- **Atomic Calculation**: Inside a single `prisma.$transaction()`, the backend:
  1. Reads authoritative base prices from `menu_items` (or `menu_item_price_tiers`).
  2. Reads verified modifier upcharges from `modifiers`.
  3. Calculates unit price: $P_i = P_{\text{base}} + \sum \Delta M$.
  4. Calculates line subtotal: $S_i = P_i \cdot Q_i$.
  5. Computes order subtotal: $S = \sum S_i$.
  6. Computes authoritative tax (8.875%): $\tau = \text{round}(S \cdot 0.08875, 2)$.
  7. Computes grand total: $T = S + \tau + A_{\text{tip}}$.
- **Fraud & Tamper Detection**: If client sends `claimedUnitPrice`, `claimedSubtotal`, or `claimedTotalAmount` differing from server calculations by $> \$0.009$, the transaction rolls back immediately with HTTP 422 `PRICE_TAMPERING_DETECTED`.

### C. Shift-Window Edge Cases & Order Settlement

1. **New Table / Order Openings (`POST /api/v1/orders`)**:
   - Protected by `enforceShiftAccessWindow`.
   - Servers attempting to open new orders outside their scheduled shift window (+30m pre-shift, +45m post-shift) are rejected with HTTP 403 `OUTSIDE_SHIFT_WINDOW`.
2. **Active Open Order Settlement (`PATCH /api/v1/orders/:orderId/settle`)**:
   - Servers are explicitly permitted to settle, tip out, and close active open orders (`PENDING`, `ACCEPTED`, `PREPARING`, `READY_FOR_PICKUP`, `SERVED`) that were initiated during their shift, **even after their scheduled shift has ended**.
   - Attempting to re-settle an already `COMPLETED` order is rejected with HTTP 400 `SETTLEMENT_FAILED`.

### D. Recipe BOM, Pessimistic Row-Level Locking & Par-Level Automation

1. **PostgreSQL Row-Level Locking (`SELECT ... FOR UPDATE`)**:
   - During order placement, all ingredients referenced by the ordered `MenuItem`s and selected `Modifier`s are resolved via `RecipeItem`.
   - Unique ingredient IDs are sorted deterministically and locked using `SELECT ... FOR UPDATE` inside `prisma.$transaction()`.
   - If two POS terminals sell the last unit simultaneously, the second transaction waits for the first to commit, re-evaluates `current_stock < requiredAmount`, and immediately throws an error without overselling or negative inventory.
2. **Automated DRAFT Purchase Order Generation**:
   - If an ingredient's stock drops below its configured `parLevel`, the system checks for an existing `DRAFT` Purchase Order for that ingredient's default supplier at that location.
   - If found, it appends or updates the line item with `reorderQuantity`; if none exists, it automatically creates a new `DRAFT` PO with a unique sequence number.
3. **Atomic PO Receiving**:
   - Approving an order transitions status to `APPROVED`.
   - Receiving an approved order (`POST /api/v1/admin/purchase-orders/:id/receive`) uses `SELECT ... FOR UPDATE` to atomically restock each ingredient's `currentStock` by the received quantity and logs the audit event.

### E. Back-Office Analytics, Geofenced Attendance & E-Invoicing Engine

1. **Food-Cost Percentage & Theoretical vs Actual Variance**:
   - $\text{Food Cost } \% = \left(\frac{\text{Theoretical Ingredient Cost}}{\text{Net Revenue}}\right) \times 100\%$.
   - Aggregates theoretical usage from completed order line items mapped through BOM recipes (`RecipeItem`).
   - Cross-checks actual stock deductions from `AuditLog` events to report operational variance and dollar discrepancies per ingredient.
2. **Staff Attendance & Geofenced Clock-In**:
   - Validates staff GPS coordinates against the location's latitude/longitude using the spherical Haversine formula:
     $$d = 2R \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta \phi}{2}\right) + \cos(\phi_1)\cos(\phi_2)\sin^2\left(\frac{\Delta \lambda}{2}\right)}\right)$$
   - Blocks clock-in if $d > \text{geofenceRadiusMeters}$ (HTTP 403 `GEOFENCE_VIOLATION`).
3. **Overtime Calculation & Payroll Export**:
   - Work shifts $\le 8$ hours are billed at regular $\text{hourlyRate}$.
   - Hours exceeding 8 hours are billed at overtime rate ($1.5 \times \text{hourlyRate}$).
   - Generates downloadable, RFC-4180 compliant CSV payroll summaries.
4. **Multi-Split Tax Engine & Sequential Fiscal E-Invoicing**:
   - Supports configurable tax splits per location (e.g. State Sales Tax, City Tax, Hospitality Surcharge) stored in `LocationTaxRule`.
   - Generates sequential fiscal receipts per location and year (`FISCAL-{LOC}-{YEAR}-{000001}`) using atomic database upserts on `FiscalSequence`.
   - Produces cryptographic SHA-256 verification digests and compliant UBL XML invoice payloads (`urn:oasis:names:specification:ubl:schema:xsd:Invoice-2`).

### F. Commercial Payments, Manager PIN Overrides & Thermal Printer Bridge

1. **Split-Check Mathematics & Penny Remainder Distribution**:
   - Supports 3 split types:
     - `EQUAL`: Divides total, subtotal, and tax into $N$ integer cent shares, allocating remaining pennies to the initial checks to ensure:
       $$\sum s_i = S, \quad \sum \tau_i = T, \quad \sum (\text{checks}) = \text{Order Total}$$
     - `SEAT`: Organizes items by seat number, computing seat subtotal and distributing tax proportionally with exact remainder allocation.
     - `ITEM`: Splits individual menu items across multiple payers by percentage shares.
2. **Stripe Terminal Payment Intents & Webhook Idempotency**:
   - Creates card-present payment intents (`pi_term_*`) and enqueues `Payment` records in `PENDING` state.
   - Enforces balance limits ($A_{\text{paid}} \le \text{Order Remaining Balance}$).
   - Webhook processor handles `payment_intent.succeeded` idempotently; duplicate webhook events are recognized and return without double-crediting.
   - Upon full payment settlement, automatically marks `Order` as `COMPLETED` and bumps `OrderTicket`s to `SERVED`.
   - Tip adjustments post-authorization recalculate payment and order tips atomically.
3. **Manager PIN Controls & Audit Trail**:
   - Requires verified 4–6 digit numeric PIN with cryptographic PBKDF2 salt verification.
   - Restricts sensitive operational overrides:
     - **Voids**: Cancels order, sets open tickets to `VOIDED`, logs reason.
     - **Comps**: Sets subtotal, tax, and total to $0, marks order completed.
     - **Discounts**:
       - $\le 15\%$: Allowed for server without PIN.
       - $> 15\%$: Strictly requires Manager PIN (returns HTTP 403 `MANAGER_PIN_REQUIRED` / `INVALID_MANAGER_PIN` otherwise).
     - **Refunds**: Validates against remaining refundable balance, issues `Refund` in `SUCCEEDED`, marks `Payment` as `REFUNDED` or `PARTIALLY_REFUNDED`.
   - Every override writes an immutable `AuditLog` entry with `action: OVERRIDE`.
4. **Thermal Printer Bridge (80mm ESC/POS Formatting & Queue)**:
   - Formats customer receipts with restaurant branding, server, table, line items, modifiers, tax splits, tip, total, card info, and partial paper cut (`\x1DVB\x00`).
   - Formats kitchen prep tickets with double-height station headers (`\x1D!\x11`), item quantities, prominent modifiers, bold allergen callouts, and paper cut.
   - Exposes local hardware print queue endpoints (`/printers/queue`) for polling and acknowledges printing completion or failure.

### G. POS Offline Resilience, IndexedDB Caching & Background Reconnect Sync

1. **Client-Side IndexedDB Persistence Layer (`posOfflineStorage`)**:
   - Caches active menu items, pricing tiers, modifier groups, and location tax rules locally.
   - Securely caches authorized manager PIN hashes, allowing offline Manager PIN overrides (discounts, comps, voids) using isomorphic PBKDF2 verification.
2. **Offline Queue Manager & Local LAN Printing**:
   - Intercepts order creation requests. When WAN internet is unavailable or requests time out, orders are safely saved to the IndexedDB `orderQueue` store with a persistent UUID `idempotencyKey`.
   - Computes offline estimated subtotals, taxes, and totals from locally cached tax rules.
   - Immediately generates and queues 80mm ESC/POS prep tickets with station routing, item modifiers, allergen alerts, and paper cuts to local LAN/USB hardware printers so kitchen operations proceed uninterrupted.
   - Surfaces dynamic `"Offline Mode — Queued (N)"` status badges and banners in the UI.
3. **Automatic Background Sync Worker (`OfflineSyncWorker`)**:
   - Monitors network connectivity via `online` window events and background polling.
   - Upon reconnect, retrieves pending queued orders and replays them in **strict chronological order** ($T_1 \to T_2$) against `POST /api/v1/orders`.
   - Transmits the original `idempotencyKey` in headers and payload to eliminate duplicate orders or double-deductions.
   - Purges successfully synced orders from local storage upon server acknowledgment.

---

## 4. API Endpoints & Response Contracts

### Endpoints

| Method | Path | Required Role | Shift Enforced? | Description |
| :--- | :--- | :--- | :--- | :--- |
| `GET`  | `/api/v1/pos/bootstrap` | Staff | No | Downloads complete menu, modifiers, tax rules, and authorized staff hashes for local IndexedDB caching. |
| `POST` | `/api/v1/orders` | `ADMIN`, `MANAGER`, `SERVER` | Yes (Strict window) | Create order, validate menu/tiers/modifiers, deduct BOM inventory, route KDS tickets. |
| `PATCH`| `/api/v1/orders/:orderId/settle` | `ADMIN`, `MANAGER`, `SERVER` | No (Open orders can close) | Settle open order, record tip, transition tickets to `SERVED`. |
| `GET`  | `/api/v1/orders/:orderId` | Staff | No | Fetch order details (enforces tenant isolation). |
| `POST` | `/api/v1/admin/suppliers` | `ADMIN`, `MANAGER` | No | Create supplier record. |
| `POST` | `/api/v1/admin/ingredients` | `ADMIN`, `MANAGER` | No | Create ingredient with parLevel and supplier link. |
| `GET`  | `/api/v1/admin/ingredients` | `ADMIN`, `MANAGER` | No | List ingredients with stock levels and `isBelowParLevel` flag. |
| `PATCH`| `/api/v1/admin/ingredients/:id`| `ADMIN`, `MANAGER` | No | Update ingredient stock, thresholds, or cost. |
| `POST` | `/api/v1/admin/recipes` | `ADMIN`, `MANAGER` | No | Link `MenuItem` + optional `Modifier` to `Ingredient` quantity. |
| `GET`  | `/api/v1/admin/recipes/:menuItemId`| `ADMIN`, `MANAGER` | No | Get BOM recipe breakdown for a menu item. |
| `GET`  | `/api/v1/admin/purchase-orders` | `ADMIN`, `MANAGER` | No | List purchase orders by location and status. |
| `POST` | `/api/v1/admin/purchase-orders/:id/approve` | `ADMIN`, `MANAGER` | No | Transition DRAFT PO to APPROVED. |
| `POST` | `/api/v1/admin/purchase-orders/:id/receive` | `ADMIN`, `MANAGER` | No | Receive PO: atomically restock ingredients and transition to RECEIVED. |
| `GET`  | `/api/v1/analytics/food-cost` | `ADMIN`, `MANAGER` | No | Food-cost %, theoretical cost, and ingredient variance report. |
| `POST` | `/api/v1/attendance/clock-in` | Staff | No | Geofenced staff clock-in (validates GPS against location). |
| `POST` | `/api/v1/attendance/clock-out` | Staff | No | Staff clock-out with overtime calculation and gross pay tally. |
| `GET`  | `/api/v1/attendance/payroll/export` | `ADMIN`, `MANAGER` | No | RFC-4180 CSV payroll export with regular/overtime splits. |
| `POST` | `/api/v1/admin/tax-rules` | `ADMIN` | No | Configure location tax rules (VAT/GST/Sales tax splits). |
| `GET`  | `/api/v1/admin/tax-rules` | Staff | No | List active tax rules for a location. |
| `GET`  | `/api/v1/orders/:orderId/fiscal-receipt`| Staff | No | Formats sequential fiscal receipt (JSON or UBL XML). |
| `POST` | `/api/v1/payments/intent` | `ADMIN`, `MANAGER`, `SERVER` | No | Creates Stripe Terminal payment intent and records pending payment. |
| `POST` | `/api/v1/payments/webhook`| Public | No | Stripe webhook endpoint with idempotent settlement handling. |
| `POST` | `/api/v1/payments/:id/adjust-tip` | `ADMIN`, `MANAGER`, `SERVER` | No | Adjusts tip post-authorization before final batch capture. |
| `POST` | `/api/v1/orders/:orderId/split-check` | Staff | No | Calculates penny-accurate equal, seat, or item split checks. |
| `POST` | `/api/v1/admin/manager-pin` | `ADMIN`, `MANAGER` | No | Sets or updates user's 4-6 digit hashed Manager PIN. |
| `POST` | `/api/v1/orders/:orderId/void` | Staff (PIN req) | No | Voids order and open tickets, logging manager override. |
| `POST` | `/api/v1/orders/:orderId/comp` | Staff (PIN req) | No | Comps order to $0 and settles, logging manager override. |
| `POST` | `/api/v1/orders/:orderId/discount` | Staff (>15% PIN req)| No | Applies discount (<=15% without PIN, >15% requires Manager PIN). |
| `POST` | `/api/v1/payments/:paymentId/refund` | Staff (PIN req) | No | Processes partial/full refund, logging manager override. |
| `POST` | `/api/v1/printers/jobs` | Staff | No | Formats ESC/POS receipt or kitchen prep ticket and queues job. |
| `GET`  | `/api/v1/printers/queue`| Staff / Bridge | No | Fetches pending print queue jobs and transitions to PROCESSING. |
| `POST` | `/api/v1/printers/jobs/:id/complete`| Staff / Bridge | No | Acknowledges print job completion, setting status to PRINTED. |
| `POST` | `/api/v1/printers/jobs/:id/fail`| Staff / Bridge | No | Reports print hardware failure, setting status to FAILED. |

### Error Codes & HTTP Status Mappings

| HTTP Status | Error Code | Description |
| :--- | :--- | :--- |
| `400 Bad Request` | `VALIDATION_ERROR` | Payload failed Zod schema validation. |
| `400 Bad Request` | `MISSING_LOCATION` | Required `X-Location-ID` header missing. |
| `400 Bad Request` | `SETTLEMENT_FAILED` | Attempted to settle already closed or invalid order. |
| `400 Bad Request` | `PAYMENT_INTENT_FAILED` | Payment amount exceeds remaining balance or invalid order status. |
| `401 Unauthorized` | `UNAUTHORIZED` | Missing or invalid Bearer token. |
| `403 Forbidden` | `OUTSIDE_SHIFT_WINDOW` | Server attempted new order creation outside shift window. |
| `403 Forbidden` | `GEOFENCE_VIOLATION` | Staff attempted clock-in outside the location geofence radius. |
| `403 Forbidden` | `MANAGER_PIN_REQUIRED` | Override (void, comp, discount >15%, refund) attempted without valid PIN. |
| `404 Not Found` | `ORDER_NOT_FOUND` | Order does not exist or belongs to another tenant. |
| `422 Unprocessable`| `PRICE_TAMPERING_DETECTED` | Client-provided prices deviate from DB calculation. |

---

## 5. Verification & Test Suite Summary

The test suites (`tests/orderIntegration.test.ts`, `tests/inventoryBomConcurrency.test.ts`, `tests/analyticsPayrollTax.test.ts`, `tests/paymentsHardware.test.ts`, and `tests/offlineSync.test.ts`) execute against live PostgreSQL and verify:

```
✓ tests/offlineSync.test.ts (5 tests)
  ✓ IndexedDB Persistence Layer & Local Caching
    ✓ Fetches POS bootstrap package from server and caches menu, tax rules, and staff PINs locally
  ✓ Offline Queue Manager & Local Thermal Prep Ticket Generation
    ✓ Enqueues order locally with UUID idempotencyKey and generates local 80mm ESC/POS ticket during network outage
    ✓ Correctly reflects "Offline Mode — Queued (N)" in the queue status badge
  ✓ Background Sync Worker & Chronological Reconnect Replay
    ✓ Replays queued orders in strict chronological order upon reconnect without duplicate orders
    ✓ Protects against duplicate creation when replaying an already processed idempotencyKey

✓ tests/paymentsHardware.test.ts (16 tests)
  ✓ Split-Check Mathematical Integrity (Penny-Accurate Remainder Allocation)
    ✓ Accurately splits check equally across 3 payers without penny drift
    ✓ Splits check by seat accurately with proportional tax allocation
    ✓ Splits check by item shares (e.g. 50/50 shared entree)
  ✓ Stripe Terminal Payments, Webhook Confirmation & Idempotency
    ✓ Creates a Stripe Terminal payment intent and records pending payment
    ✓ Rejects payment intent when amount exceeds remaining balance
    ✓ Confirms payment via Stripe Webhook and handles duplicate webhooks idempotently
    ✓ Allows tip adjustments post-payment
  ✓ Manager PIN Controls, Overrides & Audit Trail
    ✓ Permits discounts <= 15% without Manager PIN
    ✓ Blocks discounts > 15% without Manager PIN or with invalid PIN
    ✓ Allows discounts > 15% with valid Manager PIN and logs audit override
    ✓ Enforces Manager PIN for voiding an order and marks tickets voided
    ✓ Enforces Manager PIN for comps and sets total to $0
    ✓ Enforces Manager PIN for refunds and updates payment status
  ✓ Thermal Printer Bridge (80mm ESC/POS Receipts, Prep Tickets & Queue)
    ✓ Generates ESC/POS formatted customer receipt and enqueues print job
    ✓ Generates ESC/POS kitchen prep ticket with double-sized station badge and allergen callout
    ✓ Supports local hardware bridge print-queue polling and lifecycle transitions

✓ tests/orderIntegration.test.ts (9 tests)
  ✓ Multi-Tenant Data Isolation
    ✓ Tenant A server can create an order successfully
    ✓ Tenant B server CANNOT fetch Tenant A order (returns 404)
    ✓ Tenant A server CAN fetch their own order (returns 200)
  ✓ Server-Side Price & Tax Integrity
    ✓ Rejects order when client attempts price tampering (submits low price)
    ✓ Rejects order when client claims fraudulent order total
    ✓ Authoritatively computes base price, modifiers, and 8.875% tax in single transaction
  ✓ Shift-Window Edge Cases
    ✓ Blocks opening a NEW order outside scheduled shift hours
    ✓ Allows an active open order to be settled even after the shift ended
    ✓ Rejects attempting to re-settle an already closed order

✓ tests/inventoryBomConcurrency.test.ts (7 tests)
  ✓ PostgreSQL Row-Level Locking (SELECT ... FOR UPDATE) Under High Concurrency
    ✓ Accurately handles two simultaneous POS orders competing for the LAST available unit
    ✓ Automatically generates a DRAFT Purchase Order when ingredient drops below par level
  ✓ Admin Inventory API & Purchase Order Approval / Receipt
    ✓ Admin can list ingredients and view the below-par alert flag
    ✓ Admin can list purchase orders and verify draft PO
    ✓ Admin approves the draft purchase order
    ✓ Admin receives the purchase order and atomically restocks inventory
    ✓ Server can now order again because inventory was successfully restocked

✓ tests/analyticsPayrollTax.test.ts (7 tests)
  ✓ Configurable Tax Engine & Sequential E-Invoicing
    ✓ Configures multi-split tax rules per location (State 4.5% + City 4.375% + Surcharge 1%)
    ✓ Generates compliance-ready JSON and XML fiscal receipts with sequential numbers
  ✓ Food Cost Analytics & Recipe Theoretical Costing
    ✓ Calculates net revenue, theoretical ingredient cost, and food-cost percentage accurately
  ✓ Staff Attendance, GPS Geofence Verification & Payroll Export
    ✓ Blocks clock-in if staff coordinates are outside the geofence radius
    ✓ Allows clock-in when staff is inside the geofence radius
    ✓ Calculates regular and overtime pay accurately upon clock-out
    ✓ Exports compliant payroll CSV report for admin download

✓ tests/saasBillingOnboarding.test.ts (12 tests)
  ✓ Self-Serve Multi-Step Onboarding Flow
    ✓ Atomically provisions Tenant, Location, Admin User, and Tax Configuration in one transaction
    ✓ Rejects duplicate tenant registration with 409 Conflict
    ✓ Rejects registration with invalid PIN format (< 4 digits or non-numeric)
  ✓ Stripe Billing Webhook & Subscription Enforcement
    ✓ Handles customer.subscription.created/updated to upgrade plan and expand max locations
    ✓ Fetches current subscription details via GET /api/v1/billing/subscription
    ✓ Handles invoice.payment_failed to suspend tenant and block operational POS endpoints
    ✓ Handles invoice.payment_succeeded to reactivate tenant and re-enable operational endpoints
    ✓ Handles customer.subscription.deleted to cancel tenant subscription
  ✓ CSV Menu Importer & Preview Validation
    ✓ Rejects CSV with missing mandatory header columns in preview
    ✓ Detects multiple row errors: negative price, unknown prep station, and intra-file duplicate SKU
    ✓ Validates clean CSV with multiple categories, stations, modifiers, and allergens
    ✓ Rejects commit when any SKU already exists in the database with 409 SKU_COLLISION

✓ tests/crossTenantSecurityAudit.test.ts (14 tests)
  ✓ Order & Ticket Isolation
    ✓ Tenant A Server CANNOT read Tenant B order (returns 404)
    ✓ Tenant A Server CANNOT settle Tenant B order (returns 404)
    ✓ Tenant A Server CANNOT bump Tenant B KDS ticket (returns 404)
    ✓ Tenant A Server CANNOT create an order targeted at Tenant B location
  ✓ Menu & Importer Isolation
    ✓ Tenant A Admin CANNOT commit imported menu items into Tenant B location (returns 404)
    ✓ POS bootstrap endpoint returns ONLY Tenant A menu items, never Tenant B items
  ✓ Inventory & Recipe BOM Isolation
    ✓ Admin A listing ingredients sees ONLY Tenant A ingredients, never Tenant B
    ✓ Admin A CANNOT update Tenant B ingredient (returns 400 with UPDATE_FAILED)
    ✓ Admin A CANNOT read recipes for Tenant B menu item (returns error)
    ✓ Admin A CANNOT approve Tenant B purchase order (returns 400)
  ✓ WebSocket Real-Time Channel Scoping & Isolation
    ✓ Rejects unauthenticated socket connections with AUTHENTICATION_ERROR
    ✓ Blocks Tenant A socket from subscribing to Tenant B room (emits UNAUTHORIZED_ROOM)
    ✓ Tenant A socket DOES NOT receive tickets broadcast for Tenant B
    ✓ Tenant A socket CANNOT bump Tenant B ticket via websocket event

✓ tests/rushHourConcurrency.test.ts (2 tests)
  ✓ 20 concurrent POS orders competing for 5 shared units results in exactly 5 successes, 15 graceful rejections, and ZERO negative stock
  ✓ 20 concurrent requests across TWO different menu items sharing a single ingredient strictly allocate stock

✓ tests/endToEndFlow.test.ts (1 test)
  ✓ Executes full E2E lifecycle: POS Order -> KDS Appearance -> Touch Bump -> Inventory Deduction -> Food-Cost Analytics Update

Test Files  9 passed (9)
Tests       73 passed (73)
```
