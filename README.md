# E-Growth Mobile API — Available Endpoints Integration Guide

## Document Purpose

This document describes the backend APIs that are currently available for integration with the E-Growth mobile application.

The mobile team can begin integrating these APIs while the remaining mobile backend capabilities are being developed separately.

For now, the available backend capabilities are:

1. Clerk Bearer authentication support
2. Current authenticated user lookup
3. Location hierarchy listing
4. Individual location lookup with ancestor hierarchy

The following major mobile capabilities are **not yet part of this handoff**:

- Product/Tenant APIs
- Product memberships
- Mobile AGENT/Admin provisioning
- Participants
- Participant details
- Mobilisation calls
- Call history
- Bootcamps
- Venues
- Transport reference data
- Training schedules
- Dynamic forms APIs
- Form submissions
- Submission history

Do not build the mobile application against proposed endpoints that are not included in this document.

---

# 1. API Base

All endpoints are namespaced under:

```text
/api/v1
```

The final environment host will be supplied separately.

Example development structure:

```text
https://<environment-host>/api/v1
```

The mobile application should configure the host through environment configuration.

Do NOT hardcode:

```text
localhost
development host
staging host
production host
```

inside screens, repositories, or models.

Recommended Flutter configuration:

```text
API_BASE_URL
```

Example:

```text
API_BASE_URL=https://example.educateapps.work/api/v1
```

The exact deployed URL should come from the environment configuration supplied by the backend/DevOps team.

---

# 2. Authentication

The backend uses **Clerk Bearer authentication**.

Authenticated requests must send:

```http
Authorization: Bearer <clerk-session-token>
```

Example:

```http
GET /api/v1/auth/me
Authorization: Bearer eyJhbGciOi...
```

The mobile client should obtain the session token from Clerk.

The backend is responsible for validating the token and resolving the application user.

The mobile application should NOT implement a separate backend username/password authentication flow.

---

# 3. Authentication Flow

The intended mobile flow is:

```text
User opens mobile app
        ↓
Clerk authentication
        ↓
Clerk session established
        ↓
Obtain Clerk session token
        ↓
Authorization: Bearer <token>
        ↓
GET /api/v1/auth/me
        ↓
Resolve application identity
        ↓
Continue into application
```

Clerk handles authentication.

The E-Growth backend handles application authorization and identity resolution.

---

# 4. Common Request Headers

Authenticated requests should include:

```http
Authorization: Bearer <token>
Accept: application/json
Content-Type: application/json
```

For GET requests without a request body, `Content-Type` may be omitted.

Example:

```http
GET /api/v1/auth/me HTTP/1.1
Host: <api-host>
Authorization: Bearer <clerk-session-token>
Accept: application/json
```

---

# 5. Common Error Format

The backend uses structured errors.

The mobile client should parse backend errors rather than displaying raw HTTP/network exceptions.

Errors generally contain:

```json
{
  "code": "ERROR_CODE",
  "message": "Human-readable explanation",
  "details": {}
}
```

The backend may also return an:

```text
X-Request-Id
```

response header.

The mobile app should preserve this request ID for troubleshooting.

A suitable user-facing error could therefore be:

```text
We couldn't complete the request.

Reference: 7e342b19
```

Do not show users:

```text
SocketException
DioException
SQL error
stack trace
internal file path
```

---

# 6. Endpoint Summary

| Method | Endpoint | Purpose | Authentication |
|---|---|---|---|
| GET | `/api/v1/auth/me` | Get current authenticated application user | Clerk Bearer |
| GET | `/api/v1/locations` | List locations by hierarchy level | Clerk Bearer |
| GET | `/api/v1/locations/{location_id}` | Resolve one location and its ancestors | Clerk Bearer |

---

# 7. GET `/api/v1/auth/me`

## Purpose

Returns the currently authenticated application user.

Use this endpoint after successful Clerk authentication.

It allows the mobile application to resolve the Clerk-authenticated identity into the user information currently known by the E-Growth backend.

---

## Request

```http
GET /api/v1/auth/me
```

Headers:

```http
Authorization: Bearer <clerk-session-token>
Accept: application/json
```

No request body is required.

---

## Current Response Fields

The endpoint currently provides:

```text
id
full_name
email
role
is_active
phone
last_login_at
site
```

A conceptual response is:

```json
{
  "id": "application-user-id",
  "full_name": "Jane Doe",
  "email": "jane@example.com",
  "role": "ADMIN",
  "is_active": true,
  "phone": "+256700000000",
  "last_login_at": "2026-08-17T10:20:00Z",
  "site": null
}
```

Use the actual generated API contract as the final authority for nullable/nested field details.

---

# 8. Flutter Mapping for `/auth/me`

The mobile application should adapt the backend representation rather than asking the backend to rename fields.

Recommended mapping:

| Backend | Flutter |
|---|---|
| `id` | `id` |
| `full_name` | `displayName` |
| `email` | `email` |
| `role` | `role` |
| `is_active` | `isActive` |
| `phone` | `phone` |
| `last_login_at` | `lastLoginAt` |

For example:

```dart
class CurrentUser {
  final String id;
  final String displayName;
  final String email;
  final String? role;
  final bool isActive;
  final String? phone;
  final DateTime? lastLoginAt;
}
```

The mapping:

```text
full_name
```

to:

```text
displayName
```

belongs in the Flutter API/repository layer.

---

# 9. Important `/auth/me` Limitation

The endpoint is currently ready for **basic identity only**.

It does NOT yet provide the final mobile product context.

Do not expect:

```text
tenant
tenant_id
product
product_id
tenant memberships
product memberships
branding
enabled_features
mobile permissions
```

from this endpoint yet.

These are being handled as separate backend work.

The mobile implementation should therefore avoid hardcoding assumptions about product or tenant data into the current `/auth/me` integration.

---

# 10. `/auth/me` Expected Mobile Usage

Recommended architecture:

```text
Clerk
   ↓
AuthRepository
   ↓
ApiClient
   ↓
GET /auth/me
   ↓
CurrentUser DTO
   ↓
Application Session
```

Screens should NOT call:

```text
GET /auth/me
```

directly.

---

# 11. GET `/api/v1/locations`

## Purpose

Returns canonical administrative locations.

The hierarchy is:

```text
District
   ↓
Subcounty
   ↓
Parish
   ↓
Village
```

This endpoint is used progressively.

The client should normally request one level at a time.

---

# 12. Location Model

Each location contains:

```text
id
name
level
parent_id
```

Example:

```json
{
  "id": "8b6ac19d...",
  "name": "Nakisunga",
  "level": "subcounty",
  "parent_id": "268b0e..."
}
```

Supported levels are:

```text
district
subcounty
parish
village
```

---

# 13. Location IDs

Treat every location ID as an **opaque string**.

Do NOT derive anything from the ID.

Do NOT infer:

```text
district
subcounty
parish
village
product
tenant
permissions
```

from the ID format.

Always use:

```text
level
parent_id
```

returned by the server.

---

# 14. List Districts

## Endpoint

```http
GET /api/v1/locations?level=district
```

Example:

```http
GET /api/v1/locations?level=district&page=1&page_size=50
```

---

## Response

The backend uses its standard paginated format.

Example:

```json
{
  "items": [
    {
      "id": "district-id-1",
      "name": "Wakiso",
      "level": "district",
      "parent_id": null
    },
    {
      "id": "district-id-2",
      "name": "Mukono",
      "level": "district",
      "parent_id": null
    }
  ],
  "page": 1,
  "page_size": 50,
  "total": 2,
  "total_pages": 1
}
```

Districts have:

```text
parent_id = null
```

---

# 15. List Subcounties

After the user selects a District:

```http
GET /api/v1/locations?level=subcounty&parent_id=<district_id>
```

Example:

```http
GET /api/v1/locations?level=subcounty&parent_id=123
```

Example response:

```json
{
  "items": [
    {
      "id": "subcounty-id",
      "name": "Nakisunga",
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

---

# 16. List Parishes

After selecting a Subcounty:

```http
GET /api/v1/locations?level=parish&parent_id=<subcounty_id>
```

Example:

```http
GET /api/v1/locations?level=parish&parent_id=456
```

Response:

```json
{
  "items": [
    {
      "id": "parish-id",
      "name": "Example Parish",
      "level": "parish",
      "parent_id": "subcounty-id"
    }
  ],
  "page": 1,
  "page_size": 50,
  "total": 1,
  "total_pages": 1
}
```

---

# 17. List Villages

After selecting a Parish:

```http
GET /api/v1/locations?level=village&parent_id=<parish_id>
```

Example:

```http
GET /api/v1/locations?level=village&parent_id=789
```

Response:

```json
{
  "items": [
    {
      "id": "village-id",
      "name": "Example Village",
      "level": "village",
      "parent_id": "parish-id"
    }
  ],
  "page": 1,
  "page_size": 50,
  "total": 1,
  "total_pages": 1
}
```

---

# 18. Location Selection Logic

The mobile application should implement dependency clearing correctly.

## District Changes

If:

```text
District A
→ Subcounty A
→ Parish A
→ Village A
```

is selected and the user changes District:

clear:

```text
Subcounty
Parish
Village
```

Then fetch Subcounties belonging to the newly selected District.

---

## Subcounty Changes

When Subcounty changes:

clear:

```text
Parish
Village
```

Then fetch the new Parishes.

---

## Parish Changes

When Parish changes:

clear:

```text
Village
```

Then fetch the new Villages.

---

# 19. Location Pagination

The location API is paginated.

Response fields:

```text
items
page
page_size
total
total_pages
```

Do NOT assume:

```text
items.length == total
```

The mobile repository should support pagination.

Recommended model:

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

# 20. GET `/api/v1/locations/{location_id}`

## Purpose

Resolves one location and returns its administrative hierarchy.

Use this when the app already has a stored:

```text
location_id
```

and needs to display the corresponding names.

---

## Request

```http
GET /api/v1/locations/{location_id}
```

Example:

```http
GET /api/v1/locations/87a39...
```

Headers:

```http
Authorization: Bearer <token>
Accept: application/json
```

---

# 21. Location Detail Response

Example:

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
      "name": "Example Subcounty",
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

Ancestors are returned in hierarchical order:

```text
District
→ Subcounty
→ Parish
```

For a District:

```json
{
  "id": "district-id",
  "name": "Wakiso",
  "level": "district",
  "parent_id": null,
  "ancestors": []
}
```

---

# 22. Recommended Flutter Location Repository

The mobile application should expose a repository rather than raw URLs.

Suggested interface:

```dart
abstract class LocationRepository {
  Future<Page<Location>> listDistricts({
    int page = 1,
    int pageSize = 50,
  });

  Future<Page<Location>> listSubcounties(
    String districtId, {
    int page = 1,
    int pageSize = 50,
  });

  Future<Page<Location>> listParishes(
    String subcountyId, {
    int page = 1,
    int pageSize = 50,
  });

  Future<Page<Location>> listVillages(
    String parishId, {
    int page = 1,
    int pageSize = 50,
  });

  Future<LocationDetail> getLocation(String id);
}
```

The repository should hide URL construction from screens.

---

# 23. Recommended API Client Architecture

Use:

```text
Flutter Screen
      ↓
State / Controller
      ↓
Repository
      ↓
ApiClient
      ↓
HTTP
      ↓
E-Growth Backend
```

Do not use:

```text
Flutter Screen
      ↓
Dio.get(...)
```

directly.

This keeps networking replaceable and testable.

---

# 24. Recommended API Client Responsibilities

The API client should handle:

```text
API base URL
Authorization header
JSON encoding
JSON decoding
timeouts
HTTP statuses
structured API errors
X-Request-Id
network failures
```

Example conceptual call:

```dart
final response = await apiClient.get(
  '/locations',
  queryParameters: {
    'level': 'subcounty',
    'parent_id': districtId,
    'page': page,
    'page_size': pageSize,
  },
);
```

---

# 25. Error Handling

If the backend returns:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Invalid location hierarchy.",
  "details": {
    "field_errors": {
      "parent_id": "This parent is not valid for the requested level."
    }
  }
}
```

the mobile application should preserve:

```text
code
message
details
requestId
```

The repository/API layer can expose an application error such as:

```dart
class ApiException implements Exception {
  final String code;
  final String message;
  final Map<String, dynamic>? details;
  final String? requestId;
}
```

---

# 26. Handling 401

If the backend responds:

```text
401
```

the mobile app should treat the Clerk/application session as no longer valid.

Do not repeatedly retry authenticated requests with the same invalid token.

Use the authentication/session layer to recover.

---

# 27. Handling 403

A `403` means the identity is authenticated but is not authorized for the requested application capability.

Do not convert it to:

```text
401
```

Do not automatically sign the user out unless the final product flow specifically requires it.

---

# 28. Handling 404

For:

```http
GET /locations/{id}
```

a `404` means the location cannot be resolved.

The client should not infer whether it:

```text
never existed
was removed
belongs to another context
```

unless the backend explicitly says so.

---

# 29. Current Location Data Caveat

The location API implementation currently exists, but the authoritative business location dataset has not yet been supplied/loaded.

Therefore during initial integration:

```http
GET /api/v1/locations?level=district
```

may legitimately return:

```json
{
  "items": [],
  "page": 1,
  "page_size": 50,
  "total": 0,
  "total_pages": 0
}
```

An empty successful response is NOT necessarily an API integration failure.

The authoritative dataset will be loaded separately.

---

# 30. Important Product/Tenant Caveat

Business has since clarified that location reference lists can vary by product/tenant.

Backend work is underway to finalize that ownership model.

Therefore:

**Do not couple the Flutter location repository to a specific tenancy implementation yet.**

For example, do not permanently embed:

```text
tenantId
```

inside the `Location` model merely because tenancy is expected later.

The backend will expose the final request context once that architecture is complete.

---

# 31. Current Mobile Role Direction

The intended mobile roles are:

```text
AGENT
ADMIN
```

Where:

```text
AGENT = Mobilizer
```

However, mobile product-role APIs are still under development.

Do not use existing web:

```text
TRAINER
```

as a substitute for AGENT.

Do not assume the role returned by the current basic `/auth/me` endpoint represents the final mobile authorization context.

---

# 32. Mobile User Provisioning Direction

Business has specified that mobile users will eventually be allowed to access the product if their verified email appears in the relevant mobile user/product membership data.

The historical business term used is:

```text
call_agents
```

The backend is currently formalizing this product-membership model.

The mobile team should therefore:

- use Clerk for authentication
- use backend identity APIs for authorization
- never determine access locally from email
- never hardcode allowed-email lists
- never trust client-supplied role data

---

# 33. APIs Not Yet Available for Mobile Integration

Do NOT integrate or call the following yet.

## Products / Tenants

Not ready yet.

Future capability will cover:

```text
current product
available products
product membership
mobile role
```

---

## Participants

Not ready yet.

Future APIs will cover:

```text
participant list
participant detail
participant status
```

---

## Mobilisation Calls

Not ready yet.

Future APIs will cover:

```text
record call
call result
call history
mobilisation data
```

---

## Bootcamps / Venues / Transport / Schedules

Not ready yet.

These will be product-specific uploaded reference data.

---

## Dynamic Forms

Not ready yet.

Do not implement against proposed server forms APIs.

---

## Form Submission

Not ready yet.

---

## Submission History

Not ready yet.

---

# 34. Do Not Use Planning OpenAPI as Deployment Truth

The mobile team may receive:

```text
MVP plan
OpenAPI proposal
API contract
```

documents containing future endpoints.

Those documents describe planned capabilities.

The source of truth for what can be integrated right now is:

1. This handoff
2. Current deployed/generated backend OpenAPI
3. Backend team confirmation

If an endpoint appears in a planning document but is not listed here as available:

**Do not assume it is deployed.**

---

# 35. Recommended Initial Mobile Integration Order

The mobile developer can work in this order:

## Phase 1

HTTP/API client infrastructure.

## Phase 2

Clerk authentication.

## Phase 3

`GET /api/v1/auth/me`.

## Phase 4

Location DTOs/repository.

## Phase 5

Location hierarchy requests.

## Phase 6

Wait for the backend team's next API handoff for:

```text
product/access
participants
reference data
calls
forms
submissions
```

---

# 36. Suggested Flutter Environment Variables

At minimum:

```text
API_BASE_URL
CLERK_PUBLISHABLE_KEY
```

For example:

```bash
flutter run \
  --dart-define=API_BASE_URL=https://example.com/api/v1 \
  --dart-define=CLERK_PUBLISHABLE_KEY=pk_test_xxx
```

Never commit:

```text
secret keys
Clerk secret key
backend private credentials
```

The mobile app should only receive Clerk's publishable/client-safe configuration.

---

# 37. API Integration Checklist

## Authentication

- [ ] Clerk SDK/configuration added
- [ ] Clerk session token can be obtained
- [ ] Bearer token added to API requests
- [ ] `/auth/me` repository implemented
- [ ] `full_name → displayName` mapping implemented
- [ ] 401 handled
- [ ] 403 handled
- [ ] Mock password auth removed from production wiring

## Locations

- [ ] Location DTO/model created
- [ ] Location detail model created
- [ ] Pagination model supported
- [ ] District request implemented
- [ ] Subcounty request implemented
- [ ] Parish request implemented
- [ ] Village request implemented
- [ ] Location detail request implemented
- [ ] Ancestors parsed
- [ ] Parent changes clear child selections
- [ ] Empty location results handled safely

## API Client

- [ ] Base URL configurable
- [ ] No URL hardcoded
- [ ] Bearer token injected centrally
- [ ] Backend errors parsed centrally
- [ ] Request ID captured
- [ ] Timeout/network failures mapped
- [ ] Screens don't call HTTP directly

---

# 38. Backend Handoff Status

Current integration-ready capabilities:

| Capability | Status |
|---|---|
| Clerk Bearer authentication support | READY |
| Current basic authenticated user | READY |
| Location hierarchy API | IMPLEMENTED |
| Location detail + ancestors | IMPLEMENTED |
| Authoritative location records | PENDING DATA |
| Mobile product/tenant context | IN DEVELOPMENT |
| AGENT/Admin mobile membership | IN DEVELOPMENT |
| Participants | NOT YET AVAILABLE |
| Calls | NOT YET AVAILABLE |
| Bootcamps/venue/transport/schedule | NOT YET AVAILABLE |
| Forms | NOT YET AVAILABLE |
| Submissions | NOT YET AVAILABLE |

---

# 39. Support / Integration Questions

If the mobile implementation needs a field that is not documented here:

Do NOT assume it should be added to an existing request or response.

Send the backend team:

```text
Screen/feature:
Endpoint:
Field/capability needed:
Why it is needed:
Expected usage:
```

We will determine whether the correct solution belongs in:

```text
mobile mapping
existing API extension
new backend endpoint
```

This helps avoid duplicate or incompatible APIs.

---

# 40. Final Integration Principle

The backend already serves another application.

Therefore the integration strategy is:

```text
Reuse existing APIs where semantics match
        ↓
Adapt response shape in Flutter where possible
        ↓
Extend existing API additively when genuinely needed
        ↓
Create new API only when the capability does not already exist
```

Do not request a second endpoint simply because the Flutter model currently uses a different field name.

The mobile application should adapt to stable backend conventions where the business semantics are equivalent.

---

# Mobile Team Starting Scope

For now, start with:

```text
Clerk
+
GET /api/v1/auth/me
+
GET /api/v1/locations
+
GET /api/v1/locations/{location_id}
```

The backend team will provide subsequent API handoffs as the remaining product, participant, reference-data, call and submission APIs become available.
