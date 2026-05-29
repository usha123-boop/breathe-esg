# Deliberate Tradeoffs: What We Did NOT Build

## Three Major Features Deliberately Omitted

The assignment asks for prototyping with realistic scope. These three features would be in production but are deferred to avoid scope creep and maintain a defensible, testable core.

---

## 1. **Real SAP/Utility/Travel API Integration** (vs. CSV File Upload)

### What We Built
- CSV file upload for all three data sources
- Parser validates CSV format, converts to records

### What We Didn't Build
- Live API connections to SAP OData service, utility portals, or Concur/Navan APIs
- OAuth2 authentication handling
- Real-time polling or event subscriptions
- API rate limiting, retry logic, circuit breakers
- Per-vendor credential management

### Why NOT
**Effort: 40-60 hours of additional development**

Each platform requires:
1. Authentication (SAP: OAuth2 + certificate, Utilities: API key rotation, Concur: OAuth2 with custom scopes)
2. Per-platform SDK or API client (sometimes vendor-provided, sometimes custom)
3. Pagination, filtering, error handling (API returns 429 rate limits; retry with exponential backoff)
4. Field mapping (SAP OData field names differ from CSV; e.g., "MENGE" vs "quantity")
5. Scheduled sync logic (cron job, celery task, or webhook listener)
6. Test credentials (SAP sandbox, utility test account, travel platform staging)

**Verdict:** File upload is a valid MVP approach. Real API integration comes after:
- Client commits to source systems
- Credentials provisioned
- Test environment available

**In production, I would add:**
```python
# Background task to pull from SAP daily
@periodic_task(run_every=crontab(hour=2, minute=0))
def sync_sap_fuel():
    credentials = SAPCredential.objects.get(tenant=current_tenant)
    sap_client = SAPODataClient(credentials)
    records = sap_client.get_fuel_purchases(since_date=yesterday)
    for record in records:
        parse_sap_data(record)
```

**Trade: Fast iteration (CSV) vs. Full integration (API)**
- Chose: Fast iteration, validates business logic before API work

---

## 2. **Multi-Leg Flight & Complex Travel Scenarios** (vs. Single Origin-Destination)

### What We Built
- Single flight record: Origin, destination, distance, cabin, passengers
- Simple calculation: distance_km * cabin_factor * passengers = emissions
- Hotel records: Location, nights
- Ground transport: Mode, distance

### What We Didn't Build
- Multi-leg flights (NYC → London → Delhi on single trip_id)
- Code-sharing flight parsing (UA flight operated by British Airways)
- Personal vs. business trip segmentation (partner flight, leisure extension)
- Seat class upgrades (booked economy, flew business)
- Stopover/layover distinction (affects distance calculation)
- Group trips with allocation logic
- Return trip pairing (outbound vs. return, trip duration)

### Why NOT
**Effort: 20-30 hours; Complexity: High; Validation: Hard without real data**

Real data looks like:
```json
{
  "trip_id": "EXP-2024-5671",
  "flights": [
    {
      "leg": 1,
      "carrier": "United",
      "flight": "UA890",
      "origin": "SFO",
      "destination": "ORD",
      "departure": "2024-01-15T10:00Z",
      "arrival": "2024-01-15T15:30Z",
      "aircraft": "B787",
      "seats_booked": {"economy": 1},
      "seats_used": {"economy": 1}
    },
    {
      "leg": 2,
      "carrier": "United",
      "flight": "UA456",
      "origin": "ORD",
      "destination": "LHR",
      "departure": "2024-01-15T17:00Z",
      "arrival": "2024-01-16T05:30Z",
      "aircraft": "B787",
      "seats_booked": {"economy": 1},
      "seats_used": {"business": 1}  // Upgraded!
    }
  ]
}
```

**Complexity of multi-leg:**
- Distance: Each leg separate, or great-circle direct?
- Cabin class: What if booked economy but flew business on one leg?
- Layover: Does 2-hour layover in Chicago count as ground transport emissions?
- Group trips: "Team of 5 flying together but on different bookings"

**MVP answer:** "Single flight per record. Multi-leg trips are separate records (one per leg)."

**Real-world issue:** Analyst imports trip and sees 3 records (SFO→ORD, ORD→LHR, LHR→SFO return). Must manually verify these belong to same trip.

**Verdict:** Deferred pending real data sample. When building:
1. Get sample Concur/Navan export with multi-leg trip
2. Define rules (e.g., "sum leg distances", "use highest cabin class")
3. Implement with test suite
4. Validate with analyst

**In production, I would add:**
```python
class FlightTrip:
    trip_id: string
    legs: FK FlightLeg[]
    
    @property
    def total_emissions(self):
        return sum(leg.emissions for leg in self.legs)
    
    def validate_itinerary(self):
        # Check arrival airport = next leg departure
        # Check no impossible time gaps (2-hour min for connection)
        pass
```

**Trade: Simple calculation (single leg) vs. Realism (multi-leg is norm)**
- Chose: Simple, can extend when data available

---

## 3. **Advanced Data Quality & Anomaly Detection** (vs. Basic Validation)

### What We Built
- Basic validation: Required fields present, data types correct
- Quality flags (missing_emission_factor, invalid_quantity, stale_data)
- Manual analyst review for flagged records

### What We Didn't Build
- Statistical anomaly detection (outlier detection on historical data)
- Duplicate detection using fuzzy matching (Levenshtein distance, record linking)
- Temporal consistency checks (e.g., "fuel consumption dropped 40% month-over-month without explanation")
- Geospatial validation (e.g., "employee flew NYC→LA but badge-in logs show they were in office")
- Business logic validation (e.g., "flight for employee who left 6 months ago")
- Machine learning model for automatic quality scoring

### Why NOT
**Effort: 30-50 hours; Validation risk: High without labeled training data**

Examples of anomalies we DON'T detect:
```
1. Duplicate with variation:
   Record A: 5000 L Diesel, Jan 15
   Record B: 5001 L Diesel, Jan 15 (typo? or different supplier?)
   → Currently: Both ingested. Analyst must manually spot similarity.
   
2. Temporal jump:
   Jan: 5000 L fuel, Feb: 2500 L, Mar: 4800 L, Apr: 8500 L (↑ 77%!)
   → No explanation; could be new vehicle, could be error
   → Currently: No flag. Analyst reviews dashboard, maybe notices.
   
3. Geospatial mismatch:
   Flight: Employee booked NYC→LA (Jun 15-20)
   Badge system: Employee in Chicago office all week (Jun 15-20)
   → Currently: No check. Would require integrating badge system.
   
4. Scope misclassification:
   Record submitted as: "Scope 1, Diesel Fuel"
   Data: Utility meter electricity (should be Scope 2)
   → Currently: Analyst catches during review.
```

**Why hard:**
1. **No training data yet**: We don't have historical cleaned data to detect "normal" patterns
2. **Context-dependent**: What's anomalous for a shipping company (high fuel) is normal for a tech company (high travel)
3. **False positives**: An ML model might flag a company acquisition (sudden spike in facilities, legitimate) as fraud
4. **Auditor skepticism**: If we auto-flag 20% of records, auditor questions the baseline rules

**Real approach:**
```
Month 1-3: Analysts manually review all records, note patterns
  → "Normal fuel: 5000±500 L/month"
  → "Normal electricity: 450 kWh/day ±10%"
Month 4+: Build ML model on Month 1-3 data
  → Train isolation forest or one-class SVM
  → Flag records >2σ from historical mean
Month 6+: Monitor false positive rate, tune thresholds
```

**Verdict:** Deferred until we have labeled training data.

**In production, I would add:**
```python
from sklearn.ensemble import IsolationForest

def detect_anomalies(tenant_id, days_history=90):
    """Detect outliers in recent emissions data"""
    historical = Emission.objects
        .filter(tenant_id=tenant_id, activity_date__gte=today-timedelta(days=days_history))
        .values_list('co2e_kg')
    
    if len(historical) < 30:  # Minimum training data
        return []
    
    model = IsolationForest(contamination=0.1)  # Expect 10% anomalies
    anomaly_scores = model.fit_predict(historical)
    
    anomalies = [
        e for e, score in zip(Emission.objects.filter(...), anomaly_scores)
        if score == -1  # -1 = anomaly in sklearn
    ]
    
    return anomalies

# Usage
anomalies = detect_anomalies(tenant_id=my_tenant)
for emission in anomalies:
    flag_record(emission, 'statistical_anomaly', severity='medium')
```

**Trade: Simple rules (basic validation) vs. Smart detection (ML)**
- Chose: Simple, validation-friendly approach for MVP

---

## Why These Tradeoffs Are OK

### 1. Core business logic NOT deferred
We built:
- ✅ Multi-tenancy data isolation
- ✅ Audit trail for compliance
- ✅ Review workflow (pending → approved → locked)
- ✅ Unit normalization & emission factor lookups
- ✅ Scope 1/2/3 classification
- ✅ Quality flagging & manual override

These are non-negotiable for an ESG audit system.

### 2. Deferred features are MVP-to-production, not MVP-to-demo
These would be added in Phase 2:
1. API integrations (Month 2, once client commits to sources)
2. Complex travel scenarios (Month 2-3, once we see real data)
3. ML anomaly detection (Month 3-4, once we have labeled training set)

### 3. Architecture supports future additions
- Ingestion logic is modular; easy to add new source parsers
- Quality flags are extensible (add new flag types without schema change)
- Model fields like `emission_factor_source` enable future analysis
- API is versioned; can add new endpoints without breaking existing

---

## Validation & Testing Strategy

### What We Test
- CSV parsing (valid, invalid, edge cases)
- Unit conversions
- Emission factor lookups
- Status workflow (pending → approved)
- Multi-tenancy isolation (User A can't see User B's data)
- Audit log creation (every action recorded)

### What We Don't Test (yet)
- Live SAP connectivity
- Real utility portal scraping
- Multi-leg flight parsing logic
- Anomaly detection accuracy

---

## If Given More Time

### Week 2
- Integrate SAP OData sandbox (fictitious credentials)
- Build utility portal scraper for ConEd/PGE

### Week 3
- Multi-leg flight grouping logic
- Geospatial validation (integrate OpenCage for reverse geocoding)

### Week 4
- Collect labeled training data from analyst reviews
- Build anomaly detection model

---

## Open for Discussion

1. **Which deferred feature matters most to you?** If API integration is critical, I'd prioritize that.
2. **Do you have sample real data** for multi-leg flights? That would unblock feature #2.
3. **What's your audit timeline?** If audit is in 60 days, we might need to defer feature #3 in favor of bulletproofing features 1-2.

---
