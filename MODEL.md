# Data Model Documentation

## Overview

The data model is designed to handle multi-tenant carbon accounting with source-of-truth tracking, audit compliance, and analyst review workflows. Every emission record is immutable once approved, with complete edit history maintained.

---

## Core Entities

### 1. **Tenant** (Organization)
Multi-tenancy is implemented at the database row level using tenant_id foreign keys.

```python
class Tenant:
  id: UUID (primary key)
  name: string
  slug: string (unique, URL-safe identifier)
  created_at: datetime
```

**Why this design:**
- Slug enables easy user-friendly URLs without exposing internal IDs
- UUID for globally unique identifiers across distributed systems
- Soft multi-tenancy: all queries filtered by tenant_id for data isolation

---

### 2. **Facility** (Physical Location)
Facilities link emissions to real-world locations and bridge source system codes.

```python
class Facility:
  id: UUID
  tenant_id: UUID (foreign key)
  name: string
  plant_code: string          # SAP plant identifier (1000, 2000, etc)
  location_code: string       # Utility meter location code
  country: string
  created_at: datetime
  
  unique_constraint: (tenant_id, plant_code)
  index: (tenant_id, plant_code)
```

**Design rationale:**
- `plant_code` links SAP master data to facilities without hardcoding facility IDs in SAP exports
- Utilities use different identifiers; `location_code` bridges that gap
- Unique constraint prevents duplicate facility definitions per tenant
- Index on tenant+plant_code enables fast lookups during ingestion

---

### 3. **Emission** (Core Record)
Every calculation, metric, and decision point flows through this model.

```python
class Emission:
  # Identification
  id: UUID (primary key)
  tenant_id: UUID (required, not null)
  facility_id: UUID (optional - not all emissions tie to facilities)
  
  # Source tracking (implements source-of-truth requirement)
  data_source: enum [sap_fuel, sap_procurement, utility_electricity, 
                      travel_flight, travel_hotel, travel_ground]
  source_id: string           # Unique ID from originating system
  ingestion_timestamp: datetime (auto)
  
  # Activity temporal dimension
  activity_date: date         # When the activity occurred
  period_start: date (optional)  # For billing periods (utility meter)
  period_end: date (optional)
  
  # Quantitative data
  description: string         # Human-readable activity description
  quantity: decimal           # Amount consumed/traveled
  unit: string                # L, kg, kWh, km, nights
  
  # Normalization (handles unit inconsistencies)
  normalized_quantity: decimal
  normalized_unit: string (default='kg')
  
  # Emission factor (reference data)
  emission_factor: decimal    # kg CO2e per unit
  emission_factor_source: string  # IPCC, EPA, custom, grid operator
  
  # Calculated result
  co2e_kg: decimal (auto-calculated)
  
  # Classification (Scope 1/2/3)
  scope: enum ['1', '2', '3']
  category: string            # "Diesel Fuel", "Electricity - Grid Mix"
  
  # Review workflow
  status: enum [pending, flagged, approved, rejected] (default=pending)
  reviewed_by: FK User (nullable)
  reviewed_at: datetime (nullable)
  review_notes: text
  
  # Data quality
  is_estimated: boolean (default=false)
  quality_flags: JSON         # [{type, severity, message}]
  
  # Audit trail
  created_at: datetime (auto)
  updated_at: datetime (auto)
  version: integer (default=1)
  
  # Soft delete (immutability after approval)
  is_deleted: boolean (default=false)
  deleted_at: datetime (nullable)
  deleted_reason: string
  
  # Indexes for common queries
  index: (tenant_id, status)
  index: (tenant_id, activity_date)
  index: (tenant_id, data_source)
  index: (tenant_id, scope)
  unique: (tenant_id, source_id, data_source)
```

**Design decisions:**

1. **Source-of-truth tracking**: `data_source` + `source_id` + `ingestion_timestamp` forms an immutable record of origin. Original source ID is preserved even if the facility mapping changes.

2. **Unit normalization**: Separate `quantity`/`unit` from `normalized_quantity`/`normalized_unit` allows:
   - Audit trail of what was ingested (not transformed)
   - Detection of conversion errors
   - Re-calculation if conversion factors change

3. **Emission factor storage**: Inline factor + source string enables:
   - Audit compliance: "This record used EPA 2024 factors, not 2023"
   - Sensitivity analysis: re-calculate with different factors
   - Quality flags when factor is missing

4. **Status workflow**: `pending` → `approved`/`rejected` is one-way (soft delete prevents actual deletion after approval). `flagged` is a review state, not a final state.

5. **Quality flags as JSON**: Structured list allows:
   - Multiple simultaneous issues (missing factor + stale data)
   - Severity levels (high/medium/low) for triage
   - Specific messages for analyst guidance

6. **Version field**: Incremented on each update (unused currently but prepared for multi-version workflows).

---

### 4. **EmissionAuditLog** (Immutable Change Record)
Every change to an Emission is logged here. This is the audit trail.

```python
class EmissionAuditLog:
  id: UUID
  emission_id: UUID (FK, cascade delete)
  action: string              # created, updated, approved, rejected, deleted
  changed_fields: JSON        # {field_name: {old: value, new: value}}
  changed_by: FK User
  changed_at: datetime (auto)
  notes: string (optional)
  
  index: emission_id
  ordering: -changed_at
```

**Why JSON for changed_fields:**
- Flexible: can capture any field changes without schema migrations
- Human-readable: diff-like format for auditors
- Queryable: `changed_fields->'emission_factor'->'old'` in advanced queries

---

### 5. **IngestionBatch** (Import Tracking)
Tracks file uploads and ingestion success/failure rates.

```python
class IngestionBatch:
  id: UUID
  tenant_id: UUID
  data_source: enum
  filename: string
  status: enum [processing, completed, failed]
  
  rows_received: integer
  rows_successful: integer
  rows_failed: integer
  
  ingestion_started: datetime (auto)
  ingestion_completed: datetime
  error_log: JSON             # [{row: N, error: string}, ...]
```

**Purpose:**
- PM asks: "Did the utility data from March 15 fully load?" → Check batch status
- Analyst asks: "Which rows failed in that upload?" → View error_log
- Enables retry logic and human reconciliation

---

### 6. **Reference Data**

#### EmissionFactor
```python
class EmissionFactor:
  id: integer
  category: string            # "Diesel Fuel", "Electricity - Grid Mix"
  scope: enum ['1', '2', '3']
  unit: string                # per L, per kWh, per km, per night
  value: decimal              # kg CO2e
  source: string              # IPCC AR6, EPA 2024, Carbon Trust, custom
  year: integer
  country: string (optional)  # For region-specific factors (US, EU, IN)
  notes: string
  active: boolean (default=true)
```

**Design:**
- Scoped by `year` and `country` to support regulatory changes
- `active=false` for deprecated factors (maintains history)
- Most common query: `EmissionFactor.objects.filter(category=X, year__lte=current_year, active=True).latest('year')`

#### UnitConversion
```python
class UnitConversion:
  source_unit: string
  target_unit: string
  conversion_factor: decimal
  notes: string
  
  unique: (source_unit, target_unit)
```

**Examples:**
- L → mL: 1000
- kg → t: 0.001
- kWh → GJ: 0.0036

---

## Multi-Tenancy Strategy

### Row-Level Security
Every table has a `tenant_id` foreign key (except reference tables). All queries include `.filter(tenant_id=requesting_user_tenant_id)`.

```python
# In viewsets:
def get_queryset(self):
    return Emission.objects.filter(tenant__slug=self.kwargs['tenant_slug'], is_deleted=False)
```

### Isolation Guarantees
1. A user from "Acme Corp" cannot query "Chevron Corp" emissions
2. Unique constraints prevent cross-tenant collisions (e.g., `(tenant_id, source_id, data_source)`)
3. No `SELECT *` queries across tenants; all are tenant-scoped

---

## Scope Classification

| Scope | Definition | Sources | Examples |
|-------|-----------|---------|----------|
| **1** | Direct emissions from owned/controlled sources | SAP fuel, facilities | Diesel burn, natural gas, fleet vehicles |
| **2** | Indirect emissions from purchased electricity | Utility bills | Grid electricity, renewable RECs |
| **3** | All other indirect emissions | Travel, procurement, business travel | Flights, hotels, taxi, supplier emissions |

**Model enforces:** `scope` is set at ingestion based on `data_source`:
- `sap_fuel` → Scope 1
- `utility_electricity` → Scope 2
- `travel_*` → Scope 3

---

## Calculation & Validation

### CO2e Calculation (automatic)
```
co2e_kg = normalized_quantity * emission_factor

Example:
- Activity: 1000 L of Diesel
- Emission Factor: 2.68 kg CO2e per L
- Result: 2,680 kg CO2e (or 2.68 tonnes)
```

### Unit Conversion (at ingestion)
```
normalized_quantity = quantity * conversion_factor(unit → normalized_unit)

Examples:
- Input: 5000 L of fuel → normalized to 5000 L (no conversion needed)
- Input: 25000 kg of material → already in normalized unit
- Input: 45000 kWh → normalized to 45000 kWh
```

### Quality Flags (before approval)
Analyst sees these flags:
- `missing_emission_factor`: No factor found for category/scope
- `invalid_quantity`: Negative or zero quantity
- `stale_data`: Activity > 90 days old
- `estimated_data`: Record marked as estimated
- `unit_conversion_failed`: Unknown unit pair

---

## Scope 3 Special Cases

### Flight Emissions
- **Distance calculation**: If airport codes provided, use great-circle distance
- **Missing distance**: Flag for analyst review
- **Cabin class multipliers**:
  - Economy: 0.090 kg CO2e per km per pax
  - Business: 0.270 kg CO2e per km per pax (3x higher due to seat width)
  - First: 0.360 kg CO2e per km per pax (4x higher)

### Hotel Emissions
- **Fixed rate per night**: 20-25 kg CO2e per night (average hotel)
- **Rationale**: Captures buildings energy, waste, laundry, food; not per-room but per-stay

### Ground Transport
- **Mode matters**:
  - Taxi/Uber: 0.192 kg CO2e per km
  - Public transit: 0.089 kg CO2e per km
  - Train: 0.041 kg CO2e per km

---

## Approval Workflow

### States
```
Pending Review
    ↓ (Analyst reviews)
    ├→ Approved (locked for audit, immutable)
    ├→ Rejected (returned for correction)
    └→ Flagged (issues detected, needs clarification)
```

### Immutability After Approval
Once `status='approved'`:
- Cannot edit: `updated_at` does not change
- Cannot delete: Soft delete blocked (raises exception)
- Cannot change scope/category: Audit trail would be broken

To fix approved records, admin must:
1. Create new record with corrected data
2. Soft-delete old record with reason
3. Both are visible in audit trail

---

## Queries & Performance

### Common Queries (all tenant-scoped)

```python
# Dashboard: Total CO2e by scope
Emission.objects
  .filter(tenant_id=X, is_deleted=False, status='approved')
  .values('scope')
  .annotate(total=Sum('co2e_kg'))

# Review queue: Pending by source
Emission.objects
  .filter(tenant_id=X, status='pending')
  .order_by('-activity_date')

# Audit report: All changes to a record
EmissionAuditLog.objects
  .filter(emission_id=X)
  .order_by('changed_at')

# Data quality: All flagged records
Emission.objects
  .filter(tenant_id=X, status__in=['flagged', 'pending'])
  .annotate(flag_count=JSONLength('quality_flags'))
  .filter(flag_count__gt=0)
```

---

## Error Handling & Recovery

### Duplicate Detection
On ingestion, check: `Emission.objects.filter(tenant_id=X, source_id=Y, data_source=Z)`

If exists:
- Option A: Skip (idempotent)
- Option B: Update if newer timestamp
- Current implementation: Unique constraint prevents insert; raises error caught in batch error_log

### Data Quality Issues
1. Missing emission factor → `quality_flags=['missing_emission_factor']`, status=`flagged`
2. Unit conversion failure → `quality_flags=['unit_conversion_failed']`, status=`pending`
3. Analyst must review before approval

### Audit Trail Completeness
Every change recorded:
- Original import: `action='created'`
- Each edit: `action='updated'`, `changed_fields` captured
- Approval: `action='approved'`, notes recorded
- Deletion: `action='deleted'`, reason captured

---

## Schema Evolution

### Migration Strategy
- New fields added as nullable with defaults
- Existing records unaffected
- Backward-compatible API responses

### Example: Adding "Confidence Score"
```python
# Add to model
confidence_score: decimal (default=1.0)

# Migration: backward compatible
# New imports: analyst sets 0.9 for estimated data
# Old records: default 1.0 (full confidence)
```

---

## Deployment Considerations

### Database Requirements
- PostgreSQL 12+: UUID, JSON, indexes
- Initial setup:
  ```sql
  CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
  python manage.py migrate
  python manage.py shell < initialize_reference_data.py
  ```

### Reference Data Initialization
```python
# Load standard emission factors (IPCC, EPA, IEA)
from breathe.utils import initialize_emission_factors, initialize_unit_conversions
initialize_emission_factors()
initialize_unit_conversions()
```

### Backup Strategy
- Nightly backup of all tables (especially EmissionAuditLog)
- Point-in-time recovery for audit trail disputes
- Approved emissions immutable; old versions unneeded

---

## Open Design Decisions

1. **Soft delete vs. hard delete**: Chose soft delete to preserve audit trail and allow recovery
2. **Emission factor lookup**: Chose by category+scope+year (not per-record custom factors, though feasible)
3. **Multi-facility emissions**: Currently optional; could be required per PM guidance
4. **Interval-based data**: Support period_start/period_end (utility bills); activity_date is single point (flights, fuel)

---
