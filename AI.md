# MedLedger

## Description

MedLedger is a Python/Django hospital management system and clinical operations dashboard that helps healthcare staff manage patient registrations, doctor appointment scheduling, prescription generation, GST-compliant billing, and audited financial analytics. It runs as an offline-first Django web application with server-rendered HTML dashboard interfaces, RESTful API endpoints via Django REST Framework, and an in-memory SQLite database.

## Repository Structure

```text
medledger/
├── challenge.json                 Isolated runtime, port, build, start, and test configuration
├── conftest.py                    Global pytest fixtures, client helpers, and mock database seeds
├── manage.py                      Django management script and CLI entrypoint
├── pytest.ini                     Pytest configuration and test runner settings
├── requirements.txt               Python dependencies and library specifications
├── accounts/
│   ├── models.py                  Custom User model with RBAC roles (admin, doctor, receptionist)
│   ├── permissions.py             DRF role-based permission classes (IsAdminRole, IsDoctorRole, etc.)
│   ├── serializers.py             User authentication and token serializers
│   ├── urls.py                    Authentication and profile API routes
│   └── views.py                   JWT token authentication and user management views
├── appointments/
│   ├── filters.py                 Appointment date range and status filtering logic
│   ├── models.py                  Doctor registry and patient consultation data models
│   ├── serializers.py             DRF serializers for appointments and doctor metadata
│   ├── urls.py                    API URL routing for appointments endpoints
│   └── views.py                   Appointment CRUD ViewSets and deletion lifecycle handling
├── billing/
│   ├── models.py                  Patient invoice model with discount, GST, and net total calculation
│   ├── serializers.py             DRF serializers for billing records and payment states
│   ├── urls.py                    API routing for billing and revenue analytics endpoints
│   └── views.py                   Invoice ViewSets and role-restricted revenue analytics API
├── core/
│   └── views.py                   Health check and base system utilities
├── dashboard/
│   ├── urls.py                    Dashboard view routing
│   └── views.py                   Server-rendered HTML views and hospital administration dashboard
├── fixtures/
│   └── seed_data.json             Deterministic database seed fixtures (patients, doctors, appointments)
├── medledger/
│   ├── asgi.py                    ASGI configuration for asynchronous web servers
│   ├── settings.py                Django project settings, installed apps, and challenge mode toggles
│   ├── urls.py                    Root URL configuration mapping dashboard views and API routers
│   └── wsgi.py                    WSGI application configuration for web deployment
├── patients/
│   ├── models.py                  Patient registry models with medical and demographic fields
│   ├── pagination.py              Custom pagination class controlling patient page size
│   ├── serializers.py             DRF serializers for patient record serialization
│   ├── services.py                Patient search service managing queries and loading states
│   ├── urls.py                    API URL routing for patient endpoints
│   └── views.py                   Patient CRUD ViewSets and live search endpoints
├── prescriptions/
│   ├── models.py                  Clinical prescription records tied to appointments
│   ├── pdf.py                     ReportLab PDF generator for clinical prescriptions
│   ├── serializers.py             DRF serializers for prescription regimens
│   ├── urls.py                    Prescription API route definitions
│   └── views.py                   Prescription ViewSets and PDF download actions
├── static/
│   ├── css/                       Local Bootstrap styles and custom hospital UI stylesheets
│   ├── img/                       Brand assets, iconography, and medical illustration graphics
│   └── js/
│       ├── bootstrap.bundle.min.js Local Bootstrap JavaScript bundle
│       └── patient_search.js      Live asynchronous search handling and UI state management
├── templates/                     Django server-rendered dashboard, appointment, and billing templates
├── tests/
│   ├── test_api.py                Authentication tokens and API contract verification
│   ├── test_appointments.py        Appointment booking, filtering, and deletion tests
│   ├── test_billing.py            Invoice calculation, tax, and discount verification tests
│   ├── test_patients.py           Patient registry CRUD, pagination, and filtering tests
│   ├── test_permissions.py        Role-based access control and revenue security tests
│   └── test_search.py             Live patient search, error recovery, and loading state tests
└── README.md                      Candidate-facing application and bug reproduction guide
```

## Bugs and Bug Locations

These are the six behavioral bug surfaces covered by the challenge. The named locations identify the owning implementation areas for debugging and review.

### 1. Appointment date range excludes boundary date

- **Bug location:** `appointments/filters.py`, `AppointmentFilter`
- **How to observe it:** Open the Appointment Scheduling dashboard, set Start Date to `2026-09-01` and End Date to `2026-09-30`.
- **Failure:** The filter uses an exclusive `'lt'` lookup on `end_date`, omitting appointments scheduled on the exact ending boundary date (`2026-09-30`).
- **Expected:** The filter uses an inclusive `'lte'` lookup for `end_date`, ensuring all appointments scheduled on the boundary date are included in the query results.

### 2. Pagination skips the last patient on every page

- **Bug location:** `patients/pagination.py`, `CustomPatientPagination::paginate_queryset`
- **How to observe it:** Open the Patient Registry on Page 1, count displayed patient records (9 instead of 10), then navigate to Page 2.
- **Failure:** When a page reaches the configured page size of 10, slicing drops the last item (`page_data[:-1]`), displaying only 9 patients and skipping the 10th patient record entirely.
- **Expected:** Pagination returns all 10 patient records per page without truncation, preserving complete record sequences across pagination pages.

### 3. Removing appointment deletes wrong record

- **Bug location:** `appointments/views.py`, `AppointmentViewSet::destroy`
- **How to observe it:** Open Appointment Scheduling, select a specific appointment, and click Delete.
- **Failure:** The endpoint executes `Appointment.objects.first().delete()`, deleting the first record in the database instead of the targeted appointment selected by the user.
- **Expected:** The endpoint calls `self.get_object()` and `self.perform_destroy(instance)`, deleting strictly the selected appointment identified by its primary key / UUID.

### 4. Patient search remains loading forever after server error

- **Bug location:** `patients/services.py`, `PatientSearchService::execute_search`; UI display surfaced by `static/js/patient_search.js`
- **How to observe it:** Trigger a patient search query during a simulated database error or network timeout.
- **Failure:** Exception handling fails to reset `is_loading` to `False` (and client JavaScript lacks error recovery), leaving the search button permanently disabled and the spinner spinning indefinitely.
- **Expected:** Search errors reset `is_loading = False`, hide the loading indicator, and restore the search button and input to an interactive state.

### 5. Billing calculation returns incorrect invoice total

- **Bug location:** `billing/models.py`, `Bill::calculate_total`
- **How to observe it:** Inspect an invoice with Base Amount = ₹10,000, Discount = 10%, and GST = 18%.
- **Failure:** Tax is applied to the gross pre-discount amount and calculated with floating-point arithmetic, yielding ₹10,800.00 instead of ₹10,620.00 and introducing rounding drift.
- **Expected:** The discount is deducted first from the base amount to determine the taxable net value (₹9,000.00), and 18% GST (₹1,620.00) is applied to that net value using exact `Decimal` arithmetic, producing a correct total of ₹10,620.00.

### 6. Unauthorized receptionist can access admin revenue reports

- **Bug location:** `billing/views.py`, `RevenueAnalyticsView`
- **How to observe it:** Authenticate using a receptionist user account and navigate to `/api/billing/revenue/` or the Revenue Analytics dashboard.
- **Failure:** The view specifies `permission_classes = [IsAuthenticated]`, allowing non-admin roles (receptionists and doctors) to view confidential hospital revenue statistics.
- **Expected:** The view enforces `permission_classes = [IsAuthenticated, IsAdminRole]`, returning HTTP 403 Forbidden for unauthorized roles while granting access only to administrators.

## Expected Behaviour After Fixing All Bugs

- Filtering appointments by date range includes records scheduled on the selected ending boundary date.
- Patient registry pagination returns exactly 10 records per page without skipping records.
- Deleting an appointment strictly removes the targeted appointment by its UUID.
- Patient search cleanly resets loading states and re-enables controls when a query encounters an error.
- Invoices calculate GST on the discounted taxable amount using Decimal arithmetic (₹10,000 with 10% discount and 18% GST correctly equals ₹10,620.00).
- Receptionists and doctors receive HTTP 403 Forbidden when requesting confidential hospital revenue reports.
- Running `pytest` passes all 36 tests with `"Passed": 36`, `"Failed": 0`, and exits with code `0`.
