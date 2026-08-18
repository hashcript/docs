# E-Growth Mobile Backend API Integration Guide

## Purpose

This document describes the E-Growth backend APIs available to the mobile application, what each endpoint does, when the mobile application should use it, and the current implementation/deployment status.

The backend is being delivered incrementally.

Some APIs are already merged and can be tested against the development environment.

Other APIs have been implemented and fully validated but are still awaiting merge/deployment.

---

# 1. Development Environment

## Swagger / API Documentation

The live development Swagger documentation is available at:

```text
https://e-growth.dev.educateapps.work/docs#/
```

Use Swagger as the final source of truth for APIs that are **currently deployed**.

---

## API Host

```text
https://e-growth.dev.educateapps.work
```

## API Base URL

```text
https://e-growth.dev.educateapps.work/api/v1
```

Flutter should configure this through environment configuration.

Example:

```text
API_BASE_URL=https://e-growth.dev.educateapps.work/api/v1
```

Do NOT hardcode the URL directly inside:

- screens
- repositories
- controllers
- DTOs
- models

---

# 2. Current API Status

There are three statuses used in this document.

### ✅ LIVE / MERGED

Backend implementation has been merged and can be integrated/tested against the deployed development environment once deployment is healthy.

### 🟡 DEVELOPED / VALIDATED

Backend implementation is complete and tested, but the feature branch has not yet been merged/deployed.

The mobile developer can prepare models/repositories for it, but should NOT assume the endpoint exists on the live environment until backend confirms deployment.

### ⏳ NOT READY

Backend implementation has not yet been completed.

Do not integrate these APIs yet.

---

# 3. Current API Readiness Summary

| Capability | Endpoint | Status |
|---|---|---|
| Clerk authentication | Clerk SDK/session | ✅ Available |
| Mobile product access | `GET /auth/product-access` | ✅ LIVE / MERGED |
| Product list | `GET /products` | ✅ LIVE / MERGED |
| Product creation | `POST /products` | ✅ LIVE / MERGED |
| Product memberships | `GET /products/{id}/memberships` | ✅ LIVE / MERGED |
| Add product member | `POST /products/{id}/memberships` | ✅ LIVE / MERGED |
| Update membership | `PATCH /products/{id}/memberships/{membership_id}` | ✅ LIVE / MERGED |
| Locations | `GET /locations` | 🟡 DEVELOPED / VALIDATED |
| Location detail | `GET /locations/{id}` | 🟡 DEVELOPED / VALIDATED |
| Venues | `GET /venues` | 🟡 DEVELOPED / VALIDATED |
| Venue detail | `GET /venues/{id}` | 🟡 DEVELOPED / VALIDATED |
| Bootcamps | `GET /bootcamps` | 🟡 DEVELOPED / VALIDATED |
| Bootcamp detail | `GET /bootcamps/{id}` | 🟡 DEVELOPED / VALIDATED |
| Participants | `GET /participants` | 🟡 DEVELOPED / VALIDATED |
| Participant detail | `GET /participants/{id}` | 🟡 DEVELOPED / VALIDATED |
| Call history | `GET /participants/{id}/calls` | ⏳ NOT READY |
| Record mobilisation call | `POST /participants/{id}/calls` | ⏳ NOT READY |
| Dynamic Forms API | — | Deferred |
| Generic Submission API | — | Deferred |

---

# 4. Authentication

The backend uses **Clerk** for authentication.

The Flutter application should authenticate the user using Clerk and obtain a Clerk session token.

Authenticated requests should send:

```http
Authorization: Bearer <clerk-session-token>
```

Example:

```http
GET /api/v1/auth/product-access
Authorization: Bearer eyJ...
Accept: application/json
```

The application must NOT implement another username/password authentication mechanism against the E-Growth backend.

---

# 5. Authentication Architecture

The intended mobile authentication flow is:

```text
Mobile App
    ↓
Clerk Sign-In
    ↓
Clerk Session
    ↓
Clerk Session Token
    ↓
Authorization: Bearer <token>
    ↓
E-Growth API
    ↓
Resolve Product Membership
    ↓
Allow / Deny Access
```

Clerk answers:

> Who is this person?

The E-Growth database answers:

> Which product can this person access, and what role do they have there?

The mobile client must never determine authorization from email, local configuration or Clerk metadata by itself.

---

# 6. Mobile Roles

The current mobile roles are:

```text
AGENT
ADMIN
```

Where:

```text
AGENT = Mobilizer
```

These are **product roles**.

They should not be confused with existing web application roles such as:

```text
TRAINER
ADMIN
```

A mobile `ADMIN` does not automatically become a web/platform Admin.

---

# 7. Product Context

Most mobile data is scoped to a Product.

The backend uses:

```http
X-Current-Product: <product-code>
```

Example:

```http
X-Current-Product: mobilisation
```

Do NOT send:

```text
X-Current-Tenant
product_id in body
product_id in query
```

to try to override product context unless a particular endpoint explicitly documents it.

---

# 8. Product Selection Rules

## User belongs to one Product

If the user has exactly one active Product membership, the backend may resolve it automatically.

The client can continue directly into the application.

---

## User belongs to multiple Products

The backend does NOT silently select the first one.

The mobile application should:

```text
GET /auth/product-access
        ↓
display available Products
        ↓
user selects Product
        ↓
store selected Product code
        ↓
send X-Current-Product on Product-scoped requests
```

---

# 9. GET `/api/v1/auth/product-access`

## Status

✅ **LIVE / MERGED**

## Purpose

This is the primary mobile post-authentication endpoint.

Use it immediately after Clerk authentication.

It tells the mobile application:

- whether the authenticated Clerk identity has mobile access
- which Products they belong to
- what role they have for each Product
- whether a Product can be automatically selected

This should be the main mobile authorization/bootstrap endpoint.

---

## Request

```http
GET /api/v1/auth/product-access
```

Headers:

```http
Authorization: Bearer <clerk-session-token>
Accept: application/json
```

If the user has multiple Product memberships and wants to resolve a particular Product:

```http
X-Current-Product: <product-code>
```

---

## Conceptual Response

```json
{
  "email": "amina.agent@example.com",
  "products": [
    {
      "product": {
        "id": "product-id",
        "code": "mobilisation",
        "name": "Mobilisation",
        "is_active": true
      },
      "role": "AGENT"
    }
  ],
  "current": {
    "product": {
      "id": "product-id",
      "code": "mobilisation",
      "name": "Mobilisation",
      "is_active": true
    },
    "role": "AGENT"
  }
}
```

Use the live Swagger schema for the exact response structure.

---

# 10. When to Use `/auth/product-access`

Call it:

### After login

```text
Clerk login
→ /auth/product-access
→ establish mobile session
```

### After restoring a Clerk session

When the user reopens the app:

```text
restore Clerk session
→ call /auth/product-access
→ verify backend access still exists
```

Do not rely indefinitely on locally cached product permissions.

Memberships can be deactivated server-side.

---

# 11. Mobile User Provisioning

A mobile user does NOT self-register into the backend.

The user must first have a Product membership created for their email.

Workflow:

```text
Admin provisions email
        ↓
User signs in through Clerk
        ↓
Clerk verifies identity
        ↓
Backend reads verified email
        ↓
Product membership found
        ↓
Access granted
```

If no membership exists:

```text
403
```

The mobile application should display an access/provisioning message rather than trying to create the user itself.

---

# 12. GET `/api/v1/products`

## Status

✅ **LIVE / MERGED**

## Purpose

Returns configured Products.

This endpoint is primarily an administration/configuration API.

A normal AGENT should use:

```http
GET /auth/product-access
```

to determine the Products they personally have access to.

Do NOT use `/products` as the AGENT's authorization source.

---

# 13. POST `/api/v1/products`

## Status

✅ **LIVE / MERGED**

## Purpose

Creates a Product.

This is an administrative operation.

The ordinary mobile AGENT application should generally NOT need to call this endpoint.

Example use:

```text
Backend/Admin tooling
→ create new Product
→ provision Product memberships
→ load Product reference data
```

---

# 14. GET `/api/v1/products/{product_id}/memberships`

## Status

✅ **LIVE / MERGED**

## Purpose

Lists users provisioned for a Product.

This is an administrative endpoint.

Use it when building or operating Product membership management.

Do NOT use it to determine the current AGENT's session.

Use:

```http
GET /auth/product-access
```

for that.

---

# 15. POST `/api/v1/products/{product_id}/memberships`

## Status

✅ **LIVE / MERGED**

## Purpose

Pre-authorizes an email for a Product.

This allows somebody to gain mobile access when they later authenticate through Clerk.

Conceptually:

```json
{
  "email": "agent@example.com",
  "role": "AGENT"
}
```

Exact request schema must be taken from Swagger.

Use this for:

```text
Product onboarding
Product administration
```

Do not call this automatically from the mobile login screen.

---

# 16. PATCH `/api/v1/products/{product_id}/memberships/{membership_id}`

## Status

✅ **LIVE / MERGED**

## Purpose

Updates an existing Product membership.

Depending on the current API contract this may include things such as:

- role changes
- activation
- deactivation

Use the generated Swagger contract for exact writable fields.

This is an administrative endpoint.

---

# 17. GET `/api/v1/locations`

## Status

🟡 **DEVELOPED + VALIDATED — AWAITING MERGE/DEPLOYMENT**

Do not assume it is live until the backend team confirms the branch is merged and deployed.

## Purpose

Returns canonical geographical locations available to the selected Product.

Hierarchy:

```text
District
   ↓
Subcounty
   ↓
Parish
   ↓
Village
```

Locations are canonical geography.

Product availability is handled by the backend.

---

# 18. Districts

```http
GET /api/v1/locations?level=district
```

Headers:

```http
Authorization: Bearer <token>
X-Current-Product: <product-code>
```

Use when displaying the first geographical selector.

---

# 19. Subcounties

```http
GET /api/v1/locations?level=subcounty&parent_id=<district_id>
```

Use after the user chooses a District.

---

# 20. Parishes

```http
GET /api/v1/locations?level=parish&parent_id=<subcounty_id>
```

Use after choosing a Subcounty.

---

# 21. Villages

```http
GET /api/v1/locations?level=village&parent_id=<parish_id>
```

Use after choosing a Parish.

---

# 22. Location Response

Conceptual:

```json
{
  "items": [
    {
      "id": "location-id",
      "name": "Nsangi",
      "level": "subcounty",
      "parent_id": "district-id"
    }
  ],
  "page": 1,
  "page_size": 50,
  "total": 1,
  "total_pages": 1
}
```

The mobile client should support the backend pagination envelope.

---

# 23. Location Selection Behaviour

When District changes:

clear:

```text
Subcounty
Parish
Village
```

When Subcounty changes:

clear:

```text
Parish
Village
```

When Parish changes:

clear:

```text
Village
```

Never retain a child belonging to a previous parent.

---

# 24. Location IDs

Treat IDs as opaque strings.

Do NOT infer:

- hierarchy
- Product
- permissions
- database type

from the identifier.

Use:

```text
level
parent_id
```

returned by the API.

---

# 25. GET `/api/v1/locations/{location_id}`

## Status

🟡 **DEVELOPED + VALIDATED — AWAITING MERGE/DEPLOYMENT**

## Purpose

Resolves one location and returns its ancestor chain.

Use it when the app has stored only:

```text
location_id
```

but needs to display:

```text
District
Subcounty
Parish
Village
```

Example conceptual response:

```json
{
  "id": "village-id",
  "name": "Example Village",
  "level": "village",
  "parent_id": "parish-id",
  "ancestors": [
    {
      "id": "district-id",
      "name": "Wakiso",
      "level": "district"
    },
    {
      "id": "subcounty-id",
      "name": "Nsangi",
      "level": "subcounty"
    },
    {
      "id": "parish-id",
      "name": "Example Parish",
      "level": "parish"
    }
  ]
}
```

Ancestors are District-first.

---

# 26. Product-Scoped Locations

Locations returned by the API are filtered by:

```text
X-Current-Product
```

The client must NOT expect Product A locations to automatically appear in Product B.

The same canonical location ID may legitimately be available in multiple Products.

---

# 27. GET `/api/v1/venues`

## Status

🟡 **DEVELOPED + VALIDATED — AWAITING MERGE/DEPLOYMENT**

## Purpose

Returns active venues configured for the current Product.

Venues are Product-owned reference data.

Use it when the mobile workflow needs the user to select a configured venue.

Request:

```http
GET /api/v1/venues
Authorization: Bearer <token>
X-Current-Product: mobilisation
```

The backend ensures Product A cannot see Product B's venues.

---

# 28. Venue Information

A Venue may contain information such as:

```text
id
name
location_id
```

depending on the generated contract.

A venue may optionally be associated with a Product-valid location.

Do not assume all venues have geographical location records.

---

# 29. GET `/api/v1/venues/{venue_id}`

## Status

🟡 **DEVELOPED + VALIDATED — AWAITING MERGE/DEPLOYMENT**

## Purpose

Resolves a single Venue.

Use when:

- a Participant already contains `venue_id`
- the mobile app needs the full Venue information
- displaying saved/historical Participant context

A Venue belonging to another Product should appear as not found rather than leaking cross-Product information.

---

# 30. GET `/api/v1/bootcamps`

## Status

🟡 **DEVELOPED + VALIDATED — AWAITING MERGE/DEPLOYMENT**

## Purpose

Returns Bootcamp/training cohorts configured for the selected Product.

A Bootcamp represents the scheduled training/bootcamp event.

The backend intentionally did NOT create a second redundant TrainingSchedule resource.

Conceptual fields include:

```text
id
name
starts_on
ends_on
```

Use the generated API schema for exact fields/nullability.

---

# 31. When to Use Bootcamps

Use:

```http
GET /api/v1/bootcamps
```

when:

- displaying available training/cohort choices
- resolving Participant training context
- showing upcoming Product training

Only the current Product's Bootcamps are returned.

---

# 32. GET `/api/v1/bootcamps/{bootcamp_id}`

## Status

🟡 **DEVELOPED + VALIDATED — AWAITING MERGE/DEPLOYMENT**

## Purpose

Resolves one Bootcamp.

Use it when a Participant already references:

```text
bootcamp_id
```

and the UI needs full Bootcamp details.

---

# 33. Transport

There is intentionally NO:

```text
GET /transport-options
```

API.

The business clarified that transport is not an option/reference list.

Transport is:

```text
transport_amount
```

stored on the Participant.

For example:

```json
{
  "transport_amount": 15000.00
}
```

Do not build a Transport dropdown unless the business requirement changes.

---

# 34. GET `/api/v1/participants`

## Status

🟡 **DEVELOPED + VALIDATED — AWAITING MERGE/DEPLOYMENT**

## Purpose

Returns Participants belonging to the currently selected Product.

This is the main mobile dashboard/list endpoint.

Request:

```http
GET /api/v1/participants
Authorization: Bearer <token>
X-Current-Product: mobilisation
```

---

# 35. Participant List Filters

Currently implemented:

```text
status
search
page
page_size
```

Example:

```http
GET /api/v1/participants?status=in_progress&page=1&page_size=50
```

Search is Product-scoped.

Do not expect search to return another Product's Participants.

---

# 36. Participant List Use Cases

Use this endpoint for:

```text
Mobilizer Dashboard
Participant Search
Participant List
Participant Status Filtering
Pagination
```

Do NOT download an entire Participant dataset unnecessarily.

Use backend pagination.

---

# 37. Participant Identity

Participants have two identifiers internally:

### Backend ID

Canonical internal participant identifier.

The API may expose this under:

```text
youth_id
```

because that is what the mobile contract expects.

This does NOT mean it references Product A's `youths` table.

### External ID

```text
external_id
```

comes from the uploaded Product roster and is unique within that Product.

The mobile app should treat backend IDs as opaque.

---

# 38. Participant Fields

Current participant implementation includes fields such as:

```text
id / youth_id
external_id
first_name
last_name
phone_number
alternative_phone_number
age
program_name
location_id
venue_id
bootcamp_id
transport_amount
status
contact_attempt_count
last_contacted_at
is_active
```

Use Swagger for exact response shape.

Notably:

```text
gender
date_of_birth
```

were NOT part of the agreed current participant contract and were therefore not added.

---

# 39. GET `/api/v1/participants/{participant_id}`

## Status

🟡 **DEVELOPED + VALIDATED — AWAITING MERGE/DEPLOYMENT**

## Purpose

Returns full context for one Participant.

Use when the Mobilizer opens a Participant from the list.

The response is intended to give the application enough information to begin the mobilisation/contact workflow.

---

# 40. Participant Detail Context

The detail may resolve context such as:

```text
participant identity
phone numbers
status
location
venue
bootcamp
transport amount
contact state
```

The mobile app should use the detail endpoint instead of trying to construct the whole Participant record from list data.

---

# 41. Product Isolation on Participants

A Participant belongs to exactly one Product.

The backend enforces Product isolation.

If the mobile user is operating in Product A and requests a Product B Participant ID:

the API returns Product-safe not-found behavior.

The client should not attempt cross-Product lookup.

---

# 42. Contact State

Participant records currently contain:

```text
contact_attempt_count
last_contacted_at
status
```

These fields are ready for the Calls API.

Until Calls are implemented/deployed, they may remain at their initial values.

The mobile application should not update these fields locally as if local state were authoritative.

---

# 43. Participant Import Behaviour

Participants are loaded by the backend from Product-specific roster files.

The mobile app does NOT create Participant records through a mobile `POST /participants` API.

Current backend import semantics are non-destructive upsert:

```text
new row     → create
known row   → update changed roster fields
missing row → leave unchanged
```

Call/contact progress is deliberately preserved during re-import.

---

# 44. Calls / Mobilisation API

## Status

⏳ **NOT READY**

The next planned resource is:

```http
GET /api/v1/participants/{participant_id}/calls

POST /api/v1/participants/{participant_id}/calls
```

Do NOT call these endpoints until the backend team confirms they are merged/deployed.

---

# 45. Planned Call Behaviour

The Calls endpoint will represent both:

```text
ordinary contact attempt
```

and:

```text
completed mobilisation submission
```

using the SAME resource.

Conceptually:

```text
POST participant call
```

without `answers`:

```text
contact attempt
```

with `answers`:

```text
mobilisation form submission
```

The mobile developer should NOT build separate backend integrations for:

```text
contact_attempt
mobilisation_submission
```

---

# 46. Dynamic Forms

A server-side Dynamic Forms API is intentionally deferred for the current MVP.

The mobile application can continue using its bundled mobilisation JSON/form definition.

The future Calls endpoint will accept the resulting:

```text
answers
```

object.

Therefore do NOT wait for:

```text
GET /forms
GET /form-schema
```

before building the current mobile form.

---

# 47. Generic Submission API

Also deferred for the MVP.

Do NOT currently integrate:

```http
POST /forms/{form_key}/submissions
```

The current MVP submission path will be:

```text
Bundled Flutter form
        ↓
answers
        ↓
POST /participants/{participant_id}/calls
```

once that endpoint becomes available.

---

# 48. Common Pagination Format

List APIs use:

```json
{
  "items": [],
  "page": 1,
  "page_size": 50,
  "total": 0,
  "total_pages": 0
}
```

Do NOT assume:

```text
items.length == total
```

Recommended Flutter model:

```dart
class Page<T> {
  final List<T> items;
  final int page;
  final int pageSize;
  final int total;
  final int totalPages;
}
```

---

# 49. Common Error Format

Backend errors use structured JSON.

Conceptually:

```json
{
  "code": "PRODUCT_ACCESS_DENIED",
  "message": "You do not have access to this product.",
  "details": {}
}
```

The response may include:

```text
X-Request-Id
```

Preserve it.

Recommended Flutter exception:

```dart
class ApiException implements Exception {
  final String code;
  final String message;
  final Map<String, dynamic>? details;
  final String? requestId;
}
```

---

# 50. HTTP Status Handling

## 400

Usually request/context problem.

Example:

```text
PRODUCT_CONTEXT_REQUIRED
```

The client may need the user to select a Product.

---

## 401

Authentication/session problem.

The Clerk session/token is missing or invalid.

Attempt session recovery through Clerk.

Do NOT repeatedly retry with the same invalid token.

---

## 403

Authenticated but unauthorized.

Possible causes:

```text
no Product membership
inactive membership
inactive Product
attempted access to unheld Product
```

Do not convert this to `401`.

---

## 404

Resource cannot be resolved inside the caller's current Product.

Do NOT attempt to determine whether the resource exists in another Product.

This is intentional anti-enumeration behavior.

---

## 422

Validation error.

Use:

```text
details
```

to determine which input/query parameter was invalid.

---

# 51. Recommended Flutter Architecture

Use:

```text
Screen
   ↓
Controller / State
   ↓
Repository
   ↓
ApiClient
   ↓
Backend
```

Do NOT use:

```text
Screen
   ↓
Dio.get(...)
```

directly.

Suggested repositories:

```text
AuthRepository
ProductRepository
LocationRepository
VenueRepository
BootcampRepository
ParticipantRepository
CallRepository      [when ready]
```

---

# 52. Central API Client

The API client should centrally handle:

```text
API_BASE_URL
Authorization
X-Current-Product
JSON encoding
JSON decoding
pagination
timeouts
structured errors
X-Request-Id
network exceptions
```

Product context should not have to be manually added by every screen.

---

# 53. Suggested Mobile Startup Flow

```text
Launch
  ↓
Restore Clerk session
  ↓
No session?
  ├── YES → Sign In
  └── NO
       ↓
GET /auth/product-access
       ↓
0 Products?
  → Access/provisioning screen

1 Product?
  → Select automatically / continue

Multiple Products?
  → Product selector
       ↓
Store current Product code
       ↓
Use X-Current-Product
       ↓
Participant Dashboard
```

---

# 54. Suggested Mobilizer Workflow

When the remaining APIs are deployed, expected flow is:

```text
Login
 ↓
/auth/product-access
 ↓
Select Product
 ↓
/participants
 ↓
Participant selected
 ↓
/participants/{id}
 ↓
Resolve location / venue / bootcamp as needed
 ↓
Start call
 ↓
POST /participants/{id}/calls
 ↓
Participant state updated
```

---

# 55. Swagger Usage

Development API documentation:

```text
https://e-growth.dev.educateapps.work/docs#/
```

Before integrating an endpoint marked:

```text
🟡 DEVELOPED / VALIDATED
```

check Swagger.

If it does NOT appear in Swagger yet:

it has not reached the deployed development environment.

Do not work around that by guessing the route.

Ask the backend team for deployment status.

---

# 56. Current Mobile Developer Scope

The mobile developer can safely work now on:

### Live

```text
Clerk authentication

GET /auth/product-access

Product selection

X-Current-Product handling
```

Administration APIs can also be tested where relevant.

### Prepare integration for upcoming deployment

Models/repositories can be prepared for:

```text
Locations
Venues
Bootcamps
Participants
```

but runtime calls should only be enabled when those endpoints appear in deployed Swagger.

### Wait for backend

Do not enable:

```text
Call history
Call recording
Mobilisation answer submission
```

until Calls is delivered.

---

# 57. API Readiness at a Glance

```text
Authentication / Product Access    ██████████  LIVE

Product Administration             ██████████  LIVE

Locations                          ██████████  BUILT
                                  awaiting merge/deploy

Venues                             ██████████  BUILT
                                  awaiting merge/deploy

Bootcamps                          ██████████  BUILT
                                  awaiting merge/deploy

Participants                       ██████████  BUILT
                                  awaiting merge/deploy

Calls / Mobilisation               ░░░░░░░░░░  NEXT
```

---

# 58. Important Integration Rule

The backend already serves another application.

Do not request another API simply because the Flutter model uses different names.

Use this order:

```text
Existing backend API
        ↓
Flutter DTO mapping
        ↓
Additive backend extension if genuinely required
        ↓
New endpoint only when no equivalent capability exists
```

For example:

```text
backend full_name
```

can map to:

```text
Flutter displayName
```

without changing the backend.

---

# 59. Reporting API Problems

When reporting a mobile/backend integration issue, send:

```text
Feature/Screen:
HTTP method:
Endpoint:
Product code:
Request:
HTTP status:
Response code:
Response message:
X-Request-Id:
Expected behavior:
Actual behavior:
```

Never send Clerk session tokens into Slack/Jira screenshots.

---

# 60. Backend/Mobile Coordination

If a field is missing, do not silently add client-side assumptions.

Ask:

```text
Feature:
Endpoint:
Field needed:
Why it is needed:
How the UI uses it:
```

The backend team will determine whether it belongs in:

```text
existing endpoint
DTO mapping
new endpoint
future feature
```

---

# Current Handoff Summary

## Ready to use on the deployed development environment

```http
GET   /api/v1/auth/product-access

GET   /api/v1/products
POST  /api/v1/products

GET   /api/v1/products/{product_id}/memberships
POST  /api/v1/products/{product_id}/memberships

PATCH /api/v1/products/{product_id}/memberships/{membership_id}
```

The normal Mobilizer application primarily needs:

```http
GET /api/v1/auth/product-access
```

The Product management endpoints are mainly for administration.

---

## Developed and validated, awaiting merge/deployment

```http
GET /api/v1/locations
GET /api/v1/locations/{location_id}

GET /api/v1/venues
GET /api/v1/venues/{venue_id}

GET /api/v1/bootcamps
GET /api/v1/bootcamps/{bootcamp_id}

GET /api/v1/participants
GET /api/v1/participants/{participant_id}
```

---

## Next backend APIs

```http
GET  /api/v1/participants/{participant_id}/calls

POST /api/v1/participants/{participant_id}/calls
```

---

## Development Documentation

```text
https://e-growth.dev.educateapps.work/docs#/
```

Always check this Swagger page before assuming a newly delivered endpoint is available on the development environment.
