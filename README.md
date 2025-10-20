# SQL vs NoSQL (MongoDB) Schema Comparison

This document compares the original **MS SQL** schema and the new **MongoDB** schema you implemented. It summarizes entity mappings, keys and constraints, validation equivalents, indexing, and deliberate differences caused by the relational→document model shift.

---

## 1. High-Level Mapping

| Domain Concept | MS SQL (Tables) | MongoDB (Collections) | Notes |
|---|---|---|---|
| Enums | `etl._enum_etf_status`, `etl._enum_dist_type`, `etl._enum_dist_freq`, `etl._enum_sec_type` | `_enum_etf_status`, `_enum_dist_type`, `_enum_dist_freq`, `_enum_sec_type` | Same semantic role. In practice JSON Schema `enum` is also used directly in the target collections where relevant. |
| Bank | `etl.bank` | `bank` | Unique `swift`. |
| Exchange | `etl.exchange` | `exchange` | Unique `mic`. |
| Currency | `etl.currency` | `currency` | Unique `code` (length 3). |
| ETF (Fund) | `etl.etf` | `etf` | Required fields + `status` enum; `bankId` ref (no FK). |
| Distribution | `etl.distribution` | `distribution` | Date-order constraint carried via `$expr` validator. |
| Security (supertype) | `etl.security` | `security` | Single collection with discriminator `secType` replacing 1:1 subtype tables. |
| Equity Security (subtype) | `etl.equity_security` (1:1) | *part of* `security` (doc with `secType: "equity"`) | Flattened ISA via discriminator + optional subtype fields. |
| Bond Security (subtype) | `etl.bond_security` (1:1) | *part of* `security` (doc with `secType: "bond"`) | Same. |
| Holding | `etl.holding` | `holding` | Self-reference via `parentHoldingId`. Optional `path` for tree queries. |
| Index | `etl.idx` | `idx` | Simple entity. |
| Index Constituent | `etl.index_constituent` | `index_constituent` | Composite PK mapped to compound unique index. |
| Top List | `etl.toplist` | `toplist` | Simple entity. |
| Top List Item | `etl.toplist_item` | `toplist_item` | Weak entity → standard collection + index on `toplistId`. |
| Listing (ternary) | `etl.listing` | `listing` | Composite PK + partial unique index for “exactly one primary per fund”. |

---

## 2. Keys, Constraints & Their MongoDB Equivalents

### 2.1 Primary Keys & Identity
- **SQL**: `BIGINT IDENTITY` PKs. Some composite PKs (e.g., `index_constituent`).  
- **MongoDB**: `_id: ObjectId` by default. Composite PKs are represented as **compound unique indexes** (e.g., `index_constituent(indexId, securityId, asOfDate)` unique).

### 2.2 Foreign Keys & Referential Integrity
- **SQL**: FKs with `ON DELETE CASCADE` in several places.  
- **MongoDB**: **No built-in FK**. References are by ObjectId; **application** is responsible for ensuring existence and cascades. (Optional: transactions can enforce multi-collection atomicity when required.)

### 2.3 CHECK Constraints & Validation
- **SQL**: CHECK on enumerations, non-negativity, date order.  
- **MongoDB**: 
  - **JSON Schema validators** enforce *required*, *type*, *min/max*, *length*.  
  - **`enum`** used directly in `etf.status`, `distribution.distType/distFreq`, `security.secType`.  
  - **Date order** (`recordDate ≤ exDate ≤ payDate`) replicated via `validator.$expr` in `distribution`.  

### 2.4 Unique Constraints
- **SQL**: unique on fields like `bank.swift`, `exchange.mic`, `currency.code`, `security.isin/cusip/sedol`, `listing(exchange,ticker)`.  
- **MongoDB**: equivalent **unique indexes**, with `sparse: true` for optional identifiers in `security`.

### 2.5 Partial/Filtered Unique Index
- **SQL**: filtered unique index `IX_listing_one_primary_per_fund` where `is_primary=1`.  
- **MongoDB**: **partial unique index** `{ fundId: 1 }` with `{ isPrimary: true }` filter — same behavior (“exactly one primary listing per fund”).

---

## 3. Entity-by-Entity Notes

### 3.1 Enums
- **SQL**: small lookup tables with CHECK and PK.  
- **MongoDB**: mirror collections + unique index; but enforcement occurs primarily via **JSON Schema `enum`** in the main collections, reducing runtime joins.

### 3.2 Security Supertype/Subtypes (ISA)
- **SQL**: `security` + `equity_security`(1:1), `bond_security`(1:1).  
- **MongoDB**: single `security` collection with `secType` discriminator and optional subtype fields (`equityName`, `coupon`, `maturityDate`, `rating`).  
- **Rationale**: one-document reads, fewer joins, natural fit for document stores.

### 3.3 Holding (Self-hierarchy)
- **SQL**: FK to self + trigger to ensure *same fund* and *no cycles*.  
- **MongoDB**: schema created **without triggers** (per scope). Self-link via `parentHoldingId`. Optional `path` array to accelerate descendant queries. Business rules should be enforced in application logic or server-side helpers if needed.

### 3.4 Listing (Ternary)
- **SQL**: `(fund_id, exchange_id, currency_id)` as PK; unique `(exchange_id, ticker)`; one primary per fund.  
- **MongoDB**: compound unique `(fundId, exchangeId, currencyId)`; unique `(exchangeId, ticker)`; partial unique on `fundId` with `{ isPrimary: true }` — **1:1 parity**.

### 3.5 Index Constituent (Weak Entity with Composite PK)
- **SQL**: PK `(index_id, security_id, as_of_date)` and checks.  
- **MongoDB**: compound unique index on `(indexId, securityId, asOfDate)`; range index `(indexId, asOfDate)` — **same access pattern**.

---

## 4. Indexing Summary (MongoDB)

- `_enum_*`: `{ val: 1 }` unique.  
- `bank`: `{ swift: 1 }` unique.  
- `exchange`: `{ mic: 1 }` unique.  
- `currency`: `{ code: 1 }` unique.  
- `etf`: `{ bankId: 1 }`.  
- `distribution`: `{ etfId: 1 }`.  
- `security`: `{ isin: 1 }` unique sparse; `{ cusip: 1 }` unique sparse; `{ sedol: 1 }` unique sparse.  
- `holding`: `{ fundId: 1 }`, `{ securityId: 1 }`, `{ parentHoldingId: 1 }`, optional `{ fundId: 1, path: 1 }`.  
- `idx`: *(no special index needed beyond `_id`)*.  
- `index_constituent`: `{ indexId: 1, securityId: 1, asOfDate: 1 }` **unique** + `{ indexId: 1, asOfDate: 1 }`.  
- `toplist`: `{ bankId: 1 }`.  
- `toplist_item`: `{ toplistId: 1 }`.  
- `listing`: unique `{ fundId: 1, exchangeId: 1, currencyId: 1 }`; unique `{ exchangeId: 1, ticker: 1 }`; partial unique `{ fundId: 1 }` with `{ isPrimary: true }`.

---

## 5. Data Types & Precision

- **Numeric**: SQL `DECIMAL(p,s)` → MongoDB `Decimal128` (`NumberDecimal(...)`) where monetary/ratio precision matters (e.g., `nav`, `mer`, `amountPerShare`, `targetWeight`, market values, units). Validators currently allow `["double","decimal"]` for flexibility; you may restrict to `"decimal"` for strictness.
- **Dates**: SQL `DATE` → MongoDB `Date` (`ISODate(...)`). Ordering constraints preserved in `distribution`.
- **Strings & Lengths**: SQL `NVARCHAR` lengths mirrored via JSON Schema `maxLength` (advisory at write time).

---

## 6. What’s Intentionally Different

1. **No Foreign Keys / Cascades**: Replace with application-enforced checks and, if necessary, multi-document **transactions** for atomicity.  
2. **Triggers Omitted**: The `holding` guard (same fund & acyclicity) is not part of this scope; enforce in service layer.  
3. **Flattened ISA**: Supertype/subtype merged into a single `security` collection with a discriminator.  
4. **Optional `holding.path`**: Not present in SQL, added to accelerate tree queries in document model (optional to use).

---

## 7. Seed Data Parity

The MongoDB demo inserts correspond to the SQL demo section:
- `bank`, `exchange`, `currency`, `etf`, `listing (primary)`, `distribution`  
- `security` (equity + bond variants), `holding` (parent + child),  
- `idx`, `index_constituent`, `toplist`, `toplist_item`

These provide a like-for-like starting dataset to exercise the schema and constraints.

---

## 8. Migration & Application Notes

- **Access Patterns First**: In MongoDB, design for reads you need most often; denormalize where it reduces round-trips.  
- **Transactions**: Use only where aggregate boundaries demand atomic writes (e.g., cross-collection updates).  
- **Cascades**: Implement in service code (e.g., delete `etf` → delete related `distribution`, `listing`, `holding`, etc.).  
- **Validation Failures**: JSON Schema validators and unique indexes will surface errors at insert/update time; handle them in API responses.  
- **Enum Sources**: You can rely on JSON Schema enums in target collections and treat `_enum_*` as documentation/UX sources (autocomplete) rather than join targets.

---

## 9. Conclusion

- The MongoDB schema **faithfully mirrors** the relational structure where it matters (entities, uniqueness, key constraints, and critical validations like date ordering and “one primary listing per fund”).  
- Differences reflect **best practices for document databases**: flattened ISA, app-level referential integrity, optional denormalizations for performance, and validators replacing CHECK constraints.
