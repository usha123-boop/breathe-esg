# Breathe ESG - Emissions Data Ingestion & Review System

A Django REST + React application for ingesting, normalizing, and reviewing emissions data from multiple sources (SAP, utilities, corporate travel) before audit submission.

**Submission Status:** ✅ Complete  
**Live Demo:** [Deployed URL - see submission email]  
**GitHub:** [Private repository - access granted to reviewers]

---

## Overview

This application solves the ESG carbon accounting challenge: data lives everywhere (SAP, utility portals, expense systems) in different shapes. Breathe ingests all of it, normalizes it to CO2e, and provides an analyst review dashboard before auditors lock it in.

**Key capabilities:**
- ✅ Multi-source ingestion (SAP, utilities, travel)
- ✅ Unit normalization & emission factor lookups
- ✅ Analyst review & approval workflow
- ✅ Complete audit trail (immutable once approved)
- ✅ Multi-tenant isolation
- ✅ Production-ready deployment

---

## Architecture

### Tech Stack

**Backend:**
- Django 4.2 (Python REST framework)
- PostgreSQL 15 (multi-tenant data model)
- Redis (optional, for background tasks)
- Celery (async ingestion, optional)

**Frontend:**
- React 18
- Recharts (data visualization)
- Axios (API client)
- CSS3 (responsive design)

**Deployment:**
- Docker & Docker Compose (local development)
- Railway / Render / Fly (production)
- Gunicorn (WSGI server)
- Nginx (reverse proxy)

### Directory Structure

```
breathe-esg/
├── breathe_project/           # Django project settings
│   ├── settings.py           # Configuration
│   ├── urls.py               # API routes
│   ├── wsgi.py               # Production entry point
│   └── asgi.py               # Async (optional)
│
├── breathe/                  # Core Django app
│   ├── models.py             # Data model (Tenant, Facility, Emission, etc)
│   ├── views.py              # API viewsets & ingestion logic
│   ├── serializers.py        # Request/response serialization
│   ├── auth_views.py         # Authentication endpoints
│   ├── utils.py              # Helpers (unit conversion, emission factors)
│   └── migrations/           # Database migrations
│
├── frontend/                 # React application
│   ├── src/
│   │   ├── pages/            # Dashboard, ReviewQueue, etc
│   │   ├── components/       # Reusable UI components
│   │   ├── styles/           # CSS modules
│   │   ├── App.jsx           # Main app component
│   │   └── index.jsx         # Entry point
│   ├── package.json          # Dependencies
│   └── Dockerfile            # Container image
│
├── documentation/
│   ├── MODEL.md              # Data model & design rationale
│   ├── DECISIONS.md          # Ambiguities resolved
│   ├── TRADEOFFS.md          # What was deliberately not built
│   └── SOURCES.md            # Research on real-world data formats
│
├── Dockerfile                # Backend container image
├── docker-compose.yml        # Local development stack
├── Procfile                  # Production deployment (Heroku/Railway)
├── requirements.txt          # Python dependencies
├── .env.example              # Environment variable template
└── manage.py                 # Django CLI
```

---

## Quick Start

### Local Development (Docker)

**Prerequisites:** Docker & Docker Compose

```bash
# 1. Clone repository
git clone <repo-url>
cd breathe-esg

# 2. Create environment file
cp .env.example .env

# 3. Start services
docker-compose up -d

# 4. Run migrations
docker-compose exec web python manage.py migrate

# 5. Create superuser
docker-compose exec web python manage.py createsuperuser

# 6. Load sample data
docker-compose exec web python manage.py seed_data

# 7. Open browser
# Backend: http://localhost:8000
# API Docs: http://localhost:8000/api/docs/
# Frontend: http://localhost:3000
```

### Manual Setup (Without Docker)

```bash
# 1. Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure database (.env file)
# Set DB_HOST=localhost, DB_NAME=breathe_esg, etc

# 4. Create PostgreSQL database
createdb breathe_esg

# 5. Run migrations
python manage.py migrate

# 6. Create superuser
python manage.py createsuperuser

# 7. Load reference data
python manage.py seed_data

# 8. Run development server
python manage.py runserver

# 9. In another terminal, start frontend
cd frontend
npm install
npm start
```

---

## API Documentation

Full API documentation available at:
- **Swagger UI:** http://localhost:8000/api/docs/
- **ReDoc:** http://localhost:8000/api/redoc/
- **Schema:** http://localhost:8000/api/schema/

### Example API Calls

**Login:**
```bash
curl -X POST http://localhost:8000/api/auth/login/ \
  -H "Content-Type: application/json" \
  -d '{
    "username": "analyst",
    "password": "demo123456",
    "tenant_slug": "demo-org"
  }'
```

**Upload SAP fuel data:**
```bash
curl -X POST http://localhost:8000/api/tenants/demo-org/ingestion/sap_fuel/ \
  -H "Authorization: Token YOUR_TOKEN" \
  -F "file=@sap_fuel_export.csv"
```

**List emissions (pending review):**
```bash
curl -X GET "http://localhost:8000/api/tenants/demo-org/emissions/?status=pending" \
  -H "Authorization: Token YOUR_TOKEN"
```

**Approve emission record:**
```bash
curl -X POST http://localhost:8000/api/tenants/demo-org/emissions/{emission_id}/approve/ \
  -H "Authorization: Token YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"review_notes": "Verified against invoice"}'
```

---

## Data Model

### Core Entities

**Emission** (main record)
- Source tracking: `data_source`, `source_id`, `ingestion_timestamp`
- Activity data: `activity_date`, `quantity`, `unit`
- Normalization: `normalized_quantity`, `normalized_unit`
- Emission factor: `emission_factor`, `emission_factor_source`
- Result: `co2e_kg` (auto-calculated)
- Classification: `scope` (1/2/3), `category`
- Review: `status` (pending/flagged/approved/rejected), `reviewed_by`
- Audit trail: Full history via EmissionAuditLog

See **MODEL.md** for complete data model documentation.

---

## Ingestion

### Supported Sources

1. **SAP Fuel & Procurement**
   - Format: CSV with plant_code, material, quantity, unit, date, fuel_type
   - Endpoint: `POST /api/tenants/{slug}/ingestion/sap_fuel/`
   - Sample file in `sample_data/sap_fuel.csv`

2. **Utility Electricity**
   - Format: CSV with meter_id, period_start, period_end, consumption, unit
   - Endpoint: `POST /api/tenants/{slug}/ingestion/utility_electricity/`
   - Sample file in `sample_data/utility_electricity.csv`

3. **Corporate Travel** (Flights, Hotels, Ground)
   - Format: CSV with trip details
   - Endpoints: `/api/tenants/{slug}/ingestion/travel_flight/`, etc.
   - Sample files in `sample_data/`

### Ingestion Workflow

```
1. Upload CSV file
   ↓
2. Parse & validate (catch format errors, missing required fields)
   ↓
3. Look up emission factors (flag if missing)
   ↓
4. Check for duplicates (unique constraint on source_id)
   ↓
5. Create Emission records with status='pending'
   ↓
6. Return batch summary (rows_successful, rows_failed, errors)
```

---

## Analyst Review Workflow

```
Pending Review
    ↓ (Analyst examines record)
    ├→ Approve (locked for audit, status='approved')
    ├→ Reject (returned for correction, status='rejected')
    └→ Flag (issues found, status='flagged')
```

**Quality flags** that prompt analyst review:
- `missing_emission_factor`: No emission factor found for category
- `invalid_quantity`: Negative or zero quantity
- `stale_data`: Activity > 90 days old
- `estimated_data`: Record marked as estimated
- `unit_conversion_failed`: Unknown unit pair

---

## Deployment

### Option 1: Railway (Recommended)

```bash
# 1. Push to GitHub
git push origin main

# 2. Connect to Railway
railway init

# 3. Set environment variables in Railway dashboard
# - DJANGO_SECRET_KEY
# - DATABASE_URL
# - ALLOWED_HOSTS

# 4. Deploy
railway up
```

### Option 2: Render

```bash
# 1. Create new Web Service on Render
# - Connect GitHub repository
# - Set environment variables
# - Deploy

# In render.yaml (included):
services:
  - type: web
    env: python
    startCommand: "gunicorn breathe_project.wsgi:application"
```

### Option 3: Fly.io

```bash
# 1. Install Fly CLI
curl -L https://fly.io/install.sh | sh

# 2. Login
flyctl auth login

# 3. Launch app
flyctl launch

# 4. Deploy
flyctl deploy
```

### Environment Variables (Production)

```bash
DEBUG=False
DJANGO_SECRET_KEY=<generate-with-secrets.token_urlsafe(50)>
ALLOWED_HOSTS=yourdomain.com,api.yourdomain.com
DATABASE_URL=postgresql://user:password@host:port/dbname
CELERY_BROKER_URL=redis://...
SECRET_KEY_LENGTH=50
```

---

## Testing

### Run Test Suite

```bash
# All tests
pytest

# Specific test file
pytest tests/test_models.py

# With coverage
pytest --cov=breathe
```

### Sample Test Cases Included

- ✅ Multi-tenancy isolation (User A can't see User B's data)
- ✅ CSV parsing (valid, invalid, edge cases)
- ✅ Unit conversions (L to mL, kg to g, etc)
- ✅ Emission factor lookups
- ✅ Status workflow (pending → approved → immutable)
- ✅ Audit log creation
- ✅ Soft delete (is_deleted flag preserved)

---

## Sample Data

Demo credentials:
- **Organization:** demo-org
- **Username:** analyst
- **Password:** demo123456

Sample CSV files for testing:
- `sample_data/sap_fuel.csv` — SAP fuel export (5 records)
- `sample_data/utility_electricity.csv` — Utility meter data (3 records)
- `sample_data/travel_flights.csv` — Corporate flights (4 records)
- `sample_data/travel_hotels.csv` — Hotel stays (3 records)

Load sample data:
```bash
python manage.py loaddata sample_data/demo_data.json
```

---

## Performance & Scaling

### Database Indexes
- `(tenant_id, status)` — Review queue queries
- `(tenant_id, activity_date)` — Timeline queries
- `(tenant_id, data_source)` — By-source filtering
- `(tenant_id, source_id, data_source)` — Duplicate detection

### Caching (Optional)
```python
# Cache emission factors (rarely change)
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.redis.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
    }
}

# Usage:
factor = cache.get_or_set(f'emission_factor_{category}_{year}', 
    lambda: EmissionFactor.objects.get(...), 
    timeout=86400)  # 24 hours
```

### Bulk Operations
For large ingestions (100k+ records):
```python
# Use bulk_create (faster than individual saves)
Emission.objects.bulk_create(emission_list, batch_size=1000)
```

---

## Monitoring & Logging

### Logging Configuration
- **Console:** Real-time logs during development
- **File:** `/app/logs/breathe.log` (persistent)
- **Level:** DEBUG (dev), INFO (prod)

### Key Metrics to Monitor
```python
# Dashboard queries
SELECT scope, COUNT(*), SUM(co2e_kg) FROM breathe_emission 
WHERE tenant_id = X AND status = 'approved' GROUP BY scope;

# Ingestion health
SELECT data_source, COUNT(*), ROUND(100.0*COUNT(CASE WHEN status='approved'), 2) 
FROM breathe_emission WHERE tenant_id = X GROUP BY data_source;

# Approval velocity
SELECT DATE_TRUNC('day', reviewed_at), COUNT(*) FROM breathe_emission 
WHERE tenant_id = X AND reviewed_at IS NOT NULL GROUP BY DATE_TRUNC('day', reviewed_at);
```

---

## Known Limitations & Future Work

### Known Limitations (by design)

1. **Real API integration deferred** — Currently CSV upload only. Real SAP/Concur APIs would require credentials + integration testing.
2. **Multi-leg flights not parsed** — Single origin-destination only. Would add after seeing real Concur/Navan data.
3. **No ML anomaly detection** — Basic validation only. Would add after 3+ months of labeled training data.
4. **Manual facility mapping** — Utilities & SAP don't auto-map to facilities; analyst must configure.

See **TRADEOFFS.md** for rationale.

### Future Enhancements

- [ ] Real-time SAP OData API integration
- [ ] Smart meter API connections (Enel X, Siemens)
- [ ] Multi-leg flight parsing & grouping
- [ ] Geospatial validation (employee badge + flight location)
- [ ] Statistical anomaly detection (Isolation Forest)
- [ ] Science-Based Target (SBT) scenario modeling
- [ ] Carbon offset marketplace integration
- [ ] Regulatory reporting templates (GRI, TCFD, SBTi)
- [ ] Supplier ESG platform integration

---

## Troubleshooting

### Database Connection Error
```bash
# Check PostgreSQL is running
psql -U postgres -d breathe_esg -c "SELECT 1;"

# If fails, restart Docker
docker-compose restart db
```

### "Token not found" during login
```bash
# Create auth token for user
python manage.py shell
>>> from django.contrib.auth.models import User
>>> from rest_framework.authtoken.models import Token
>>> user = User.objects.get(username='analyst')
>>> token, created = Token.objects.get_or_create(user=user)
>>> print(token.key)
```

### CORS errors in frontend
```bash
# Verify CORS_ALLOWED_ORIGINS in .env includes frontend URL
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:8000
```

### CSV upload fails with "Unknown unit"
```bash
# Check unit conversion reference table
python manage.py shell
>>> from breathe.models import UnitConversion
>>> UnitConversion.objects.filter(source_unit='L').values_list('target_unit')
```

---

## Support & Contact

For questions during review:
- **Architecture:** See MODEL.md for data model rationale
- **Decisions:** See DECISIONS.md for every ambiguity resolved
- **Tradeoffs:** See TRADEOFFS.md for what was deliberately omitted
- **Research:** See SOURCES.md for real-world data format analysis

---

## License

Proprietary — Breathe ESG. All rights reserved.

---

## Submission Checklist

- ✅ Working app deployed to production (Railway/Render/Fly)
- ✅ MODEL.md: Data model with multi-tenancy, audit trail, scope classification
- ✅ DECISIONS.md: Every ambiguity (SAP format, utility format, travel data, deduplication, scope rules)
- ✅ TRADEOFFS.md: Three deliberate omissions (API integration, multi-leg flights, ML anomaly detection)
- ✅ SOURCES.md: Research on real-world formats, sample data rationale, deployment challenges
- ✅ GitHub repository private, shared with saurav@, rahul@, shivang@breatheesg.com
- ✅ Live app URL in submission email
- ✅ Demo credentials included (analyst / demo123456 / demo-org)
- ✅ All credentials needed for login in submission email

---
