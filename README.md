# E-Growth Mobile Backend API Integration Guide

## Status

**Updated after PR #35 — `feature/mobile-participants` was merged into `develop`.**

This document describes the backend APIs available to the E-Growth mobile application, how authentication and product access work, which endpoints should be used for each mobile feature, and which backend capabilities are still under development.

---

# 1. Development Environment

## Swagger Documentation

The development API documentation is available at:

```text
https://e-growth.dev.educateapps.work/docs#/
```

Swagger should always be treated as the final source of truth for what has actually reached the deployed development environment.

---

## API Host

```text
https://e-growth.dev.educateapps.work
```

## API Base URL

```text
https://e-growth.dev.educateapps.work/api/v1
```

Recommended Flutter environment configuration:

```text
API_BASE_URL=https://e-growth.dev.educateapps.work/api/v1
```

Do not hardcode the development URL inside:

- Screens
- Controllers
- Repositories
- DTOs
- Models
- Services

The API base URL should come from environment configuration.

---

# 2. Backend Architecture

The mobile application uses the same E-Growth backend infrastructure but operates as a separate Product workflow.

The backend already supports another E-Growth application, so existing APIs and business rules must be preserved.

The mobile application should consume existing APIs wherever possible rather than requiring duplicate mobile-specific endpoints.

The general rule is:

```text
Existing Backend API
        ↓
Flutter DTO / Repository Mapping
        ↓
Small additive backend extension if necessary
        ↓
New API only when no equivalent capability exists
```

The mobile client should adapt to the backend contract rather than forcing the backend to duplicate an API simply because the Flutter model uses a different field name.

---

# 3. Current API Status

The backend currently has three categories of API readiness.

## ✅ MERGED

Implementation has been merged into `develop`.

The endpoint should be considered integration-ready from the codebase perspective.

Before testing against the development URL, confirm that it appears in Swagger.

---

## 🟡 DEVELOPED / VALIDATED

Implementation is complete and tested but may still be awaiting merge/deployment.

Mobile models and repositories can be prepared, but runtime integration should only be enabled once the endpoint appears in deployed Swagger.

---

## ⏳ NOT READY

The backend capability has not yet been completed.

The mobile application should not invent or mock a permanent API contract for these features.

---

# 4. Current API Readiness

| Capability | Endpoint | Status |
|---|---|---|
| Clerk Authentication | Clerk SDK | ✅ AVAILABLE |
| Product Access | `GET /auth/product-access` | ✅ MERGED |
| Product List | `GET /products` | ✅ MERGED |
| Product Creation | `POST /products` | ✅ MERGED |
| Product Memberships | `GET /products/{id}/memberships` | ✅ MERGED |
| Add Product Member | `POST /products/{id}/memberships` | ✅ MERGED |
| Update Product Member | `PATCH /products/{id}/memberships/{membership_id}` | ✅ MERGED |
| Locations | `GET /locations` | ✅ DEVELOPED |
| Location Detail | `GET /locations/{id}` | ✅ DEVELOPED |
| Venues | `GET /venues` | ✅ DEVELOPED |
| Venue Detail | `GET /venues/{id}` | ✅ DEVELOPED |
| Bootcamps | `GET /bootcamps` | ✅ DEVELOPED |
| Bootcamp Detail | `GET /bootcamps/{id}` | ✅ DEVELOPED |
| Participants | `GET /participants` | ✅ MERGED — PR #35 |
| Participant Detail | `GET /participants/{id}` | ✅ MERGED — PR #35 |
| Call History | `GET /participants/{id}/calls` | ⏳ NOT READY |
| Record Call | `POST /participants/{id}/calls` | ⏳ NOT READY |
| Dynamic Forms API | — | ⏸ DEFERRED |
| Generic Submission API | — | ⏸ DEFERRED |

For Locations, Venues and Bootcamps, verify deployment status in Swagger before using them against the development server.

---

# 5. Authentication

The mobile application uses **Clerk** for authentication.

The mobile application should NOT implement a separate E-Growth username/password authentication system.

The flow is:

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
E-Growth Backend
```

Authenticated requests should include:

```http
Authorization: Bearer <clerk-session-token>
```

Example:

```http
GET /api/v1/auth/product-access
Authorization: Bearer <clerk-session-token>
Accept: application/json
```

---

# 6. Authentication vs Authorization

Clerk is responsible for answering:

```text
Who is this person?
```

The E-Growth backend is responsible for answering:

```text
Which Product can this person access?
```

and:

```text
What role does this person have in that Product?
```

Do NOT determine authorization from:

- Email domain
- Flutter configuration
- Clerk metadata
- Hardcoded role mappings
- Local storage

The backend Product membership is authoritative.

---

# 7. Mobile Roles

The currently supported mobile Product roles are:

```text
AGENT
ADMIN
```

For the mobilisation workflow:

```text
AGENT = Mobilizer
```

These are Product roles.

They should not be confused with existing E-Growth web roles such as:

```text
TRAINER
ADMIN
```

A Product `ADMIN` does not automatically become a platform/web Admin.

---

# 8. Mobile User Provisioning

There is no self-registration approval workflow for the mobile Product.

Instead, the user's email must already exist in the Product membership data.

The intended flow is:

```text
Admin provisions email
        ↓
User signs in through Clerk
        ↓
Clerk verifies identity
        ↓
Backend reads verified email
        ↓
Backend finds Product membership
        ↓
Access granted
```

If the authenticated user does not have a valid Product membership, the backend denies Product access.

The mobile application must NOT automatically create a Product membership during login.

---

# 9. Product Context

Mobile data is Product-scoped.

The backend uses the following header:

```http
X-Current-Product: <product-code>
```

Example:

```http
X-Current-Product: mobilisation
```

Product context must NOT be replaced with:

```text
X-Current-Tenant
product_id query parameter
product_id request body
```

unless a specific endpoint explicitly documents otherwise.

---

# 10. Product Selection

A user may belong to one or multiple Products.

The mobile application should determine this using:

```http
GET /api/v1/auth/product-access
```

## One Product

If the user has one active Product membership:

```text
Login
 ↓
Product Access
 ↓
Resolve Product
 ↓
Continue to Participant Dashboard
```

## Multiple Products

If the user has multiple Product memberships:

```text
Login
 ↓
GET /auth/product-access
 ↓
Display Product Selector
 ↓
User selects Product
 ↓
Store selected Product code
 ↓
Send X-Current-Product
```

Do not silently choose the first Product returned by the backend.

---

# 11. GET `/api/v1/auth/product-access`

## Status

✅ **MERGED**

## Purpose

This is the primary mobile bootstrap endpoint after Clerk authentication.

It tells the application:

- Whether the authenticated user has mobile Product access
- Which Products they can access
- Their role in each Product
- Which Product is currently selected/resolved

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

When resolving a particular Product:

```http
X-Current-Product: mobilisation
```

---

## When to Call It

Call this endpoint:

### Immediately after login

```text
Clerk Login
 ↓
GET /auth/product-access
 ↓
Product Resolution
 ↓
Application
```

### When restoring a session

```text
Application Opens
 ↓
Restore Clerk Session
 ↓
GET /auth/product-access
 ↓
Confirm Product Access
```

Do not rely indefinitely on locally cached permissions.

Product memberships may be changed or deactivated server-side.

---

# 12. Product Administration APIs

The backend also exposes Product administration APIs.

These are mainly intended for Product administration rather than the ordinary Mobilizer workflow.

---

# 13. GET `/api/v1/products`

## Status

✅ **MERGED**

## Purpose

Returns configured Products.

This is primarily an administration/configuration API.

A normal Mobilizer should use:

```http
GET /api/v1/auth/product-access
```

to determine the Products they personally have access to.

Do NOT use `/products` as the Mobilizer's authorization source.

---

# 14. POST `/api/v1/products`

## Status

✅ **MERGED**

## Purpose

Creates a new Product.

Example administrative workflow:

```text
Create Product
 ↓
Provision Product Members
 ↓
Load Product Locations
 ↓
Load Venues / Bootcamps
 ↓
Load Participants
```

The ordinary AGENT/Mobilizer application should not need this endpoint.

---

# 15. GET `/api/v1/products/{product_id}/memberships`

## Status

✅ **MERGED**

## Purpose

Returns users provisioned for a Product.

Use this for Product administration.

Do not use this endpoint as the current user's login/bootstrap API.

Use:

```http
GET /api/v1/auth/product-access
```

instead.

---

# 16. POST `/api/v1/products/{product_id}/memberships`

## Status

✅ **MERGED**

## Purpose

Creates a Product membership for an email.

Conceptually:

```json
{
  "email": "mobilizer@example.com",
  "role": "AGENT"
}
```

Use Swagger for the exact current request contract.

This endpoint is what allows an authenticated Clerk user to later receive Product access.

---

# 17. PATCH `/api/v1/products/{product_id}/memberships/{membership_id}`

## Status

✅ **MERGED**

## Purpose

Updates an existing Product membership.

Depending on the current Swagger contract, this may support:

- Role changes
- Activation
- Deactivation

Always use Swagger for the exact writable fields.

---

# 18. Locations

Locations represent canonical geographical reference data.

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

Locations themselves are canonical.

Product availability is controlled separately by the backend.

This prevents creating duplicate copies of locations such as Wakiso for every Product.

---

# 19. GET `/api/v1/locations`

## Status

✅ **DEVELOPED**

Confirm the endpoint appears in development Swagger before runtime testing.

## Purpose

Returns locations available to the currently selected Product.

The endpoint supports hierarchical filtering.

---

# 20. Get Districts

```http
GET /api/v1/locations?level=district
```

Headers:

```http
Authorization: Bearer <token>
X-Current-Product: mobilisation
```

Use this to populate the District selector.

---

# 21. Get Subcounties

```http
GET /api/v1/locations?level=subcounty&parent_id=<district_id>
```

Call after the user selects a District.

---

# 22. Get Parishes

```http
GET /api/v1/locations?level=parish&parent_id=<subcounty_id>
```

Call after the user selects a Subcounty.

---

# 23. Get Villages

```http
GET /api/v1/locations?level=village&parent_id=<parish_id>
```

Call after the user selects a Parish.

---

# 24. Location Response

Conceptual response:

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

The mobile application must support backend pagination.

---

# 25. Location Selection Behaviour

When District changes, clear:

```text
Subcounty
Parish
Village
```

When Subcounty changes, clear:

```text
Parish
Village
```

When Parish changes, clear:

```text
Village
```

Never retain a child selection from a previous parent.

---

# 26. Location IDs

Treat location IDs as opaque strings.

Do NOT derive:

- Level
- Product
- Hierarchy
- Permissions

from the ID.

Use the API fields:

```text
level
parent_id
```

for hierarchy.

---

# 27. GET `/api/v1/locations/{location_id}`

## Status

✅ **DEVELOPED**

## Purpose

Returns one location and its geographical ancestors.

Use this when the application has a stored:

```text
location_id
```

but needs to display the complete location context.

Conceptually:

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

Ancestors are returned District-first.

---

# 28. Product-Scoped Locations

The backend determines which locations are available to the current Product.

For example:

```text
Product A
 ├── Kampala
 └── Mukono

Product B
 ├── Kampala
 └── Wakiso
```

Kampala can be one canonical location record shared by both Products.

The client does not need to manage this relationship itself.

---

# 29. Venues

Venues are Product-owned reference data.

Unlike canonical geography, two Products can legitimately have separate venues with the same name.

Example:

```text
Product A → Community Hall
Product B → Community Hall
```

These are separate Product-owned records.

---

# 30. GET `/api/v1/venues`

## Status

✅ **DEVELOPED**

Confirm deployment in Swagger before runtime testing.

## Purpose

Returns active Venues configured for the currently selected Product.

Request:

```http
GET /api/v1/venues
Authorization: Bearer <token>
X-Current-Product: mobilisation
```

Use this endpoint when the mobile workflow needs to display/select a configured Venue.

---

# 31. Venue Data

A Venue can contain information such as:

```text
id
name
location_id
```

Use Swagger for the exact response contract.

A Venue may optionally reference a Product-valid geographical location.

Do not assume every Venue has a location.

---

# 32. GET `/api/v1/venues/{venue_id}`

## Status

✅ **DEVELOPED**

## Purpose

Returns one Venue.

Use this when:

- A Participant contains `venue_id`
- The application needs the Venue name/details
- Displaying saved Participant context

Product isolation is enforced by the backend.

---

# 33. Bootcamps

Bootcamp and Training Schedule are represented by one backend model.

A Bootcamp represents the scheduled training/cohort event.

The backend intentionally does not maintain a redundant second Training Schedule model.

---

# 34. GET `/api/v1/bootcamps`

## Status

✅ **DEVELOPED**

Confirm deployment in Swagger before runtime testing.

## Purpose

Returns Bootcamps configured for the current Product.

Conceptual fields include:

```text
id
name
starts_on
ends_on
```

Use Swagger for the exact contract.

---

# 35. Bootcamp Usage

Use:

```http
GET /api/v1/bootcamps
```

when:

- Displaying available Bootcamps
- Resolving Participant training context
- Showing scheduled training/cohort information

Only the current Product's Bootcamps are returned.

---

# 36. GET `/api/v1/bootcamps/{bootcamp_id}`

## Status

✅ **DEVELOPED**

## Purpose

Returns one Bootcamp.

Use it when a Participant already contains:

```text
bootcamp_id
```

and the application needs the full Bootcamp information.

---

# 37. Transport

Transport is NOT a reference-data API.

There is intentionally no endpoint such as:

```http
GET /transport-options
```

Transport is represented as a monetary value on the Participant:

```text
transport_amount
```

Example:

```json
{
  "transport_amount": 15000.00
}
```

Do not build a Transport dropdown unless the business requirement changes.

---

# 38. Participants

## Status

✅ **MERGED INTO DEVELOP — PR #35**

Participant support is now part of `develop`.

These APIs provide the main Participant data required by the mobile Mobilizer workflow.

---

# 39. GET `/api/v1/participants`

## Status

✅ **MERGED — PR #35**

## Purpose

Returns Participants belonging to the currently selected Product.

This is the primary endpoint for the Mobilizer Participant dashboard.

Use it for:

- Participant list
- Participant search
- Participant status filtering
- Pagination
- Selecting a Participant

---

## Request

```http
GET /api/v1/participants
```

Headers:

```http
Authorization: Bearer <clerk-session-token>
X-Current-Product: mobilisation
Accept: application/json
```

Example:

```http
GET /api/v1/participants?page=1&page_size=50
```

---

# 40. Participant Filters

Currently supported:

```text
status
search
page
page_size
```

Example search:

```http
GET /api/v1/participants?search=Amina&page=1&page_size=50
```

Example status filter:

```http
GET /api/v1/participants?status=pending&page=1&page_size=50
```

The mobile application should use server-side pagination.

Do not download the entire roster unnecessarily.

---

# 41. Participant List Response

The API follows the standard pagination format.

Conceptually:

```json
{
  "items": [
    {
      "youth_id": "participant-id",
      "external_id": "MOB-001",
      "first_name": "Amina",
      "last_name": "Nakato",
      "phone_number": "0770000000",
      "status": "pending"
    }
  ],
  "page": 1,
  "page_size": 50,
  "total": 1,
  "total_pages": 1
}
```

Use Swagger for the exact schema.

---

# 42. Participant Identity

Participants have a canonical internal UUID.

At the API boundary this may be exposed as:

```text
youth_id
```

This naming comes from the mobile contract.

It does NOT mean the Participant is linked to the existing E-Growth:

```text
youths
```

table.

The mobilisation workflow and the existing youth verification workflow remain separate datasets.

---

# 43. External Participant ID

Participants also contain:

```text
external_id
```

This comes from the Product roster.

It is unique within a Product.

Example:

```text
MOB-001
```

The mobile application should treat both internal IDs and external IDs according to their documented purpose.

Do not infer relationships from their format.

---

# 44. Participant Fields

The current Participant implementation includes fields such as:

```text
youth_id / participant identifier
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

Use Swagger for the exact:

- Field names
- Data types
- Nullable fields
- Enum values

The current agreed Participant contract did not include:

```text
gender
date_of_birth
```

so they were not introduced.

---

# 45. GET `/api/v1/participants/{participant_id}`

## Status

✅ **MERGED — PR #35**

## Purpose

Returns full information for one Participant.

Use this endpoint when the Mobilizer opens a Participant from the Participant list.

Example:

```http
GET /api/v1/participants/<participant-id>
Authorization: Bearer <token>
X-Current-Product: mobilisation
```

---

# 46. Participant Detail Usage

Recommended flow:

```text
Participant Dashboard
        ↓
GET /participants
        ↓
Select Participant
        ↓
GET /participants/{id}
        ↓
Participant Detail Screen
```

The Participant detail endpoint should be used as the primary source for the Participant screen.

Do not attempt to reconstruct the entire detail screen from the Participant list item.

---

# 47. Participant References

A Participant may contain:

```text
location_id
venue_id
bootcamp_id
```

These references can be resolved through the corresponding reference APIs.

```text
Participant
   │
   ├── location_id
   │       ↓
   │   GET /locations/{id}
   │
   ├── venue_id
   │       ↓
   │   GET /venues/{id}
   │
   └── bootcamp_id
           ↓
       GET /bootcamps/{id}
```

Do not duplicate the full reference records inside local Participant models unnecessarily.

---

# 48. Participant Product Isolation

Participants belong to Products.

The mobile application must send:

```http
X-Current-Product: <product-code>
```

The backend enforces Product isolation.

If a user operating in Product A requests a Participant belonging only to Product B, the API should behave as though that Participant cannot be resolved within Product A.

The mobile application should not attempt cross-Product discovery.

---

# 49. Participant Import

Participants are loaded from Product-specific roster files by the backend.

The mobile application does NOT create the Participant roster.

There is currently no mobile workflow requiring:

```http
POST /participants
```

Participant loading is handled administratively.

---

# 50. Participant Import Behaviour

Current backend import semantics are non-destructive upsert.

```text
New participant
    → created

Known participant with changed roster data
    → updated

Participant absent from new upload
    → retained

Existing call/contact progress
    → preserved
```

This protects mobilisation progress when a roster is uploaded again.

---

# 51. Contact State

Participants already contain:

```text
status
contact_attempt_count
last_contacted_at
```

These fields prepare the Participant model for the Calls API.

Until Calls are implemented, these values may remain at their initial state.

The mobile application must not treat locally calculated contact state as authoritative.

---

# 52. Calls / Contact Attempts

## Status

⏳ **NEXT BACKEND FEATURE**

The next backend capability required for the complete mobilisation workflow is Calls / Contact Attempts.

Planned endpoints:

```http
GET /api/v1/participants/{participant_id}/calls

POST /api/v1/participants/{participant_id}/calls
```

Do not call these endpoints until the backend confirms they have been implemented and deployed.

---

# 53. Planned Call Behaviour

A Call represents a contact attempt against a Participant.

The same resource should support both:

```text
ordinary contact attempt
```

and:

```text
completed mobilisation interaction
```

The intended distinction is based on the submitted data.

For example:

```text
POST Call
without mobilisation answers
        ↓
Contact Attempt
```

and:

```text
POST Call
with mobilisation answers
        ↓
Completed Mobilisation Interaction
```

The mobile developer should not create two separate backend integrations unless the final Calls contract explicitly requires it.

---

# 54. Call History

Once available:

```http
GET /api/v1/participants/{participant_id}/calls
```

will be used to display previous contact attempts for a Participant.

Expected use:

```text
Participant Detail
      ↓
Call History
      ↓
GET /participants/{id}/calls
```

Do not invent the final response fields before the backend contract is delivered.

---

# 55. Record Call

Once available:

```http
POST /api/v1/participants/{participant_id}/calls
```

will be the write path for Mobilizer contact activity.

The backend should own updates to:

```text
participant.status
participant.contact_attempt_count
participant.last_contacted_at
```

The mobile application should not manually synchronize those fields through separate requests.

---

# 56. Dynamic Forms

Dynamic Forms are deferred for the current MVP.

The mobile application can continue using its bundled mobilisation form definition.

There is currently no requirement to wait for endpoints such as:

```http
GET /forms
GET /forms/{id}
GET /form-schema
```

before implementing the mobile form.

---

# 57. Mobilisation Answers

The current direction is:

```text
Flutter Bundled Form
        ↓
User completes form
        ↓
Flutter creates answers object
        ↓
POST /participants/{id}/calls
        ↓
Backend stores mobilisation interaction
```

This becomes active once the Calls API is delivered.

---

# 58. Generic Submission API

A generic form submission API is also deferred.

Do NOT currently build against:

```http
POST /forms/{form_key}/submissions
```

unless a later backend contract explicitly introduces it.

For the MVP, mobilisation submission belongs to the Participant Call workflow.

---

# 59. Pagination

List APIs use the backend's standard pagination structure.

Conceptually:

```json
{
  "items": [],
  "page": 1,
  "page_size": 50,
  "total": 0,
  "total_pages": 0
}
```

Do not assume:

```text
items.length == total
```

For example:

```text
items.length = 50
total = 450
```

means the current page contains 50 of 450 records.

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

# 60. Error Handling

The backend uses structured error responses.

Conceptually:

```json
{
  "code": "PRODUCT_ACCESS_DENIED",
  "message": "You do not have access to this product.",
  "details": {}
}
```

The response may also include:

```text
X-Request-Id
```

Preserve this value for debugging/support.

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

# 61. HTTP 400

A `400` normally represents an invalid request context.

Example:

```text
PRODUCT_CONTEXT_REQUIRED
```

The mobile application may need to ask the user to select a Product.

---

# 62. HTTP 401

A `401` represents an authentication/session problem.

Examples:

- Missing Clerk token
- Invalid Clerk token
- Expired Clerk session

The application should attempt Clerk session recovery.

Do not repeatedly retry the same invalid token.

---

# 63. HTTP 403

A `403` means the user is authenticated but not authorized.

Possible causes include:

- No Product membership
- Inactive membership
- Inactive Product
- User attempting to access an unauthorized Product

Do not convert a `403` into a login failure automatically.

Authentication may still be valid.

---

# 64. HTTP 404

A `404` means the requested resource cannot be resolved in the caller's current Product context.

For Product-scoped resources, this may intentionally hide whether the resource exists in another Product.

Do not attempt cross-Product discovery after receiving a `404`.

---

# 65. HTTP 422

A `422` represents validation failure.

Inspect:

```text
code
message
details
```

and, where available:

```text
details.field_errors
```

Use those fields to present appropriate validation feedback.

---

# 66. Recommended Flutter Architecture

Use a layered architecture.

```text
Screen
   ↓
Controller / State Management
   ↓
Repository
   ↓
API Client
   ↓
Backend
```

Avoid:

```text
Screen
   ↓
Dio.get(...)
```

directly.

---

# 67. Recommended Repositories

Suggested Flutter repositories:

```text
AuthRepository
ProductRepository
LocationRepository
VenueRepository
BootcampRepository
ParticipantRepository
CallRepository
```

`CallRepository` can be implemented once the Calls contract is available.

---

# 68. Central API Client

The API client should centrally manage:

```text
API_BASE_URL
Authorization header
X-Current-Product header
JSON encoding
JSON decoding
Pagination
Timeouts
Structured API errors
X-Request-Id
Network errors
```

Screens should not manually attach Product headers.

---

# 69. Product Context Storage

After Product selection, store the current Product code in application state.

Example:

```text
currentProduct.code = mobilisation
```

The API client can then automatically send:

```http
X-Current-Product: mobilisation
```

for Product-scoped requests.

If the user switches Products:

```text
Clear Product-specific caches
 ↓
Update Current Product
 ↓
Reload Product-scoped data
```

Do not display stale Participants from the previous Product.

---

# 70. Recommended Startup Flow

```text
Application Launch
        ↓
Restore Clerk Session
        ↓
Session exists?
   ┌────┴────┐
   │         │
  NO        YES
   │         │
Sign In      ↓
        GET /auth/product-access
             ↓
        Products available?
             ↓
        ┌────┴────┐
        │         │
       NO        YES
        │         │
   Access      One Product?
   Message        ↓
              Auto-select
                  │
              Multiple?
                  ↓
            Product Selector
                  ↓
          Set X-Current-Product
                  ↓
          Participant Dashboard
```

---

# 71. Recommended Participant Flow

The mobile developer can now implement:

```text
Participant Dashboard
        ↓
GET /participants
        ↓
Search / Filter / Paginate
        ↓
Select Participant
        ↓
GET /participants/{id}
        ↓
Participant Detail
        ↓
Display:
  - Participant identity
  - Contact details
  - Location
  - Venue
  - Bootcamp
  - Transport amount
  - Current status
```

---

# 72. Complete Target Mobilizer Flow

Once Calls are available:

```text
Clerk Login
      ↓
Product Access
      ↓
Product Selection
      ↓
Participant List
      ↓
Participant Detail
      ↓
Contact Participant
      ↓
Record Call
      ↓
Complete Mobilisation Form
      ↓
Submit Answers
      ↓
Backend Updates Participant State
      ↓
Updated Participant / Call History
```

---

# 73. What the Mobile Team Can Work on Now

The mobile team can proceed with the following.

## Authentication

```text
Clerk login
Clerk session restoration
Token handling
Logout
```

## Product Access

```text
GET /auth/product-access
Product selection
X-Current-Product handling
Access-denied state
```

## Participants

```text
Participant DTO
Participant repository
Participant list
Participant search
Participant filtering
Pagination
Participant detail
Participant empty state
Participant error state
```

## Reference Data

Where confirmed in deployed Swagger:

```text
Locations
Venues
Bootcamps
```

---

# 74. What the Mobile Team Should NOT Build Yet

Do not invent permanent backend contracts for:

```text
Call creation
Call history
Mobilisation submission endpoint
Dynamic form retrieval
Generic form submission
Transport option list
Participant creation
```

These either remain under development or are intentionally not part of the MVP architecture.

---

# 75. API Usage Summary

## Authentication

```http
GET /api/v1/auth/product-access
```

Use after Clerk authentication and when restoring a session.

---

## Product Administration

```http
GET    /api/v1/products
POST   /api/v1/products

GET    /api/v1/products/{product_id}/memberships
POST   /api/v1/products/{product_id}/memberships
PATCH  /api/v1/products/{product_id}/memberships/{membership_id}
```

Primarily administrative.

---

## Locations

```http
GET /api/v1/locations
GET /api/v1/locations/{location_id}
```

Used for District → Subcounty → Parish → Village reference data.

---

## Venues

```http
GET /api/v1/venues
GET /api/v1/venues/{venue_id}
```

Used for Product-specific Venue information.

---

## Bootcamps

```http
GET /api/v1/bootcamps
GET /api/v1/bootcamps/{bootcamp_id}
```

Used for Product-specific training/Bootcamp information.

---

## Participants

```http
GET /api/v1/participants
GET /api/v1/participants/{participant_id}
```

**Merged through PR #35.**

Used for the Mobilizer Participant dashboard and Participant detail.

---

## Calls

Planned:

```http
GET  /api/v1/participants/{participant_id}/calls
POST /api/v1/participants/{participant_id}/calls
```

Not ready yet.

---

# 76. Swagger Verification

Before integrating any newly delivered API against development, check:

```text
https://e-growth.dev.educateapps.work/docs#/
```

If an endpoint appears there, the deployed development backend exposes it.

If an endpoint exists in Git but does not appear in Swagger, assume deployment has not reached the latest backend version yet.

Do not guess the endpoint or create a client-side workaround.

---

# 77. Current Mobile API Coverage

```text
AUTHENTICATION
     │
     ├── Clerk                              ✅
     │
     └── Product Access                     ✅
                │
                ▼
PRODUCT CONTEXT                              ✅
                │
                ▼
LOCATIONS                                    ✅ BUILT
                │
                ├── District
                ├── Subcounty
                ├── Parish
                └── Village
                │
                ▼
VENUES                                       ✅ BUILT
                │
                ▼
BOOTCAMPS                                    ✅ BUILT
                │
                ▼
PARTICIPANTS                                 ✅ MERGED — PR #35
                │
                ├── List
                ├── Search
                ├── Filter
                ├── Pagination
                └── Detail
                │
                ▼
CALLS / CONTACT ATTEMPTS                     ⏳ NEXT
                │
                ▼
MOBILISATION ANSWERS                         ⏳ NEXT
```

---

# 78. Important Integration Rules

### Rule 1 — Backend contract wins

Do not change backend APIs simply because Flutter uses different model names.

Map them in DTOs.

---

### Rule 2 — Product context is server-enforced

Do not perform Product authorization only on the client.

---

### Rule 3 — IDs are opaque

Do not derive business information from UUIDs.

---

### Rule 4 — Swagger is deployment truth

Git tells you what has been developed.

Swagger tells you what is currently available on the deployed development environment.

---

### Rule 5 — Do not duplicate reference data

Use:

```text
location_id
venue_id
bootcamp_id
```

and their respective APIs.

---

### Rule 6 — Do not create Participants from the mobile app

Participants originate from Product roster uploads.

---

### Rule 7 — Do not update contact counters locally

Once Calls are available, the backend owns:

```text
status
contact_attempt_count
last_contacted_at
```

---

# 79. Reporting Integration Issues

When reporting an API integration issue to the backend team, provide:

```text
Feature / Screen:
HTTP Method:
Endpoint:
Product Code:
Request Body / Query:
HTTP Status:
Response Code:
Response Message:
X-Request-Id:
Expected Behaviour:
Actual Behaviour:
```

Example:

```text
Feature: Participant Detail

Method: GET

Endpoint:
/api/v1/participants/123

Product:
mobilisation

Status:
404

Response Code:
PARTICIPANT_NOT_FOUND

Request ID:
abc123

Expected:
Participant detail should load.

Actual:
Participant is shown in the list but detail returns 404.
```

Never include Clerk session tokens in:

- Slack
- Jira
- Screenshots
- Logs shared publicly

---

# 80. Backend / Mobile Coordination

If the mobile application requires a field that does not exist, report:

```text
Feature:
Endpoint:
Field Needed:
Why It Is Needed:
Where It Is Displayed:
Example Value:
```

The backend team can then determine whether the correct solution is:

```text
Existing field mapping
        ↓
Existing endpoint
        ↓
Additive field
        ↓
New endpoint
```

Do not silently create incompatible client assumptions.

---

# 81. Current Handoff

## Ready / Merged

```http
GET   /api/v1/auth/product-access

GET   /api/v1/products
POST  /api/v1/products

GET   /api/v1/products/{product_id}/memberships
POST  /api/v1/products/{product_id}/memberships
PATCH /api/v1/products/{product_id}/memberships/{membership_id}

GET   /api/v1/participants
GET   /api/v1/participants/{participant_id}
```

Participant support was merged into `develop` through:

```text
PR #35
feature/mobile-participants
```

---

# 82. Reference APIs

Backend implementations have been completed for:

```http
GET /api/v1/locations
GET /api/v1/locations/{location_id}

GET /api/v1/venues
GET /api/v1/venues/{venue_id}

GET /api/v1/bootcamps
GET /api/v1/bootcamps/{bootcamp_id}
```

Check Swagger to confirm each has reached the current development deployment.

---

# 83. Next Backend Delivery

The next backend work should focus on:

```http
GET /api/v1/participants/{participant_id}/calls

POST /api/v1/participants/{participant_id}/calls
```

This completes the missing link between:

```text
Participant
     ↓
Mobilizer contacts Participant
     ↓
Contact attempt
     ↓
Mobilisation answers
     ↓
Participant status/progress
```

Once these endpoints are merged and deployed, the mobile team can integrate the complete core mobilisation workflow.

---

# 84. Development Documentation

Swagger:

```text
https://e-growth.dev.educateapps.work/docs#/
```

API Base URL:

```text
https://e-growth.dev.educateapps.work/api/v1
```

Authentication:

```http
Authorization: Bearer <clerk-session-token>
```

Product-scoped requests:

```http
X-Current-Product: <product-code>
```

---

# Final Mobile Developer Note

The backend is being delivered incrementally while preserving the existing E-Growth application.

Do not wait for every future API before continuing mobile development.

The current implementation is sufficient to continue with:

```text
Authentication
      ↓
Product Access
      ↓
Product Selection
      ↓
Participant Listing
      ↓
Participant Search / Filtering
      ↓
Participant Detail
      ↓
Location / Venue / Bootcamp Resolution
```

The backend team will next deliver the Calls / Contact Attempts API required to continue from:

```text
Participant Detail
```

to:

```text
Contact Participant
      ↓
Record Call
      ↓
Submit Mobilisation Answers
      ↓
Update Participant Progress
```

Always verify newly delivered endpoints against the development Swagger before enabling them in the mobile application.
