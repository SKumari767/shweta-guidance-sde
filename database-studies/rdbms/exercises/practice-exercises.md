# Advanced Practice Exercises: RDBMS Concepts, DDL & DML (Modules 01 & 02)

> [!IMPORTANT]
> **Environment Requirement**: If solving these exercises online, you must use an online editor that specifically supports PostgreSQL (v14 or newer), rather than generic SQLite or MySQL sandboxes.
> 
> *Recommended free options: [DB-Fiddle](https://www.db-fiddle.com/) (select Postgres 15/16), [OneCompiler PostgreSQL](https://onecompiler.com/postgresql), or your local container via [db-setup.md](db-setup.md).*

Welcome to the advanced practice exercises for **Module 01 (Core RDBMS Concepts, Relational Theory, ACID & Normalization)** and **Module 02 (DDL, DML & PostgreSQL Data Types)**.

These exercises are designed to simulate **real-world production database engineering problems** encountered in high-scale systems, financial ledgers, and SaaS platforms. They move beyond basic syntax into rigorous schema design, relational algebra invariants, transaction safety, and edge-case handling.

---

## Exercise Roadmap

| # | Challenge Name | Primary Concepts Tested | Difficulty |
|---|---|---|---|
| **1** | [Complex Relational Decomposition & BCNF Architecture](#exercise-1-complex-relational-decomposition--bcnf-architecture) | Normal Forms (1NF -> BCNF), Functional Dependencies, Anomaly Elimination, Strict DDL | Hard |
| **2** | [Zero-Loss Double-Entry Ledger & ACID Invariants](#exercise-2-zero-loss-double-entry-ledger--acid-invariants) | ACID Properties, Multi-table Balance Invariants, WAL Durability, Check Constraints | Very Hard |
| **3** | [High-Throughput Inventory Reservation via Conditional UPSERT](#exercise-3-high-throughput-inventory-reservation-via-conditional-upsert) | `INSERT ... ON CONFLICT DO UPDATE`, `EXCLUDED`, Dynamic `RETURNING`, Race Condition Prevention | Hard |
| **4** | [Semi-Structured JSONB & Array Telemetry System](#exercise-4-semi-structured-jsonb--array-telemetry-system) | `JSONB` Operators (`->>`, `@>`, `jsonb_set`), `ARRAY` Manipulation, JSON Schema Check Constraints | Very Hard |
| **5** | [Production Incident Post-Mortem & Schema Code Review](#exercise-5-production-incident-post-mortem--schema-code-review) | IEEE 754 vs `NUMERIC`, `TIMESTAMPTZ` Pitfalls, Cascade Deletion Disasters, `NULLS NOT DISTINCT` | Hard |

---

## Exercise 1: Complex Relational Decomposition & BCNF Architecture

### Context & Problem Statement

A fast-growing multi-tenant SaaS company has been storing its subscription billing data in a single denormalized table called `legacy_subscription_invoicing`. Over time, the engineering team has encountered severe data anomalies:
- Updating a plan price required updating millions of historical invoice records.
- Deleting an unbilled customer canceled their entire plan catalog definition.
- Some rows store comma-separated lists of notification emails.

You are given the following denormalized relation and attributes:

```sql
Relation: legacy_subscription_invoicing (
    invoice_id,                  -- [Primary Key]
    tenant_id,
    sub_id,
    plan_code,
    plan_name,
    plan_monthly_rate,
    billing_country,
    tax_rate_pct,
    user_notification_emails,    -- Non-atomic comma-separated list
    billing_cycle_start,
    billing_cycle_end,
    amount_due,
    gateway_txn_id,
    payment_status
)
```

**Relational Notation:**
> **R** (<u>**invoice_id**</u>, `tenant_id`, `sub_id`, `plan_code`, `plan_name`, `plan_monthly_rate`, `billing_country`, `tax_rate_pct`, `user_notification_emails`, `billing_cycle_start`, `billing_cycle_end`, `amount_due`, `gateway_txn_id`, `payment_status`)  
> *(Note: <u>**invoice_id**</u> is the primary key)*

---

### Business Rules & Functional Dependencies (FDs)

1. **FD 1**: `invoice_id` -> `(all other single-valued attributes)`  
   *(Each invoice has a unique identifier determining all other fields)*
2. **FD 2**: `sub_id` -> `tenant_id`, `plan_code`, `billing_country`  
   *(A subscription belongs to one tenant, subscribes to one plan, and bills to one country)*
3. **FD 3**: `plan_code` -> `plan_name`, `plan_monthly_rate`  
   *(The plan code dictates the plan name and monthly base rate)*
4. **FD 4**: `billing_country` -> `tax_rate_pct`  
   *(Each country has a fixed statutory tax rate percentage)*
5. **FD 5**: `(sub_id, billing_cycle_start)` -> `invoice_id`, `billing_cycle_end`, `amount_due`, `payment_status`, `gateway_txn_id`  
   *(A subscription has at most one invoice per billing cycle start date)*
6. **FD 6**: `gateway_txn_id` -> `invoice_id`, `payment_status`  
   *(A payment gateway transaction identifier uniquely maps to an invoice attempt)*
7. **Multivalued Attribute**: `user_notification_emails` contains a comma-separated list of emails (e.g. `"alice@acme.com, bob@acme.com"`).

---

### Your Tasks

1. **Analyze Normalization Violations**:
   - Identify the violation of **1NF**.
   - Identify which functional dependencies cause violations of **2NF** and **3NF**.
   - Determine if there are any violations of **Boyce-Codd Normal Form (BCNF)** (every determinant must be a candidate key).
2. **Decompose into BCNF**:
   - Decompose relation `R` into a set of normalized relations that are in **BCNF**.
   - Ensure the decomposition is **lossless-join** and verify if all functional dependencies are preserved.
3. **Write Production-Grade PostgreSQL DDL**:
   - Implement the decomposed schema using standard PostgreSQL features:
     - Use `UUID DEFAULT gen_random_uuid()` or `BIGINT GENERATED ALWAYS AS IDENTITY` for primary keys.
     - Use appropriate data types (`NUMERIC(12, 4)` for currency/rates, `TIMESTAMPTZ` for timestamps, `VARCHAR(255)` / `TEXT` for strings).
     - Configure foreign keys with appropriate `ON DELETE` rules (`RESTRICT`, `CASCADE`, or `SET NULL`) with explicit justification.
     - Add `CHECK` constraints to ensure `billing_cycle_end > billing_cycle_start`, `tax_rate_pct BETWEEN 0 AND 100`, and `plan_monthly_rate >= 0`.

<details>
<summary>Reveal Solution & Explanation</summary>

### 1. Normalization Analysis

- **1NF Violation**: `user_notification_emails` contains non-atomic comma-separated values.
- **2NF & 3NF Violations**:
  - `plan_code -> plan_name, plan_monthly_rate` is a **transitive dependency** on `sub_id`. Changing a plan's price creates an update anomaly across all historical invoices or active subscriptions.
  - `billing_country -> tax_rate_pct` is a **transitive dependency**. If country tax rates change, updating old invoices risks corrupting financial history.
- **BCNF Violations**:
  - In `plan_code -> plan_name, plan_monthly_rate`, `plan_code` is not a superkey of the original relation.
  - In `billing_country -> tax_rate_pct`, `billing_country` is not a superkey.

---

### 2. Decomposed BCNF Schema

1. `tenants`: (`tenant_id`, `tenant_name`, `created_at`)
2. `plans`: (`plan_code`, `plan_name`, `monthly_rate_usd`, `is_active`)
3. `country_tax_rates`: (`country_code`, `tax_rate_pct`, `effective_from`)
4. `subscriptions`: (`sub_id`, `tenant_id`, `plan_code`, `billing_country`, `created_at`)
5. `subscription_notification_recipients`: (`sub_id`, `email`) - eliminates 1NF multivalued violation
6. `invoices`: (`invoice_id`, `sub_id`, `cycle_start`, `cycle_end`, `subtotal_amount`, `tax_amount`, `total_amount_due`, `payment_status`, `gateway_txn_id`, `created_at`)

---

### 3. PostgreSQL DDL Implementation

```sql
-- Clean up test tables if re-running
DROP TABLE IF EXISTS subscription_notification_recipients CASCADE;
DROP TABLE IF EXISTS invoices CASCADE;
DROP TABLE IF EXISTS subscriptions CASCADE;
DROP TABLE IF EXISTS country_tax_rates CASCADE;
DROP TABLE IF EXISTS plans CASCADE;
DROP TABLE IF EXISTS tenants CASCADE;

-- 1. Tenants table
CREATE TABLE tenants (
    tenant_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    company_name VARCHAR(150) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);

-- 2. Subscription Plans
CREATE TABLE plans (
    plan_code VARCHAR(30) PRIMARY KEY,
    plan_name VARCHAR(100) NOT NULL,
    monthly_rate_usd NUMERIC(10, 2) NOT NULL CHECK (monthly_rate_usd >= 0),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);

-- 3. Country Tax Rates
CREATE TABLE country_tax_rates (
    country_code CHAR(2) PRIMARY KEY,
    tax_rate_pct NUMERIC(5, 2) NOT NULL CHECK (tax_rate_pct >= 0 AND tax_rate_pct <= 100),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);

-- 4. Subscriptions
CREATE TABLE subscriptions (
    sub_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    tenant_id UUID NOT NULL REFERENCES tenants(tenant_id) ON DELETE RESTRICT,
    plan_code VARCHAR(30) NOT NULL REFERENCES plans(plan_code) ON DELETE RESTRICT,
    billing_country CHAR(2) NOT NULL REFERENCES country_tax_rates(country_code) ON DELETE RESTRICT,
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'past_due', 'canceled')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);

-- 5. 1NF Resolution: Subscription Notification Recipients
CREATE TABLE subscription_notification_recipients (
    sub_id UUID NOT NULL REFERENCES subscriptions(sub_id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    PRIMARY KEY (sub_id, email),
    CONSTRAINT chk_valid_email CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

-- 6. Invoices (Immutable financial transaction snapshot)
CREATE TABLE invoices (
    invoice_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    sub_id UUID NOT NULL REFERENCES subscriptions(sub_id) ON DELETE RESTRICT,
    cycle_start TIMESTAMPTZ NOT NULL,
    cycle_end TIMESTAMPTZ NOT NULL,
    subtotal_amount NUMERIC(12, 2) NOT NULL CHECK (subtotal_amount >= 0),
    tax_amount NUMERIC(12, 2) NOT NULL CHECK (tax_amount >= 0),
    total_amount_due NUMERIC(12, 2) NOT NULL CHECK (total_amount_due >= 0),
    payment_status VARCHAR(20) NOT NULL DEFAULT 'unpaid' 
        CHECK (payment_status IN ('unpaid', 'processing', 'paid', 'failed', 'refunded')),
    gateway_txn_id VARCHAR(100) UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    
    -- Invariants: End of cycle must be strictly after start, and total = subtotal + tax
    CONSTRAINT chk_valid_billing_window CHECK (cycle_end > cycle_start),
    CONSTRAINT chk_total_reconciliation CHECK (total_amount_due = subtotal_amount + tax_amount)
);
```

</details>

---

## Exercise 2: Zero-Loss Double-Entry Ledger & ACID Invariants

### Context & Problem Statement

In financial accounting engines (Stripe, Modern Treasury, banks), account balances are never simply updated with `UPDATE accounts SET balance = balance - 100`. Doing so loses the audit trail, makes concurrent reconciliations impossible, and violates consistency invariants.

Instead, production financial systems use **Double-Entry Bookkeeping**:
- Every financial movement is represented by a **Journal Entry**.
- A Journal Entry contains two or more **Posting Lines**.
- The fundamental accounting equation must hold for every entry:  
  `SUM(Debit amounts) - SUM(Credit amounts) = 0`
- Accounts have a non-negative balance invariant (or must respect a strict overdraft credit limit).

---

### Your Tasks

1. **Schema Design (DDL)**:
   - Create table `ledger_accounts`:
     - `account_id` (`UUID`, PK)
     - `holder_name` (`TEXT`)
     - `currency` (`CHAR(3)`, e.g. `'USD'`)
     - `balance` (`NUMERIC(14, 4)`, default `0.0000`)
     - `credit_limit` (`NUMERIC(14, 4)`, default `0.0000`)
     - Constraint: Enforce that `balance + credit_limit >= 0`.
     - `is_frozen` (`BOOLEAN`, default `FALSE`)
   - Create table `journal_entries`:
     - `entry_id` (`BIGINT GENERATED ALWAYS AS IDENTITY`, PK)
     - `reference_id` (`TEXT UNIQUE`) — idempotency key from caller
     - `description` (`TEXT NOT NULL`)
     - `created_at` (`TIMESTAMPTZ` default `clock_timestamp()`)
   - Create table `journal_lines`:
     - `line_id` (`BIGINT GENERATED ALWAYS AS IDENTITY`, PK)
     - `entry_id` (`BIGINT REFERENCES journal_entries ON DELETE CASCADE`)
     - `account_id` (`UUID REFERENCES ledger_accounts ON DELETE RESTRICT`)
     - `amount` (`NUMERIC(14, 4) NOT NULL CHECK (amount > 0)`)
     - `direction` (`VARCHAR(6) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT'))`)

2. **Transactional DML Challenge (ACID)**:  
   Write a single atomic PostgreSQL transaction block that transfers `$250.0000` from Account A to Account B:
   - Verifies neither account is frozen.
   - Updates both account balances safely.
   - Inserts the journal entry and both journal lines.
   - Explicitly handles the failure case (if Account A does not have enough balance, the transaction must abort without creating any phantom entry or orphan posting line).
   - Explain how PostgreSQL **Atomicity** and the **Write-Ahead Log (WAL)** guarantee that a sudden crash during execution cannot leave money created or destroyed.

<details>
<summary>Reveal Solution & Explanation</summary>

### 1. Schema Setup DDL

```sql
DROP TABLE IF EXISTS journal_lines CASCADE;
DROP TABLE IF EXISTS journal_entries CASCADE;
DROP TABLE IF EXISTS ledger_accounts CASCADE;

CREATE TABLE ledger_accounts (
    account_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    holder_name TEXT NOT NULL,
    currency CHAR(3) NOT NULL,
    balance NUMERIC(14, 4) NOT NULL DEFAULT 0.0000,
    credit_limit NUMERIC(14, 4) NOT NULL DEFAULT 0.0000 CHECK (credit_limit >= 0),
    is_frozen BOOLEAN NOT NULL DEFAULT FALSE,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    
    -- Absolute solvency constraint: Account balance cannot exceed allowed credit limit
    CONSTRAINT chk_solvency CHECK (balance + credit_limit >= 0)
);

CREATE TABLE journal_entries (
    entry_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    reference_id TEXT NOT NULL UNIQUE,
    description TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()
);

CREATE TABLE journal_lines (
    line_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    entry_id BIGINT NOT NULL REFERENCES journal_entries(entry_id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES ledger_accounts(account_id) ON DELETE RESTRICT,
    amount NUMERIC(14, 4) NOT NULL CHECK (amount > 0),
    direction VARCHAR(6) NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT'))
);

-- Seed two test accounts
INSERT INTO ledger_accounts (account_id, holder_name, currency, balance, credit_limit)
VALUES 
    ('11111111-1111-1111-1111-111111111111', 'Alice Cooper', 'USD', 500.0000, 0.0000),
    ('22222222-2222-2222-2222-222222222222', 'Bob Smith', 'USD', 100.0000, 0.0000);
```

---

### 2. Atomic Transfer DML

```sql
DO $$
DECLARE
    v_acc_sender UUID := '11111111-1111-1111-1111-111111111111';
    v_acc_receiver UUID := '22222222-2222-2222-2222-222222222222';
    v_transfer_amount NUMERIC(14, 4) := 250.0000;
    v_ref TEXT := 'TXN-REQ-987456';
    v_entry_id BIGINT;
    v_sender_frozen BOOLEAN;
    v_receiver_frozen BOOLEAN;
BEGIN
    -- Check account status
    SELECT is_frozen INTO v_sender_frozen FROM ledger_accounts WHERE account_id = v_acc_sender;
    SELECT is_frozen INTO v_receiver_frozen FROM ledger_accounts WHERE account_id = v_acc_receiver;

    IF v_sender_frozen OR v_receiver_frozen THEN
        RAISE EXCEPTION 'Transfer failed: One or both accounts are frozen.';
    END IF;

    -- 1. Deduct from Sender (Enforces chk_solvency constraint immediately)
    UPDATE ledger_accounts
    SET balance = balance - v_transfer_amount,
        updated_at = clock_timestamp()
    WHERE account_id = v_acc_sender;

    -- 2. Credit to Receiver
    UPDATE ledger_accounts
    SET balance = balance + v_transfer_amount,
        updated_at = clock_timestamp()
    WHERE account_id = v_acc_receiver;

    -- 3. Create Audit Journal Entry
    INSERT INTO journal_entries (reference_id, description)
    VALUES (v_ref, 'Transfer from Alice to Bob')
    RETURNING entry_id INTO v_entry_id;

    -- 4. Record Double-Entry Posting Lines
    INSERT INTO journal_lines (entry_id, account_id, amount, direction)
    VALUES 
        (v_entry_id, v_acc_sender, v_transfer_amount, 'CREDIT'),
        (v_entry_id, v_acc_receiver, v_transfer_amount, 'DEBIT');

    RAISE NOTICE 'Transfer successful! Journal Entry ID: %', v_entry_id;
END $$;
```

---

### Why ACID & WAL Protect this Transaction:

1. **Atomicity**: If `balance - v_transfer_amount` violates `chk_solvency` (e.g. attempting to transfer $9999), the PostgreSQL engine immediately triggers a constraint violation exception. The entire block rolls back. No partial rows in `journal_entries` or `journal_lines` are committed.
2. **Durability via WAL**: When `COMMIT` completes, all modified data pages and row states are written to the Write-Ahead Log (WAL) on non-volatile disk. Even if the server suffers an abrupt power cut 1 microsecond after commit confirmation, recovery replay on restart scans the WAL and recovers the exact committed state without loss.

</details>

---

## Exercise 3: High-Throughput Inventory Reservation via Conditional UPSERT

### Context & Problem Statement

During a flash-sale event, thousands of distributed microservices receive checkout requests for popular products. You need to write an idempotent reservation query that handles concurrent order spikes.

A reservation table tracks quantities reserved by each user session:
- If a user reserves an item they don't yet have in their cart, insert a new reservation.
- If the user already has a pending reservation for that `(cart_id, sku)`, increase their reserved quantity by the new requested amount.
- **The Catch**: You cannot simply allow unlimited increments! An increment should only occur if the total requested quantity does not exceed the maximum allowed purchase limit (e.g., maximum 5 items per customer) AND the warehouse has sufficient stock.
- The query must return the old reserved quantity, the newly updated reserved quantity, and the status.

---

### Your Tasks

1. **Schema DDL**:
   - Create table `inventory`:
     - `sku` (`VARCHAR(50)` PRIMARY KEY)
     - `title` (`TEXT NOT NULL`)
     - `available_stock` (`INT NOT NULL CHECK (available_stock >= 0)`)
   - Create table `cart_reservations`:
     - `reservation_id` (`BIGINT GENERATED ALWAYS AS IDENTITY` PRIMARY KEY)
     - `cart_id` (`UUID NOT NULL`)
     - `sku` (`VARCHAR(50) NOT NULL REFERENCES inventory(sku) ON DELETE RESTRICT`)
     - `reserved_qty` (`INT NOT NULL CHECK (reserved_qty > 0 AND reserved_qty <= 5)`)
     - `expires_at` (`TIMESTAMPTZ NOT NULL`)
     - `updated_at` (`TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp()`)
     - Ensure `(cart_id, sku)` is enforced as unique!

2. **Advanced DML Challenge**:  
   Write a single atomic `INSERT ... ON CONFLICT (...) DO UPDATE` statement that:
   - Attempts to reserve quantity `2` for `cart_id = 'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa'` on SKU `'GPU-RTX-5090'`.
   - On conflict on `(cart_id, sku)`:
     - Only updates if the new total `cart_reservations.reserved_qty + EXCLUDED.reserved_qty <= 5`.
     - Extends `expires_at` by 15 minutes (`clock_timestamp() + INTERVAL '15 minutes'`).
     - Updates `updated_at` to `clock_timestamp()`.
   - Uses `RETURNING` to output `reservation_id`, the old quantity, the new quantity, and the updated expiration timestamp.

<details>
<summary>Reveal Solution & Explanation</summary>

### 1. Schema Setup DDL

```sql
DROP TABLE IF EXISTS cart_reservations CASCADE;
DROP TABLE IF EXISTS inventory CASCADE;

CREATE TABLE inventory (
    sku VARCHAR(50) PRIMARY KEY,
    title TEXT NOT NULL,
    available_stock INT NOT NULL CHECK (available_stock >= 0)
);

CREATE TABLE cart_reservations (
    reservation_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cart_id UUID NOT NULL,
    sku VARCHAR(50) NOT NULL REFERENCES inventory(sku) ON DELETE RESTRICT,
    reserved_qty INT NOT NULL CHECK (reserved_qty > 0 AND reserved_qty <= 5),
    expires_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT clock_timestamp(),
    
    -- Composite unique constraint required for target in ON CONFLICT
    CONSTRAINT uq_cart_sku UNIQUE (cart_id, sku)
);

-- Seed stock
INSERT INTO inventory (sku, title, available_stock)
VALUES ('GPU-RTX-5090', 'NVIDIA GeForce RTX 5090', 20);
```

---

### 2. Atomic Conditional UPSERT Query

```sql
-- Step 1: Initial Insert (Reserves 2 units)
INSERT INTO cart_reservations (cart_id, sku, reserved_qty, expires_at)
VALUES (
    'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', 
    'GPU-RTX-5090', 
    2, 
    clock_timestamp() + INTERVAL '15 minutes'
)
ON CONFLICT (cart_id, sku) 
DO UPDATE SET
    reserved_qty = cart_reservations.reserved_qty + EXCLUDED.reserved_qty,
    expires_at = clock_timestamp() + INTERVAL '15 minutes',
    updated_at = clock_timestamp()
WHERE (cart_reservations.reserved_qty + EXCLUDED.reserved_qty) <= 5
RETURNING 
    reservation_id,
    cart_id,
    sku,
    (cart_reservations.reserved_qty - EXCLUDED.reserved_qty) AS old_reserved_qty,
    cart_reservations.reserved_qty AS new_reserved_qty,
    expires_at;

-- Step 2: Running the exact same statement again (Attempts to add 2 more -> new total 4)
INSERT INTO cart_reservations (cart_id, sku, reserved_qty, expires_at)
VALUES (
    'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', 
    'GPU-RTX-5090', 
    2, 
    clock_timestamp() + INTERVAL '15 minutes'
)
ON CONFLICT (cart_id, sku) 
DO UPDATE SET
    reserved_qty = cart_reservations.reserved_qty + EXCLUDED.reserved_qty,
    expires_at = clock_timestamp() + INTERVAL '15 minutes',
    updated_at = clock_timestamp()
WHERE (cart_reservations.reserved_qty + EXCLUDED.reserved_qty) <= 5
RETURNING 
    reservation_id,
    cart_id,
    sku,
    (cart_reservations.reserved_qty - EXCLUDED.reserved_qty) AS old_reserved_qty,
    cart_reservations.reserved_qty AS new_reserved_qty,
    expires_at;

-- Step 3: Running with 2 more (Total would be 6 > 5 limit)
-- Notice that the WHERE filter stops the update, and 0 rows are returned!
INSERT INTO cart_reservations (cart_id, sku, reserved_qty, expires_at)
VALUES (
    'aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa', 
    'GPU-RTX-5090', 
    2, 
    clock_timestamp() + INTERVAL '15 minutes'
)
ON CONFLICT (cart_id, sku) 
DO UPDATE SET
    reserved_qty = cart_reservations.reserved_qty + EXCLUDED.reserved_qty,
    expires_at = clock_timestamp() + INTERVAL '15 minutes',
    updated_at = clock_timestamp()
WHERE (cart_reservations.reserved_qty + EXCLUDED.reserved_qty) <= 5
RETURNING 
    reservation_id,
    cart_id,
    sku,
    cart_reservations.reserved_qty AS new_reserved_qty;
```

---

### Why This Works Under Concurrency:
- The pseudo-table `EXCLUDED` represents the row proposed for insertion.
- The `WHERE` clause in `DO UPDATE SET` acts as a concurrency barrier. If the condition evaluates to `FALSE`, the row is **not updated**, preventing race conditions where multiple parallel threads could bypass application validation checks.

</details>

---

## Exercise 4: Semi-Structured JSONB & Array Telemetry System

### Context & Problem Statement

You are architecting the backend for an edge IoT computing platform. Devices run custom firmware and periodically submit configuration documents containing dynamic hardware properties and metric tags.

---

### Your Tasks

1. **Schema DDL with Advanced JSONB Validation**:  
   Create table `edge_devices`:
   - `device_id` (`UUID` PRIMARY KEY)
   - `serial_number` (`VARCHAR(64)` UNIQUE NOT NULL)
   - `firmware_version` (`VARCHAR(32)` NOT NULL)
   - `tags` (`TEXT[]` DEFAULT `ARRAY[]::TEXT[]`)
   - `config` (`JSONB` NOT NULL DEFAULT `'{}'::jsonb`)
   - `is_online` (`BOOLEAN` DEFAULT FALSE)
   - `last_heartbeat` (`TIMESTAMPTZ`)
   
   **Mandatory Integrity Check Constraints**:
   - Write a `CHECK` constraint validating that `config` is a JSON Object (not a scalar or array).
   - Write a `CHECK` constraint ensuring `config` contains the mandatory top-level keys: `"cpu_cores"` and `"memory_mb"`.
   - Write a `CHECK` constraint enforcing that `(config->>'cpu_cores')::int` is between 1 and 128, and `(config->>'memory_mb')::int >= 512`.
   - Write a `CHECK` constraint enforcing that no tag in `tags` is an empty string.

2. **DML & JSONB Manipulation Queries**:
   - **Query A**: Insert a device with serial `'SN-EDGE-9001'`, firmware `'v2.4.1'`, tags `['factory-floor', 'zone-a', 'arm64']`, and config:
     ```json
     {
       "cpu_cores": 8,
       "memory_mb": 4096,
       "network": {
         "interface": "eth0",
         "ip_mode": "dhcp",
         "dns": ["1.1.1.1", "8.8.8.8"]
       }
     }
     ```
   - **Query B**: Perform an atomic `UPDATE` that adds a secondary DNS server `"9.9.9.9"` to `config->'network'->'dns'` without replacing existing entries, using `jsonb_set`.
   - **Query C**: Perform an atomic `UPDATE` that appends the tag `'critical'` to the `tags` array only if `'critical'` is not already present in the array.
   - **Query D**: Write a `SELECT` query using the JSON containment operator (`@>`) and array operator (`&&`) that finds all online devices that have `"ip_mode": "dhcp"` and contain either `'factory-floor'` or `'warehouse'` in their `tags`.

<details>
<summary>Reveal Solution & Explanation</summary>

### 1. DDL Schema Definition

```sql
DROP TABLE IF EXISTS edge_devices CASCADE;

CREATE TABLE edge_devices (
    device_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    serial_number VARCHAR(64) UNIQUE NOT NULL,
    firmware_version VARCHAR(32) NOT NULL,
    tags TEXT[] NOT NULL DEFAULT ARRAY[]::TEXT[],
    config JSONB NOT NULL DEFAULT '{}'::jsonb,
    is_online BOOLEAN NOT NULL DEFAULT FALSE,
    last_heartbeat TIMESTAMPTZ,
    
    -- Constraint 1: Must be a JSON object
    CONSTRAINT chk_config_is_object CHECK (jsonb_typeof(config) = 'object'),
    
    -- Constraint 2: Must contain mandatory keys
    CONSTRAINT chk_config_has_mandatory_keys CHECK (
        config ? 'cpu_cores' AND config ? 'memory_mb'
    ),
    
    -- Constraint 3: Numerical boundaries within JSONB
    CONSTRAINT chk_config_hardware_bounds CHECK (
        (config->>'cpu_cores')::int BETWEEN 1 AND 128 AND
        (config->>'memory_mb')::int >= 512
    ),
    
    -- Constraint 4: No empty string in tags array
    CONSTRAINT chk_tags_not_empty CHECK (
        NOT ('' = ANY(tags))
    )
);
```

---

### 2. DML Queries

```sql
-- Query A: Insert Edge Device
INSERT INTO edge_devices (serial_number, firmware_version, tags, config, is_online, last_heartbeat)
VALUES (
    'SN-EDGE-9001',
    'v2.4.1',
    ARRAY['factory-floor', 'zone-a', 'arm64'],
    '{
        "cpu_cores": 8,
        "memory_mb": 4096,
        "network": {
            "interface": "eth0",
            "ip_mode": "dhcp",
            "dns": ["1.1.1.1", "8.8.8.8"]
        }
    }'::jsonb,
    TRUE,
    clock_timestamp()
);

-- Query B: Append DNS to nested JSON array using jsonb_set
UPDATE edge_devices
SET config = jsonb_set(
    config, 
    '{network,dns}', 
    (config->'network'->'dns') || '"9.9.9.9"'::jsonb
)
WHERE serial_number = 'SN-EDGE-9001'
RETURNING serial_number, config->'network' AS updated_network;

-- Query C: Append 'critical' tag idempotently
UPDATE edge_devices
SET tags = array_append(tags, 'critical')
WHERE serial_number = 'SN-EDGE-9001'
  AND NOT ('critical' = ANY(tags))
RETURNING serial_number, tags;

-- Query D: Query with JSONB containment (@>) and Array overlap (&&)
SELECT 
    device_id,
    serial_number,
    firmware_version,
    tags,
    config->'network'->>'interface' AS iface,
    last_heartbeat
FROM edge_devices
WHERE is_online = TRUE
  AND config @> '{"network": {"ip_mode": "dhcp"}}'::jsonb
  AND tags && ARRAY['factory-floor', 'warehouse'];
```

</details>

---

## Exercise 5: Production Incident Post-Mortem & Schema Code Review

### Context & Problem Statement

You are a Principal Database Architect reviewing pull requests and investigating recent production anomalies. Identify the root cause for each of the following 4 engineering disasters, explain why it broke in production, and write the corrected SQL.

---

### Incident A: The Financial Drift Discrepancy

A cryptocurrency settlement engine designed its transaction account ledger as follows:

```sql
CREATE TABLE wallet_balances (
    wallet_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    balance REAL NOT NULL DEFAULT 0.0,
    total_deposited FLOAT NOT NULL DEFAULT 0.0
);
```

#### The Bug:
After processing 50,000 micro-deposits of `$0.10`, the expected balance was `$5,000.00`. Instead, the system reported `$5000.042384`, causing daily automated reconciliation scripts to halt deposits.

**Your Task**: Explain why `REAL` and `FLOAT` failed, and rewrite the table with appropriate data types.

---

### Incident B: The Timezone Blindness Incident

A global booking platform logged flight departure timestamps using:

```sql
CREATE TABLE flight_schedules (
    flight_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    flight_number VARCHAR(10) NOT NULL,
    departure_time TIMESTAMP NOT NULL,
    arrival_time TIMESTAMP NOT NULL
);
```

#### The Bug:
An application server located in San Francisco (`UTC-7`) inserted a departure time of `'2026-10-15 10:00:00'`. A notification service running on an AWS cluster in Frankfurt (`UTC+2`) queried the flight table and alerted passengers 9 hours earlier than scheduled.

**Your Task**: Explain why `TIMESTAMP` (without timezone) caused this issue and write the corrected DDL using `TIMESTAMPTZ` with timezone offset handling.

---

### Incident C: The Multi-Tenant Unique Constraint Hole

A SaaS application allowed global super-admins (`tenant_id IS NULL`) as well as tenant-specific users (`tenant_id = <UUID>`):

```sql
CREATE TABLE users (
    user_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    tenant_id UUID REFERENCES tenants(tenant_id),
    email VARCHAR(255) NOT NULL,
    CONSTRAINT uq_tenant_email UNIQUE (tenant_id, email)
);
```

#### The Bug:
The system intended to ensure that no duplicate email could exist for global super-admins (`tenant_id IS NULL`). However, a hacker was able to register 20 different global accounts using the exact same email `admin@global.com`.

**Your Task**: Explain why standard SQL `UNIQUE` constraints permit duplicate rows when columns contain `NULL`, and provide the PostgreSQL 15+ solution as well as the partial index solution for older versions.

---

### Incident D: The Accidental Cascade Purge

An architect designed an e-commerce platform with the following relationships:

```sql
CREATE TABLE merchants (
    merchant_id UUID PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE orders (
    order_id BIGINT PRIMARY KEY,
    merchant_id UUID REFERENCES merchants(merchant_id) ON DELETE CASCADE,
    total_amount NUMERIC(10, 2) NOT NULL
);

CREATE TABLE audit_tax_records (
    record_id BIGINT PRIMARY KEY,
    order_id BIGINT REFERENCES orders(order_id) ON DELETE CASCADE,
    tax_authority TEXT NOT NULL,
    filed_at TIMESTAMPTZ NOT NULL
);
```

#### The Bug:
A junior developer ran a cleanup query: `DELETE FROM merchants WHERE name = 'Test Merchant';`.
Unbeknownst to them, a bug in onboarding had associated thousands of real customer orders with that merchant ID. The `ON DELETE CASCADE` wiped out thousands of orders and legally mandated `audit_tax_records`, violating financial compliance laws.

**Your Task**: Explain proper referential actions (`RESTRICT`, `NO ACTION`, soft-deletes) to protect critical audit records from accidental cascading destruction.

<details>
<summary>Reveal Solution & Code Review Explanations</summary>

### Solution A: Floating Point vs Exact Numeric

- **Root Cause**: `REAL` (single-precision 32-bit) and `FLOAT` (double-precision 64-bit) implement **IEEE 754 floating-point arithmetic**. In IEEE 754, decimal numbers like `0.1` cannot be represented with exact precision in binary fractions, leading to cumulative rounding errors.
- **Remedy**: Always use `NUMERIC(precision, scale)` or `DECIMAL` for financial and currency balances:

```sql
CREATE TABLE wallet_balances (
    wallet_id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    balance NUMERIC(18, 8) NOT NULL DEFAULT 0.00000000 CHECK (balance >= 0),
    total_deposited NUMERIC(18, 8) NOT NULL DEFAULT 0.00000000 CHECK (total_deposited >= 0)
);
```

---

### Solution B: `TIMESTAMP` vs `TIMESTAMPTZ`

- **Root Cause**: `TIMESTAMP` (or `TIMESTAMP WITHOUT TIME ZONE`) completely discards timezone context. It simply stores the local year, month, day, hour, minute, second. When two servers in different timezones query the same row, they interpret the digits as their own local time, resulting in severe discrepancies.
- **Remedy**: Use `TIMESTAMPTZ` (`TIMESTAMP WITH TIME ZONE`). PostgreSQL converts `TIMESTAMPTZ` values to UTC upon storage and automatically formats them relative to the client session's timezone on retrieval:

```sql
CREATE TABLE flight_schedules (
    flight_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    flight_number VARCHAR(10) NOT NULL,
    departure_time TIMESTAMPTZ NOT NULL,
    arrival_time TIMESTAMPTZ NOT NULL,
    CONSTRAINT chk_flight_duration CHECK (arrival_time > departure_time)
);
```

---

### Solution C: Nullability Semantics in Unique Constraints

- **Root Cause**: According to the ANSI SQL standard, `NULL` represents an "unknown" value. Because `NULL = NULL` evaluates to `NULL` (unknown), standard SQL `UNIQUE` constraints treat each `NULL` value as distinct from every other `NULL`. Therefore, multiple rows with `tenant_id = NULL` and `email = 'admin@global.com'` do not conflict.
- **Remedies**:

1. **PostgreSQL 15+ Syntax** (`NULLS NOT DISTINCT`):
   ```sql
   CREATE TABLE users (
       user_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
       tenant_id UUID REFERENCES tenants(tenant_id),
       email VARCHAR(255) NOT NULL,
       CONSTRAINT uq_tenant_email UNIQUE NULLS NOT DISTINCT (tenant_id, email)
   );
   ```

2. **PostgreSQL 14 and earlier** (Partial Unique Indexes):
   ```sql
   -- Enforces uniqueness for tenant users
   CREATE UNIQUE INDEX uq_tenant_users 
   ON users (tenant_id, email) 
   WHERE tenant_id IS NOT NULL;

   -- Enforces uniqueness for global users
   CREATE UNIQUE INDEX uq_global_users 
   ON users (email) 
   WHERE tenant_id IS NULL;
   ```

---

### Solution D: Defensive Foreign Key Design

- **Root Cause**: Applying `ON DELETE CASCADE` blindly to root-level entities cascades deletions downstream, obliterating compliance-critical audit tables.
- **Remedy**:
  1. Use `ON DELETE RESTRICT` (or `NO ACTION`) on financial and audit-bound entities. PostgreSQL will throw an error if an administrator attempts to delete a parent entity that still has child orders or tax records attached.
  2. Implement an explicit **Soft-Delete** pattern (`is_deleted BOOLEAN`, `deleted_at TIMESTAMPTZ`) for merchants rather than hard `DELETE` statements.

```sql
CREATE TABLE merchants (
    merchant_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    name TEXT NOT NULL,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    deleted_at TIMESTAMPTZ
);

CREATE TABLE orders (
    order_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    merchant_id UUID NOT NULL REFERENCES merchants(merchant_id) ON DELETE RESTRICT,
    total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount >= 0)
);

CREATE TABLE audit_tax_records (
    record_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(order_id) ON DELETE RESTRICT,
    tax_authority TEXT NOT NULL,
    filed_at TIMESTAMPTZ NOT NULL
);
```

</details>
