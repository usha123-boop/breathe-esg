# Design Decisions & Ambiguities Resolved

## Assignment Ambiguities

### 1. **SAP Export Format**

**Ambiguity:** Assignment says "SAP exports are not friendly" and lists IDoc, flat file, OData. Which one?

**Decision:** OData service with CSV fallback

**Rationale:**
- **OData (primary)**: Most modern SAP deployments expose OData REST API. Real integration would use this.
  - Advantages: Real-time, structured JSON, easy pagination, filtering
  - Challenges: Authentication (OAuth2), requires IT setup
  - Real-world: SAP S/4HANA primarily uses OData /sap/opu/odata/sap/
  
- **CSV fallback**: For legacy systems or manual exports
  - What we actually ingest: Facility exports from SAP BW/BI or transaction MM (Inventory Management)
  - Columns: plant_code, material_description, quantity, unit, document_date, fuel_type

**Real-world data shapes encountered:**
- Plant codes like "1000", "2000" don't map to facility names; master data required
- Units inconsistent: sometimes "L", sometimes "Liters", sometimes "LTR"
- Dates in various formats: YYYY-MM-DD, DD.MM.YYYY (German), MM/DD/YYYY (US)
- German column headers in some configs: "Lieferdatum" (delivery date), "Materialgruppe" (material group)
- Data 2-3 weeks behind actual fuel purchase due to reconciliation lag

**What we handle:** CSV export assuming English headers, ISO 8601 dates, standard units (L, kg, m³)

**What we don't:** IDoc parsing (proprietary binary), real-time SAP API integration (requires credentials, OAuth)

**What I'd ask the PM:**
- "Do you have SAP credentials we can use, or should we assume file exports?"
- "What's your current plant master data? Can we get a snapshot?"
- "What unit conversions do you need beyond L, kg, m³?"

---

### 2. **Utility Data Format**

**Ambiguity:** Portal CSV, PDF bill, API. Which?

**Decision:** Portal CSV export

**Rationale:**
- **Portal CSV (primary)**: Most accessible for non-technical facilities teams
  - Widely available: Every major utility (e.g., ConEd, PGE, Eskom) offers web portal exports
  - Format: Standard columns (meter ID, period, consumption, rate)
  - Challenges: Manual export process, no real-time updates
  
- **PDF bills**: OCR-able but error-prone; not our primary format
  - Would require: Regex patterns per utility (each has different format)
  - Real data: Inconsistent layouts, merged cells, footnotes
  - Not pursuing: Too fragile
  
- **API**: Available from newer smart meter providers (Enel X, EDF, etc.)
  - Real-world: Usually requires utility-specific credentials
  - Not pursuing: Limited adoption, maintenance burden per utility

**Real-world utility data shapes:**
- Meter IDs: Alphanumeric strings like "METER-001", sometimes facility codes
- Billing periods: Don't align with calendar months (e.g., Jan 5 - Feb 4)
- Units: Mostly kWh, some MWh, occasionally GJ (gas utilities)
- Tariff structures: Time-of-Use (peak/off-peak), fixed rates, tiered rates
- Missing data: Estimated readings flagged with "E", actual readings with "A"
- Renewable energy: Separate line items for solar/wind if on-site generation

**Sample CSV we accept:**
```
Meter ID,Meter Description,Period Start,Period End,Units,Consumption,Tariff Type,Cost
METER-001,Building A Main,2024-01-01,2024-01-31,kWh,45000,Standard Peak,6750.00
METER-002,Building B,2024-01-01,2024-01-31,kWh,8500,Standard Off-Peak,850.00
```

**What we don't handle:** PDF parsing, API authentication, estimated vs. actual reconciliation

**What I'd ask the PM:**
- "Which utility companies do your facilities use? (ConEd, PGE, Duke, etc.)"
- "Can facilities export CSVs from their utility portals, or do we need to set up portal scraping?"
- "Do you have historical data to backfill, or starting from now?"
- "How do you currently handle estimated readings? Should we flag and require approval?"

---

### 3. **Corporate Travel Data Format**

**Ambiguity:** Concur, Navan, or generic platform API?

**Decision:** Navan-like REST API structure (JSON), with fallback CSV for flight/hotel/ground separately

**Rationale:**
- **Navan/Concur APIs**: Industry standard for expense reporting
  - Advantages: Structured data, standardized fields, real-time
  - Challenges: Authentication required, API rate limits, per-platform implementation
  - Real-world: Most enterprises use one of: Concur, Navan, TravelPerk, BCD
  
- **What's realistic**: Expense reports are created by employees in platform, then exported/accessible via API
  - Flights: Departure, destination, cabin class, carrier, distance
  - Hotels: Location, check-in, check-out, number of nights
  - Ground transport: Mode (taxi, rental, transit), distance, location

**Real-world travel data challenges:**
- **Flights**: 
  - Code-sharing flights: Multiple carriers for one trip (AA flight but operated by British Airways)
  - Missing distance: Not always provided; must calculate from airport codes
  - Cabin class interpretation: "Premium Economy" vs "Business" multipliers differ
  - Connections: Multi-leg trips hard to parse; which legs are business vs. personal?
  
- **Hotels**:
  - Chain vs. independent: Different emission factors
  - Days booked vs. nights stayed: System often shows checkout date same as overnight count
  - Corporate rates vs. personal: Some business travel expenses are personal (partner flights)
  
- **Ground transport**:
  - Mode ambiguity: "Lyft" vs "Uber" both captured as ride-share
  - Distance missing: Often only location given ("NYC taxi" — to where?)
  - Corporate vs. personal: Entire trip might be personal; only some segments business

**Sample data structure we handle:**
```json
[
  {
    "trip_id": "EXP-2024-001",
    "employee_id": "EMP-1234",
    "trip_start": "2024-01-15",
    "trip_end": "2024-01-18",
    "expenses": [
      {
        "type": "flight",
        "carrier": "United",
        "origin": "SFO",
        "destination": "NYC",
        "distance_km": 4140,
        "cabin": "economy",
        "passengers": 1
      },
      {
        "type": "hotel",
        "location": "New York, NY",
        "check_in": "2024-01-15",
        "nights": 3
      },
      {
        "type": "ground_transport",
        "mode": "taxi",
        "distance_km": 45
      }
    ]
  }
]
```

**What we don't handle:** Multi-leg flight parsing (assume single origin-destination), personal vs. business trip split, code-share complexity

**What I'd ask the PM:**
- "Which travel platform does your company use? (Concur, Navan, TravelPerk, etc.)"
- "Can we pull trip data from the API, or do we need CSV exports?"
- "How do you currently handle business vs. personal portions of trips? (e.g., partner flying to destination)"
- "Do you have baseline distance data for frequent routes, or should we always calculate?"

---

## Technical Ambiguities

### 4. **Scope Assignment for Electricity**

**Ambiguity:** If a company has on-site solar, is that Scope 1 or Scope 2?

**Decision:** Grid electricity = Scope 2. On-site renewable = Scope 1 (generation).

**Rationale:**
- **Scope 2 (purchased electricity)**: Grid-sourced power, regardless of mix (coal/gas/renewable)
  - Emission factor: Region-specific grid mix (US avg ~0.42 kg CO2e/kWh, EU ~0.25)
  - Real-world: Utilities provide grid mix annually; may vary by region/season
  - Handling: Assign all utility meter data to Scope 2
  
- **Scope 1 (direct generation)**: On-site solar, wind, geothermal
  - Emission factor: Often 0 for renewable, or small for T&D losses
  - Real-world: Separate utility line items or building management system logs
  - Handling: If utility CSV includes "Solar Generated: 5000 kWh", assign Scope 1
  
- **Scope 3 (purchased renewable credits)**: RECs, PPAs, green tariffs
  - Rationale: Utility offers "green" electricity at premium
  - Handling: If noted, offset Scope 2 emissions with REC quantity

**Current implementation:** All utility_electricity ingestions → Scope 2, category "Electricity - Grid Mix"

**What I'd ask the PM:**
- "Do any facilities have on-site solar/wind? How is it metered?"
- "Are you purchasing RECs or on a green tariff? Should we separate these?"
- "Which grid region(s) do your facilities operate in? I need region-specific emission factors."

---

### 5. **Unit Normalization Strategy**

**Ambiguity:** Should all units convert to SI units (kg, kWh), or keep as-is?

**Decision:** Keep original units; normalize only for calculation. Store both.

**Rationale:**
- **Why normalize**: Enables consistent calculations (e.g., all fuel in kg, all energy in kWh)
- **Why keep original**: Auditors expect to see what was reported (e.g., "5000 L diesel", not "5900 kg diesel")
- **Implementation**: 
  - Store `quantity` + `unit` (original)
  - Store `normalized_quantity` + `normalized_unit` (for calculation)
  - `co2e_kg = normalized_quantity * emission_factor`

**Example:**
```
Original: 5000 L Diesel
Normalized: 5000 L (no conversion needed, already in normalized unit)
Emission Factor: 2.68 kg CO2e per L
Result: 5000 * 2.68 = 13,400 kg CO2e

vs.

Original: 5,900 kg Diesel (if supplier provided in kg)
Normalized: 5,900 kg
Factor: 2.27 kg CO2e per kg (different factor per unit!)
Result: 5,900 * 2.27 = 13,393 kg CO2e (roughly same, small difference)
```

**Conversion table we maintain:**
- L ↔ mL ↔ gallon
- kg ↔ g ↔ tonne
- kWh ↔ MWh ↔ GJ
- km ↔ m ↔ miles

**What we don't handle:** Converting between fundamentally different units (L to kg), which requires density. This is an error; we flag for analyst review.

**What I'd ask the PM:**
- "What units do your suppliers typically use for fuel? (L, gallon, kg, m³)"
- "Should we auto-reject records with unit mismatches, or flag for analyst override?"

---

### 6. **Missing Emission Factors**

**Ambiguity:** If we can't find an emission factor for a fuel type or scope, what do we do?

**Decision:** Flag the record with `quality_flags=['missing_emission_factor']` and `status='flagged'`. Don't calculate co2e. Analyst must provide or reject.

**Rationale:**
- **Option A (reject)**: Refuse to ingest. Pro: Forces data completeness. Con: Loses data, frustrates users.
- **Option B (flag for review)**: Ingest with co2e_kg=0, wait for analyst. Pro: Preserves data, allows override. Con: Risk of forgetting to fill in.
- **Option C (estimate)**: Use closest factor (e.g., "Unknown Fuel" → avg of all fuels). Pro: Always calculates. Con: Misleading numbers, audit risk.

**Current:** Option B. Analyst sees:
```
Record: 1000 L of "BioFuel" (unknown)
Status: Flagged
Quality Flags: [missing_emission_factor]
CO2e: 0 kg (pending factor)
Action required: Analyst chooses
  1. Provide factor
  2. Clarify fuel type (maybe it's Biodiesel? 2.61 kg CO2e/L)
  3. Reject record
```

**Real-world scenarios:**
- New fuel type (e.g., sustainable aviation fuel, SAF)
- Regional variant (e.g., electricity grid mix for country we don't cover)
- Custom supplier factor (e.g., "Our natural gas comes from 40% renewable sources")

**What I'd ask the PM:**
- "What's your target data completeness? 100% approved or acceptable to have ~5% pending factor review?"
- "Do you have custom emission factors from your science team? Can we load them as reference data?"
- "For new fuel types, who's the decider: your team or an auditor?"

---

### 7. **Deduplication & Duplicate Detection**

**Ambiguity:** If the same record is ingested twice (e.g., analyst re-uploads same file), what happens?

**Decision:** Unique constraint on `(tenant_id, source_id, data_source)`. Duplicate insert fails. Logged in batch error_log.

**Rationale:**
- **source_id** is the unique identifier from the originating system (e.g., SAP document number, utility meter+period)
- Prevents accidental double-counting when a file is uploaded twice
- Allows idempotent uploads: "re-run this upload; if records exist, skip"

**Example:**
```
Batch 1: Upload SAP_export_2024-01-15.csv
  Source IDs: SAP-FUEL-20240115-1, SAP-FUEL-20240115-2, ...
  Status: 10 rows successful

Batch 2: Upload same file again (analyst mistake)
  Source ID SAP-FUEL-20240115-1 already exists
  Error: "Duplicate source_id. Skipped 10 rows."
  Status: 0 rows successful, 10 rows failed
```

**What we don't handle:** Fuzzy deduplication (e.g., "same fuel, same date, but slightly different quantity — probably the same purchase"). Analyst must manually flag as duplicate and soft-delete one.

**What I'd ask the PM:**
- "Do your source systems have audit IDs we can use? (SAP movement document, utility invoice number)"
- "How often do you expect duplicate uploads? Should we warn before ingesting the same file twice?"

---

### 8. **Multi-Facility Emissions**

**Ambiguity:** An electricity meter serves multiple buildings. Which facility should it belong to?

**Decision:** Optional facility assignment. Emissions can be facility-less if not mappable.

**Rationale:**
- **Option A (required)**: Every emission must have a facility. Pro: Clean reporting. Con: Breaks on shared infrastructure.
- **Option B (optional, current)**: Facility is nullable. Pro: Flexible, allows shared assets. Con: Harder to allocate to one building.
- **Option C (many-to-many)**: Meter → Multiple facilities with allocation %. Pro: Most accurate. Con: Complex data model, ingestion overhead.

**Current implementation:** Facility is ForeignKey(null=True, blank=True). Emissions can exist without facility mapping.

**Real-world cases:**
- Central chiller plant serves 3 buildings; meter shows total energy
- Corporate office building with shared HVAC; difficult to allocate per floor
- Travel data (flights, hotels) don't have a "facility"; they have locations

**UI consequence:** Dashboard shows:
- Emissions by scope (works, facility-less emissions counted)
- Emissions by facility (works, null facility shown separately)

**What I'd ask the PM:**
- "Do you need facility-level reporting? Or is company-level sufficient?"
- "For shared infrastructure, how do you currently allocate (equal split, floor area, occupancy)?"

---

### 9. **Analyst Review Permissions**

**Ambiguity:** Can a user approve their own uploads?

**Decision:** Yes. No special permission needed.

**Rationale:**
- **Option A (no self-approval)**: Require peer review. Pro: Segregation of duties. Con: Slower, bottleneck for small teams.
- **Option B (allow, current)**: Anyone can approve. Pro: Flexible, fast. Con: Reduced audit trail credibility.
- **Option C (role-based)**: Analyst can review, Senior Analyst can approve. Pro: Best practice. Con: More complex.

**Current:** Option B. `reviewed_by` field records who approved; auditors can detect same person uploading and approving.

**Real-world:** Small companies (2-3 analysts) can't afford peer review. Large companies use Option C.

**What I'd ask the PM:**
- "How many people are on your ESG team? What's your review capacity?"
- "Do you need segregation of duties (upload ≠ approve)?"

---

### 10. **Historical Data & Backfill**

**Ambiguity:** Do you need historical emissions (last 5 years) or just going forward?

**Decision:** Architecture supports both. Ingestion accepts any activity_date. Dashboard filters to recent (UI default last 12 months).

**Rationale:**
- **Going forward (simpler)**: Start tracking from deployment date. Pro: Less work. Con: Baseline year incomplete.
- **Historical (auditor-preferred)**: Backfill 5 years. Pro: Full baseline, trend analysis. Con: Data recovery effort, might be incomplete.

**Current implementation:**
```python
# Ingestion accepts any date
emission.activity_date = row['activity_date']  # Can be 2019-01-01 or 2024-01-01

# Dashboard default filters to last 12 months
Emission.objects.filter(
  activity_date__gte=today - timedelta(days=365)
)
# But query can override to show all years
```

**What I'd ask the PM:**
- "What's your baseline year for ESG reporting? (e.g., 2020 for Science-Based Target)"
- "Do you have historical data we can backfill, or starting from Jan 2024?"

---

## Open Questions

1. **Scope 3 completeness**: Are you tracking Scope 3 for: business travel (yes), supply chain (no), waste (no), commuting (no)? This affects what sources we need.

2. **Regulatory reporting**: Which standard do you follow? (GRI, TCFD, SBTi, ISO 14064). This affects data retention, audit requirements, emission factor versions.

3. **Audit frequency**: Annual, or continuous audits? This affects how "locked" approved records need to be.

4. **Vendor/supplier integration**: Should we integrate with supplier ESG platforms later? This affects data model extensibility.

5. **Renewables & RECs**: Are you carbon-neutral or carbon-negative? Do you purchase offsets? How do we model this?

6. **Science-Based Targets**: Are you setting SBTs? This affects scenario modeling (e.g., "show 50% reduction target by 2030").

---
