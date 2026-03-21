# Master Execution Plan — Taager Procurement System v2

> Finalized execution plan. Individual specs (DB, Backend, Frontend) will be extracted from this document.

---

## Clarifications

### Session 2026-03-22

- Q: Single-item or multi-item Purchase Requests? → A: Single-item per PR (simpler, one product per PR)
- Q: Who selects suppliers for RFQ? → A: Fully automatic — system sends to top N suppliers, no human step
- Q: Google Sheets authentication method? → A: Service Account (JSON key file) — server-to-server
- Q: Real-time dashboard update mechanism? → A: Polling via React Query (refetchInterval 10-30s)
- Q: Deployment target infrastructure? → A: Single VPS with Docker Compose (all services on one server)

---

## 1. Updated Workflow (8 Steps)

```
Step 1 → Step 2 → Step 3 → Step 4 → Step 5 → Step 6 → Step 7 → Step 8

┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ 1. PR    │──▶│ 2. Match │──▶│ 3. Send  │──▶│ 4. Collect│
│ Purchase │   │ Supplier │   │ RFQ via  │   │ Quotations│
│ Request  │   │ by       │   │ WhatsApp │   │ Auction   │
│ (GSheets)│   │ History  │   │ (Free)   │   │ Dashboard │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
                                                    │
┌──────────┐   ┌──────────┐   ┌──────────┐         │
│ 8. Send  │◀──│ 7. Get   │◀──│ 6. Export │◀────────┘
│ PO to    │   │ Financial│   │ Approved  │   ┌──────────┐
│ Supplier │   │ Approval │   │ Quotation │──▶│ 5. Mgr   │
│ WhatsApp │   │ (Odoo OR │   │ to Odoo   │   │ Approval │
└──────────┘   │ GSheets) │   │ OR GSheet │   └──────────┘
               └──────────┘   └──────────┘
```

### Step-by-Step Detail

| # | Step | Input | Process | Output | Data Store |
|---|------|-------|---------|--------|------------|
| 1 | Purchase Request | Supply chain team OR auto from Google Sheet | User fills PR form → saves to DB first → then syncs to Google Sheets | New PR row | PostgreSQL (first) + Google Sheets (sync) |
| 2 | Supplier Matching | PR product details | System queries purchase history, scores suppliers by frequency, price, delivery | Ranked supplier list with scores | PostgreSQL (query) |
| 3 | Send RFQ via WhatsApp | Top N ranked suppliers + PR details (auto-selected) | System auto-sends WhatsApp message to top-ranked suppliers for this product | RFQ records + WhatsApp messages sent | PostgreSQL → Google Sheets |
| 4 | Collect Quotations | Supplier WhatsApp replies OR manual entry | Parse responses or officer enters manually → auction dashboard | Quotation rows per PR | PostgreSQL → Google Sheets |
| 5 | Manager Approval | Best quotation submitted by officer | Manager reviews in dashboard → approve/reject | Approval decision | PostgreSQL → Google Sheets |
| 6 | Export to Odoo / GSheet | Approved quotation | Create PO in Odoo via XML-RPC OR log to Google Sheet (configurable) | Draft PO | Odoo + PostgreSQL → Google Sheets |
| 7 | Financial Approval | Draft PO in Odoo or approval column in Google Sheet | Finance team approves in Odoo native workflow OR marks approved in Google Sheet | PO approved | Odoo OR Google Sheets → PostgreSQL |
| 8 | Send PO to Supplier | Approved PO | Send WhatsApp confirmation to winning supplier | PO confirmed + supplier notified | PostgreSQL → Google Sheets |

### Product Unavailability Handling

```
Supplier responds "not available"
    │
    ▼
Mark this RFQ as "unavailable"
    │
    ▼
Are there other suppliers who haven't responded yet?
    ├── YES → Wait for remaining responses
    │
    └── NO → Check: any quotations received from other suppliers?
              ├── YES → Continue to Step 4 (auction) with available quotes
              │
              └── NO → All suppliers said unavailable
                        │
                        ▼
                  Auto-send RFQ to next best supplier(s)
                  from the ranked list (who weren't in the first batch)
                        │
                        ▼
                  Any more suppliers available in the system?
                        ├── YES → Send RFQ, repeat cycle
                        │
                        └── NO → All suppliers exhausted
                                  │
                                  ▼
                            PR status → "no_supplier_available"
                            Notify Officer to:
                              - Search for new suppliers manually
                              - Add new supplier to system
                              - Re-trigger RFQ with new supplier
                              - OR put PR on hold / cancel
```

**Key rules:**
- System auto-escalates to next-ranked suppliers (no manual intervention needed)
- Maximum 3 auto-escalation rounds (configurable in Sheet 8)
- Officer gets notified at each escalation ("Supplier X said unavailable, sending to Supplier Y")
- If ALL suppliers exhausted → PR enters "no_supplier_available" status
- Officer can add a new supplier and re-trigger RFQ from the PR detail page
- Unavailability responses are logged in purchase_history (useful for future scoring)

---

## 2. Google Sheets as Central Data Hub

### Architecture: Google Sheets + PostgreSQL (DB-First Write)

```
┌────────────────────────────────────────────────────────┐
│                PostgreSQL (Source of Truth)             │
│                                                        │
│  - All writes go here FIRST                            │
│  - Fast queries for dashboard                          │
│  - Complex joins for analytics                         │
│  - Audit trail                                         │
│  - Session/auth data                                   │
│  - Background job state                                │
│                                                        │
└───────────────┬──────────────┬─────────────────────────┘
                │  Bi-directional Sync  │
                │  (Google Sheets API)  │
                │  Auth: Service Account│
┌───────────────┴──────────────┴─────────────────────────┐
│              Google Sheets (Human-Readable Layer)       │
│                                                        │
│  Sheet 1: Purchase Requests                            │
│  Sheet 2: Suppliers                                    │
│  Sheet 3: Purchase History                             │
│  Sheet 4: RFQ Log                                      │
│  Sheet 5: Quotations                                   │
│  Sheet 6: Approvals                                    │
│  Sheet 7: Purchase Orders                              │
│  Sheet 8: Settings / Config                            │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### Sync Strategy (Constitution-Aligned)

```
Write Operations (App → Both):
  User action → Write to PostgreSQL FIRST → Then sync to Google Sheets
  (Constitution: "App writes → DB first → then Sheets")

Read Operations (App):
  Always read from PostgreSQL (fast)

Manual Sheet Edits (Sheets → DB):
  Celery beat polls Sheets every 5 min (configurable in Sheet 8)
  Detects changes by comparing timestamps/checksums
  Updates PostgreSQL accordingly

On Conflict:
  PostgreSQL wins → sync back to Sheets
  (Constitution: "On conflict: DB wins → sync back to Sheets")

Sync Failures:
  Logged in sync_log table and retried (never silently dropped)
  (Constitution: "Sync failures are logged and retried")
```

### Google Sheets Structure

**Sheet 1: purchase_requests**

| PR# | Product | SKU | Qty | Unit | Required Date | Urgency | Status | Requested By | Assigned Officer | Notes | Source | Created At |
|-----|---------|-----|-----|------|---------------|---------|--------|-------------|-----------------|-------|--------|------------|

**Sheet 2: suppliers**

| ID | Name | Contact | WhatsApp | Email | Categories | Active | Avg Score | Total Orders | Odoo Partner ID |
|----|------|---------|----------|-------|------------|--------|-----------|-------------|-----------------|

**Sheet 3: purchase_history**

| Product | SKU | Supplier | Qty | Unit Price | Total | Date | Required Date | Delivery Date | Payment Terms |
|---------|-----|----------|-----|------------|-------|------|---------------|---------------|---------------|

**Sheet 4: rfq_log**

| RFQ# | PR# | Supplier | WhatsApp Status | Sent At | Deadline | Reminder Sent | Response Status |
|------|-----|----------|-----------------|---------|----------|---------------|-----------------|

**Sheet 5: quotations**

| Quote# | PR# | Supplier | Unit Price | Total | Delivery Days | Payment Terms | Recommended | Submitted By | Status |
|--------|-----|----------|------------|-------|---------------|---------------|-------------|-------------|--------|

**Sheet 6: approvals**

| PR# | Quote# | Approver | Decision | Reason | Decided At |
|-----|--------|----------|----------|--------|------------|

**Sheet 7: purchase_orders**

| PO# | PR# | Quote# | Supplier | Total | Odoo PO ID | Odoo Status | Financial Approved | Supplier Notified | Created At |
|-----|-----|--------|----------|-------|------------|-------------|-------------------|-------------------|------------|

**Sheet 8: settings**

| Key | Value | Description |
|-----|-------|-------------|
| rfq_deadline_hours | 48 | Default RFQ response deadline |
| reminder_before_hours | 24 | Send reminder X hours before deadline |
| sync_interval_minutes | 5 | How often to sync Sheets ↔ DB |
| score_weight_frequency | 0.25 | Supplier scoring weight |
| score_weight_price | 0.35 | Supplier scoring weight |
| score_weight_delivery | 0.25 | Supplier scoring weight |
| score_weight_recency | 0.15 | Supplier scoring weight |
| approval_mode | odoo | "odoo" or "gsheet" |
| max_escalation_rounds | 3 | Max auto-escalation attempts |
| auto_rfq_top_n | 5 | Number of top suppliers to auto-send RFQ |
| odoo_url | https://taager.odoo.com | Odoo instance |
| dashboard_poll_seconds | 15 | Dashboard polling interval |

---

## 3. WhatsApp — Evolution API (MVP) → Meta Cloud API (Production)

### Phase 1: Evolution API (MVP — Free)

**What:** Open-source WhatsApp integration via WhatsApp Web protocol (Baileys library).

**Setup (Docker Compose — included in main deployment):**
```yaml
evolution-api:
  image: atendai/evolution-api
  ports:
    - "8080:8080"
  environment:
    - AUTHENTICATION_API_KEY=${EVOLUTION_API_KEY}
```

**Risks & Mitigation:**
- Risk of number banning → rate limit: max **15 messages/hour** (constitution)
- Protocol changes → have Meta Cloud API ready as backup
- Dedicated business phone number required

### Phase 2: Meta Cloud API (Production)

- All inbound service conversations = **free** (since Nov 2024)
- Outbound utility templates ≈ $0.01/message
- ~50 suppliers × 3 messages each ≈ **$1.50/month**
- Requires Meta Business Account verification + template approval

### All WhatsApp Messages Logged

Per constitution: "All messages logged in DB + Google Sheets"
- Every sent/received message stored in `whatsapp_messages` table
- Synced to Google Sheets for team visibility

---

## 4. Specs Structure (3 Separate Specs)

```
specs/
├── 001-procurement-v2/
│   ├── spec.md              # Full workflow spec
│   └── plan.md              # Technical implementation plan
│
├── spec-database/
│   ├── spec.md              # Database schema, sync engine
│   ├── plan.md              # DB technical decisions
│   └── tasks.md             # DB tasks (DB-001 → DB-009)
│
├── spec-backend/
│   ├── spec.md              # API contracts, services, integrations
│   ├── plan.md              # Backend architecture decisions
│   └── tasks.md             # Backend tasks (BE-001 → BE-023)
│
└── spec-frontend/
    ├── spec.md              # UI/UX requirements, pages
    ├── plan.md              # Frontend architecture decisions
    └── tasks.md             # Frontend tasks (FE-001 → FE-017)
```

---

## 5. Spec 1: Database (spec-database/)

### 5.1 Scope

- PostgreSQL schema design
- Google Sheets structure design
- Bi-directional sync mechanism (Service Account auth)
- Data migration from existing Google Sheet
- Seed data

### 5.2 Tables

#### users
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL CHECK (role IN ('requester', 'officer', 'manager', 'admin')),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### suppliers
```sql
CREATE TABLE suppliers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    contact_name VARCHAR(255),
    whatsapp_number VARCHAR(20) NOT NULL,  -- E.164 format
    email VARCHAR(255),
    categories TEXT[],                      -- Stored as comma-separated in GSheets
    is_active BOOLEAN DEFAULT true,
    avg_response_hours DECIMAL(5,1),
    avg_score DECIMAL(5,2),
    total_orders INTEGER DEFAULT 0,
    odoo_partner_id INTEGER,
    gsheet_row INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### purchase_history
```sql
CREATE TABLE purchase_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_name VARCHAR(255) NOT NULL,
    product_sku VARCHAR(100),
    supplier_id UUID REFERENCES suppliers(id),
    quantity DECIMAL(12,2) NOT NULL,
    unit VARCHAR(20),
    unit_price DECIMAL(12,2) NOT NULL,
    total_price DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'SAR',
    purchase_date DATE NOT NULL,
    required_date DATE,                     -- Added: needed for delivery score
    delivery_date DATE,
    payment_terms VARCHAR(100),
    source VARCHAR(20) DEFAULT 'imported',
    gsheet_row INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### purchase_requests
```sql
CREATE TABLE purchase_requests (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pr_number VARCHAR(20) UNIQUE NOT NULL,  -- PR-2026-0001
    product_name VARCHAR(255) NOT NULL,     -- Single item per PR
    product_sku VARCHAR(100),
    quantity DECIMAL(12,2) NOT NULL,
    unit VARCHAR(20) DEFAULT 'pcs',
    required_date DATE NOT NULL,
    urgency VARCHAR(10) DEFAULT 'medium' CHECK (urgency IN ('low', 'medium', 'high', 'critical')),
    status VARCHAR(30) DEFAULT 'pending' CHECK (status IN (
        'pending', 'matching', 'rfq_sent', 'quotations_received',
        'submitted_for_approval', 'approved', 'rejected',
        'no_supplier_available', 'on_hold',
        'in_odoo', 'financially_approved', 'po_confirmed', 'cancelled'
    )),
    escalation_round INTEGER DEFAULT 0,     -- Track auto-escalation count
    requested_by UUID REFERENCES users(id),
    assigned_officer UUID REFERENCES users(id),
    notes TEXT,
    source VARCHAR(20) DEFAULT 'manual',
    gsheet_row INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### rfqs
```sql
CREATE TABLE rfqs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rfq_number VARCHAR(20) UNIQUE NOT NULL,
    pr_id UUID REFERENCES purchase_requests(id) NOT NULL,
    supplier_id UUID REFERENCES suppliers(id) NOT NULL,
    whatsapp_message_id VARCHAR(100),
    sent_at TIMESTAMPTZ,
    deadline TIMESTAMPTZ NOT NULL,
    reminder_sent BOOLEAN DEFAULT false,
    escalation_round INTEGER DEFAULT 1,     -- Which round this was sent in
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
        'pending', 'sent', 'delivered', 'read', 'responded', 'unavailable', 'expired'
    )),
    gsheet_row INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### quotations
```sql
CREATE TABLE quotations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quote_number VARCHAR(20) UNIQUE NOT NULL,
    rfq_id UUID REFERENCES rfqs(id),
    pr_id UUID REFERENCES purchase_requests(id) NOT NULL,
    supplier_id UUID REFERENCES suppliers(id) NOT NULL,
    unit_price DECIMAL(12,2) NOT NULL,
    total_price DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'SAR',
    delivery_days INTEGER,
    payment_terms VARCHAR(100),
    valid_until DATE,
    notes TEXT,
    is_recommended BOOLEAN DEFAULT false,
    submitted_by VARCHAR(20) DEFAULT 'manual',
    status VARCHAR(20) DEFAULT 'received' CHECK (status IN (
        'received', 'recommended', 'submitted', 'approved', 'rejected'
    )),
    gsheet_row INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### approvals
```sql
CREATE TABLE approvals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pr_id UUID REFERENCES purchase_requests(id) NOT NULL,
    quotation_id UUID REFERENCES quotations(id) NOT NULL,
    approver_id UUID REFERENCES users(id) NOT NULL,
    decision VARCHAR(20) DEFAULT 'pending' CHECK (decision IN (
        'pending', 'approved', 'rejected', 'info_requested'
    )),
    reason TEXT,
    decided_at TIMESTAMPTZ,
    gsheet_row INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### purchase_orders
```sql
CREATE TABLE purchase_orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    po_number VARCHAR(20) UNIQUE NOT NULL,
    pr_id UUID REFERENCES purchase_requests(id) NOT NULL,
    quotation_id UUID REFERENCES quotations(id) NOT NULL,
    supplier_id UUID REFERENCES suppliers(id) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'SAR',
    odoo_po_id INTEGER,
    odoo_status VARCHAR(20),
    financial_approval_source VARCHAR(20) DEFAULT 'odoo',
    financial_approved BOOLEAN DEFAULT false,
    financial_approved_at TIMESTAMPTZ,
    financial_approved_by VARCHAR(255),
    supplier_notified_at TIMESTAMPTZ,
    supplier_acknowledged BOOLEAN DEFAULT false,
    gsheet_row INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

#### whatsapp_messages (NEW — constitution requirement)
```sql
CREATE TABLE whatsapp_messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    direction VARCHAR(10) NOT NULL CHECK (direction IN ('outbound', 'inbound')),
    whatsapp_number VARCHAR(20) NOT NULL,   -- E.164 format (masked in logs)
    supplier_id UUID REFERENCES suppliers(id),
    pr_id UUID REFERENCES purchase_requests(id),
    rfq_id UUID REFERENCES rfqs(id),
    message_type VARCHAR(20) NOT NULL,      -- rfq, reminder, po_confirmation, reply
    content TEXT NOT NULL,
    external_message_id VARCHAR(100),       -- Evolution API / Meta message ID
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN (
        'pending', 'sent', 'delivered', 'read', 'failed'
    )),
    error_message TEXT,
    gsheet_row INTEGER,
    sent_at TIMESTAMPTZ,
    received_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_wa_messages_supplier ON whatsapp_messages(supplier_id);
CREATE INDEX idx_wa_messages_pr ON whatsapp_messages(pr_id);
```

#### notifications (NEW)
```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) NOT NULL,
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    type VARCHAR(30) NOT NULL,              -- escalation, approval_needed, po_approved, etc.
    entity_type VARCHAR(50),                -- purchase_request, quotation, purchase_order
    entity_id UUID,
    is_read BOOLEAN DEFAULT false,
    read_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_notifications_user ON notifications(user_id, is_read);
```

#### sync_log (NEW — constitution requirement)
```sql
CREATE TABLE sync_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    direction VARCHAR(10) NOT NULL CHECK (direction IN ('db_to_sheet', 'sheet_to_db')),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID,
    sheet_name VARCHAR(50) NOT NULL,
    sheet_row INTEGER,
    status VARCHAR(20) NOT NULL CHECK (status IN ('success', 'failed', 'retrying')),
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    next_retry_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_sync_log_status ON sync_log(status, next_retry_at);
```

#### audit_log
```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL,
    old_value JSONB,
    new_value JSONB,
    user_id UUID REFERENCES users(id),
    ip_address VARCHAR(45),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);
```

### 5.3 Database Tasks

| Task ID | Task | Description | Depends On |
|---------|------|-------------|------------|
| DB-001 | PostgreSQL setup | Docker + init script + extensions (uuid-ossp) | — |
| DB-002 | Schema creation | All tables with constraints, indexes | DB-001 |
| DB-003 | Alembic setup | Migration framework + initial migration | DB-002 |
| DB-004 | Google Sheets structure | Create all 8 sheets with headers + formatting | — |
| DB-005 | Service Account setup | GCP project, enable Sheets API, create SA key | — |
| DB-006 | Sync engine core | Bi-directional sync service (DB-first write, Sheets ↔ DB) | DB-002, DB-004, DB-005 |
| DB-007 | Historical data import | Import existing purchase history from current Google Sheet | DB-002, DB-004 |
| DB-008 | Seed data | Sample users, suppliers for development | DB-002 |
| DB-009 | Sync testing | Test all sync scenarios (create, update, conflict, failure+retry) | DB-006 |

---

## 6. Spec 2: Backend (spec-backend/)

### 6.1 Scope

- FastAPI application
- All API endpoints
- Google Sheets API integration (Service Account)
- WhatsApp integration (Evolution API + Meta Cloud API)
- Odoo XML-RPC integration
- Supplier matching algorithm (auto-selects top N)
- Background jobs (Celery)
- Authentication + RBAC

### 6.2 Service Architecture

```
FastAPI App
├── api/v1/
│   ├── auth.py              # Login, refresh token
│   ├── purchase_requests.py # PR CRUD
│   ├── suppliers.py         # Supplier CRUD + scoring
│   ├── rfqs.py              # Auto-send RFQs, track status
│   ├── quotations.py        # Quote CRUD + comparison
│   ├── approvals.py         # Approval workflow
│   ├── purchase_orders.py   # PO management
│   ├── dashboard.py         # Analytics endpoints (polling)
│   ├── notifications.py     # Notification endpoints
│   └── webhooks.py          # WhatsApp webhook receiver
│
├── services/
│   ├── sheets_service.py    # Google Sheets API wrapper (Service Account)
│   ├── sync_service.py      # Bi-directional sync engine (DB-first)
│   ├── whatsapp_service.py  # Evolution API / Meta Cloud API
│   ├── odoo_service.py      # Odoo XML-RPC connector
│   ├── supplier_scoring.py  # Scoring + auto-select top N
│   ├── message_parser.py    # Parse supplier WhatsApp responses
│   ├── escalation_service.py # Unavailability auto-escalation
│   ├── approval_service.py  # Approval workflow logic
│   └── notification_service.py
│
├── tasks/                   # Celery background jobs
│   ├── sync_tasks.py        # GSheets ↔ DB sync (every 5 min)
│   ├── whatsapp_tasks.py    # Send messages, check status
│   ├── odoo_tasks.py        # PO creation, status polling
│   ├── reminder_tasks.py    # RFQ deadline reminders
│   ├── escalation_tasks.py  # Auto-escalation checks
│   └── scoring_tasks.py     # Recalculate supplier scores
│
└── middleware/
    ├── auth.py              # JWT verification (15 min expiry)
    └── rbac.py              # Role-based access
```

### 6.3 API Endpoints

#### Auth
```
POST   /api/v1/auth/login              # Returns JWT + refresh token
POST   /api/v1/auth/refresh            # Refresh JWT
GET    /api/v1/auth/me                 # Current user info
```

#### Purchase Requests
```
POST   /api/v1/purchase-requests       # Create PR (writes to DB first → GSheet)
GET    /api/v1/purchase-requests       # List (filterable: status, urgency, date)
GET    /api/v1/purchase-requests/:id   # Detail + timeline + linked RFQs/quotes
PATCH  /api/v1/purchase-requests/:id   # Update (syncs to GSheet)
POST   /api/v1/purchase-requests/:id/cancel  # Cancel PR
POST   /api/v1/purchase-requests/:id/trigger-rfq  # Auto-match + auto-send
```

#### Suppliers
```
GET    /api/v1/suppliers               # List (search, category filter)
GET    /api/v1/suppliers/:id           # Detail + history + score breakdown
POST   /api/v1/suppliers               # Add supplier
PATCH  /api/v1/suppliers/:id           # Update
GET    /api/v1/suppliers/recommend/:pr_id  # Scored recommendations for PR
```

#### RFQs
```
POST   /api/v1/rfqs/auto-send/:pr_id  # Auto-select top N + send WhatsApp
GET    /api/v1/rfqs/by-pr/:pr_id      # All RFQs for a PR
GET    /api/v1/rfqs/:id/status        # RFQ delivery/response status
```

#### Quotations
```
POST   /api/v1/quotations              # Manual quotation entry
GET    /api/v1/quotations/by-pr/:pr_id # All quotations for PR (comparison view)
POST   /api/v1/quotations/:id/recommend  # Officer marks as recommended
POST   /api/v1/quotations/:id/submit     # Submit for manager approval
```

#### Approvals
```
GET    /api/v1/approvals               # Pending approval queue (managers)
GET    /api/v1/approvals/:id           # Approval detail
POST   /api/v1/approvals/:id/approve   # Approve
POST   /api/v1/approvals/:id/reject    # Reject (reason required)
POST   /api/v1/approvals/:id/request-info  # Ask for more info
```

#### Purchase Orders
```
POST   /api/v1/purchase-orders/create-from-approval/:approval_id  # Step 6
GET    /api/v1/purchase-orders/:id     # PO detail + Odoo status
POST   /api/v1/purchase-orders/:id/export-odoo    # Export to Odoo
POST   /api/v1/purchase-orders/:id/send-to-supplier  # Step 8 (manual trigger)
```

#### Dashboard (Polling-based)
```
GET    /api/v1/dashboard/pipeline      # PR counts by status
GET    /api/v1/dashboard/analytics     # Spend, cycle time, trends
GET    /api/v1/dashboard/supplier-performance  # Response time, win rate
```

#### Notifications
```
GET    /api/v1/notifications           # User notifications (paginated)
GET    /api/v1/notifications/unread-count  # Badge count
POST   /api/v1/notifications/:id/read  # Mark as read
POST   /api/v1/notifications/read-all  # Mark all as read
```

#### Webhooks
```
POST   /api/v1/webhooks/whatsapp       # Receive supplier WhatsApp responses
POST   /api/v1/webhooks/gsheet-change  # Google Sheets onChange trigger
```

### 6.4 WhatsApp Message Templates (English)

**RFQ Request:**
```
Hi {supplier_name},

We have a new purchase request:
Product: {product_name}
Quantity: {quantity} {unit}
Required delivery date: {delivery_date}

Please reply with:
1. Price per unit
2. Availability (available / not available)
3. Delivery timeline

Response deadline: {deadline}

Thanks — Taager Procurement Team
```

**Reminder:**
```
Reminder:

We haven't received your response for:
Product: {product_name}
Deadline: {deadline}

Please reply as soon as possible.

— Taager Procurement
```

**PO Confirmation:**
```
Purchase Order Confirmed

PO Number: {po_number}
Product: {product_name}
Quantity: {quantity}
Agreed Price: {price} SAR
Delivery Date: {delivery_date}

Please confirm receipt by replying to this message.

Thanks — Taager
```

### 6.5 Supplier Scoring Algorithm

```python
def calculate_supplier_score(supplier_id, product_sku, history_data, weights):
    """
    Score a supplier 0-100 for a specific product based on purchase history.
    Called during Step 2. Top N suppliers are auto-selected for RFQ.
    """
    supplier_history = [h for h in history_data if h.supplier_id == supplier_id]
    product_history = [h for h in supplier_history if h.product_sku == product_sku]
    all_product_history = [h for h in history_data if h.product_sku == product_sku]

    # 1. Frequency Score (0-1)
    if not all_product_history:
        frequency_score = 0
    else:
        max_freq = max(Counter(h.supplier_id for h in all_product_history).values())
        our_freq = len(product_history)
        frequency_score = our_freq / max_freq if max_freq > 0 else 0

    # 2. Price Score (0-1): lower = better
    if not product_history:
        price_score = 0.5
    else:
        avg_market = mean(h.unit_price for h in all_product_history)
        avg_supplier = mean(h.unit_price for h in product_history)
        price_ratio = avg_supplier / avg_market if avg_market > 0 else 1
        price_score = max(0, min(1, 2 - price_ratio))

    # 3. Delivery Score (0-1): on-time rate
    deliveries = [h for h in product_history
                  if h.delivery_date and h.required_date]
    if not deliveries:
        delivery_score = 0.5
    else:
        on_time = sum(1 for h in deliveries
                      if h.delivery_date <= h.required_date)
        delivery_score = on_time / len(deliveries)

    # 4. Recency Score (0-1): exponential decay, half-life ~6 months
    if not product_history:
        recency_score = 0
    else:
        last_purchase = max(h.purchase_date for h in product_history)
        days_ago = (date.today() - last_purchase).days
        recency_score = math.exp(-days_ago / 180)

    # Composite
    total = (
        weights['frequency'] * frequency_score +
        weights['price'] * price_score +
        weights['delivery'] * delivery_score +
        weights['recency'] * recency_score
    )
    return round(total * 100, 1)
```

### 6.6 Backend Tasks

| Task ID | Task | Description | Depends On |
|---------|------|-------------|------------|
| BE-001 | FastAPI scaffolding | Project structure, config, Docker | DB-002 |
| BE-002 | Auth + RBAC | JWT login (15 min), refresh tokens, role middleware | BE-001 |
| BE-003 | Google Sheets service | Read/write/append wrapper (Service Account) | BE-001, DB-005 |
| BE-004 | Sync service | DB-first write, bi-directional sync, failure logging | BE-003, DB-006 |
| BE-005 | PR API | CRUD + auto PR number + DB-first GSheet sync | BE-002, BE-004 |
| BE-006 | Supplier API | CRUD + search + GSheet sync | BE-002, BE-004 |
| BE-007 | Supplier scoring | Scoring algorithm + auto-select top N | BE-006, DB-007 |
| BE-008 | WhatsApp service | Evolution API integration (send/receive) | BE-001 |
| BE-009 | RFQ API | Auto-send to top N, track status | BE-005, BE-007, BE-008 |
| BE-010 | WhatsApp webhook | Receive + parse supplier responses | BE-008 |
| BE-011 | Message parser | Extract price/availability from text | BE-010 |
| BE-012 | Escalation service | Auto-send to next suppliers on unavailability (max 3 rounds) | BE-011, BE-007 |
| BE-013 | Quotation API | CRUD + comparison + recommend | BE-009 |
| BE-014 | Approval API | Submit/approve/reject workflow | BE-013 |
| BE-015 | Odoo connector | XML-RPC auth + CRUD | BE-001 |
| BE-016 | PO creation | Create PO in Odoo or GSheet (configurable) | BE-014, BE-015 |
| BE-017 | Financial approval polling | Check Odoo status OR GSheet column | BE-016 |
| BE-018 | PO → Supplier notification | Send WhatsApp PO confirmation | BE-017, BE-008 |
| BE-019 | Notification service | Create, list, mark-read for user notifications | BE-002 |
| BE-020 | Celery setup | Workers, beat, task registration | BE-001 |
| BE-021 | Background jobs | Sync, reminders, scoring, polling, escalation | BE-020 |
| BE-022 | Dashboard API | Pipeline, analytics, supplier perf (polling) | BE-005 thru BE-018 |
| BE-023 | Audit trail | Auto-log all state changes | BE-002 |
| BE-024 | Integration testing | End-to-end workflow test | All above |

---

## 7. Spec 3: Frontend (spec-frontend/)

### 7.1 Scope

- Next.js 14+ (App Router) with TypeScript strict
- English default + Arabic RTL toggle
- Tailwind CSS with RTL plugin
- Data fetching via React Query with polling (refetchInterval)
- State management with Zustand
- All pages responsive

### 7.2 Pages

| Page | Route | Roles | Description |
|------|-------|-------|-------------|
| Login | /login | All | Email + password |
| Dashboard | / | All | Pipeline kanban + analytics (polling) |
| PR List | /purchase-requests | All | Filterable table |
| PR Detail | /purchase-requests/:id | All | Full PR lifecycle + auto-RFQ trigger |
| Create PR | /purchase-requests/new | Requester, Officer, Admin | PR creation form |
| Supplier List | /suppliers | Officer, Manager, Admin | Searchable directory |
| Supplier Detail | /suppliers/:id | Officer, Manager, Admin | History + score breakdown |
| RFQ Management | /rfqs/:pr_id | Officer, Admin | Track auto-sent RFQs + responses |
| Quotation Comparison | /quotations/:pr_id | Officer, Manager, Admin | Auction dashboard |
| Approval Queue | /approvals | Manager, Admin | Pending approvals |
| PO Management | /purchase-orders | Officer, Manager, Admin | PO list + Odoo status |
| Admin Settings | /admin | Admin | Users, config, sync status |
| Notification Center | /notifications | All | All notifications |

### 7.3 Key UI Components

**Auction Dashboard (Quotation Comparison):**
- Side-by-side cards for each supplier quotation
- Columns: Supplier, Unit Price, Total, Delivery, Payment Terms
- Historical avg price as a reference line
- Green/red indicator (cheaper/more expensive than historical)
- "Recommend" toggle
- "Submit for Approval" button

**Pipeline Board (Dashboard):**
- Kanban-style board with columns per status
- PR cards moving through stages
- Click card → detail view
- Count badges per column
- Urgency color coding (critical = red, high = orange)
- Auto-refreshes via polling (every 15s configurable)

**Approval Card:**
- PR summary at top
- Officer recommendation highlighted
- All quotations in a mini comparison table
- Approve / Reject / Request Info buttons
- Rejection requires reason modal

### 7.4 Frontend Tasks

| Task ID | Task | Description | Depends On |
|---------|------|-------------|------------|
| FE-001 | Next.js scaffolding | App Router, Tailwind + RTL plugin, i18n (EN/AR) | — |
| FE-002 | Auth flow | Login page, JWT handling, protected routes | FE-001 |
| FE-003 | Layout + navigation | Sidebar, topbar, mobile responsive | FE-001 |
| FE-004 | API client | Axios wrapper, interceptors, error handling | FE-001 |
| FE-005 | PR list page | Table with filters, search, pagination (polling) | FE-002, FE-004 |
| FE-006 | PR creation form | All fields, validation, product autocomplete | FE-004 |
| FE-007 | PR detail page | Full lifecycle view with timeline + trigger RFQ | FE-005 |
| FE-008 | Supplier list + detail | Directory, search, score breakdown | FE-004 |
| FE-009 | RFQ tracking panel | Auto-sent RFQs status, escalation indicators | FE-007 |
| FE-010 | Quotation comparison (auction) | Side-by-side cards, historical price | FE-007 |
| FE-011 | Approval queue | Approval cards, approve/reject actions | FE-002 |
| FE-012 | PO management | PO list, Odoo status, export actions | FE-004 |
| FE-013 | Dashboard | Pipeline kanban, analytics charts (polling) | FE-004 |
| FE-014 | Notification center | Bell icon, dropdown, unread count, mark read | FE-002 |
| FE-015 | Admin panel | Users, settings, sync status | FE-002 |
| FE-016 | Arabic RTL toggle | RTL support when user switches to Arabic | FE-003 thru FE-015 |
| FE-017 | Mobile responsive | All pages responsive | FE-003 thru FE-015 |

---

## 8. Deployment (Docker Compose — Single VPS)

```yaml
# docker-compose.yml
version: "3.9"
services:
  postgres:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: taager_procurement
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  redis:
    image: redis:7-alpine

  backend:
    build: ./backend
    depends_on: [postgres, redis]
    environment:
      DATABASE_URL: postgresql+asyncpg://${DB_USER}:${DB_PASSWORD}@postgres/taager_procurement
      REDIS_URL: redis://redis:6379
      GOOGLE_SA_KEY_PATH: /secrets/gsa-key.json
      EVOLUTION_API_URL: http://evolution-api:8080
      # ... other env vars

  celery-worker:
    build: ./backend
    command: celery -A app.celery worker -l info
    depends_on: [postgres, redis]

  celery-beat:
    build: ./backend
    command: celery -A app.celery beat -l info
    depends_on: [redis]

  frontend:
    build: ./frontend
    depends_on: [backend]
    ports:
      - "3000:3000"

  evolution-api:
    image: atendai/evolution-api
    environment:
      AUTHENTICATION_API_KEY: ${EVOLUTION_API_KEY}

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    depends_on: [backend, frontend]

volumes:
  pgdata:
```

---

## 9. Execution Timeline

```
Week 1-2:  Database Spec (DB-001 → DB-009)
           ├── Agent: Claude Code + GLM4.7
           ├── Deliverable: Working database + GSheets + sync
           └── Milestone: Data layer complete ✓

Week 3-4:  Backend Core (BE-001 → BE-007)
           ├── Agent: Claude Code + GLM4.7
           ├── Deliverable: Auth, PR API, Supplier API, Scoring
           └── Milestone: Core APIs working ✓

Week 5-6:  Backend Integrations (BE-008 → BE-024)
           ├── Agent: Claude Code
           ├── Deliverable: WhatsApp, Odoo, approval, escalation
           └── Milestone: Full backend complete ✓

Week 7-9:  Frontend (FE-001 → FE-017)
           ├── Agent: AntiGravity + Gemini 3 Pro
           ├── Deliverable: Complete web application
           └── Milestone: UI complete ✓

Week 10:   Testing & Launch
           ├── Integration testing
           ├── UAT with supply chain team
           ├── Bug fixes
           └── Milestone: Production launch ✓
```

---

## 10. Risk Register

| Risk | Impact | Mitigation |
|------|--------|------------|
| Evolution API number banned | High | Dedicated number, 15 msg/hr limit, Meta Cloud API backup |
| Google Sheets API quota exceeded | Medium | Cache reads, batch writes, exponential backoff |
| Google Sheets concurrent edit conflicts | Medium | PostgreSQL is source of truth, overwrite GSheet on conflict |
| Odoo API unavailable | Medium | Queue POs locally, retry. GSheet fallback for financial approval |
| Supplier doesn't use WhatsApp | Low | Manual quotation entry always available |
| All suppliers say product unavailable | Medium | Auto-escalation 3 rounds. Officer adds new suppliers. PR on hold |
| Arabic RTL toggle bugs | Low | RTL only for toggle, use established RTL libraries |
| Historical data quality | High | Data cleaning script, fuzzy matching for supplier names |
| Sync engine failure | High | Failures logged in sync_log, auto-retry with backoff, alerts |
