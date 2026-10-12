# Public API Automation Test Plan

## Application Overview

Automated black-box contract and lifecycle coverage for every documented unauthenticated public endpoint on https://api.restful-api.dev. Scope is GET /objects, GET /objects/{id}, POST /objects, PUT /objects/{id}, PATCH /objects/{id}, and DELETE /objects/{id}; authenticated collection, registration, and login routes are explicitly out of scope. Execute against a fresh test run without credentials. Public sample records are shared fixtures: never modify or delete them. For every mutating test, create a uniquely named object, retain its returned ID, and delete it in a finally/cleanup step; if cleanup fails, report the leaked ID. Validate HTTP response class, JSON content type, response schema, and persisted state; use exact status codes only where the API contract documents them. Do not assert the public list's total count or ordering because the documentation describes it as a limited predefined sample set. API base URL: https://api.restful-api.dev. Source documentation: https://restful-api.dev/.

## Test Scenarios

### 1. Public Objects API

**Seed:** `None (API-only tests; each test creates or uses isolated state as described)`

#### 1.1. List predefined public objects

**File:** `Automation_practicee\tests\public-api\list-objects.spec.ts`

**Steps:**
  1. From a fresh run with no authentication headers, send GET https://api.restful-api.dev/objects.
    - expect: Response is successful (2xx) and has a JSON media type.
    - expect: Response body is a non-empty JSON array; each item has a string id, string name, and a data property that is either a JSON value or null.
    - expect: Do not require a fixed array size or order.
  2. Inspect representative documented sample records when present, including IDs 1, 7, and 13.
    - expect: Any returned records conform to the common object shape; treat these IDs as documentation examples, not a requirement for total-list completeness.

#### 1.2. Filter public objects by repeated ID query parameters

**File:** `Automation_practicee\tests\public-api\filter-objects-by-id.spec.ts`

**Steps:**
  1. Send GET /objects?id=3&id=5&id=10 with no authentication.
    - expect: Response is successful and contains a JSON array only of requested IDs that exist in the public dataset.
    - expect: Returned records preserve their id, name, and data fields; do not assume result ordering unless the API guarantees it.
  2. Send GET /objects?id=3 and compare it with the corresponding record in the unfiltered list or GET /objects/3.
    - expect: The filtered result contains the matching object and its fields agree with the single-object representation.
  3. Send GET /objects?id=nonexistent-public-test-id and GET /objects?id=3&id=nonexistent-public-test-id.
    - expect: The request does not return unrelated objects; the response is a valid JSON array containing only matches (including an empty array if there are no matches).
  4. Send GET /objects?id=3&id=3 to exercise a repeated duplicate value.
    - expect: The response remains valid JSON and contains no object with an ID other than 3; record whether duplicate values produce one or repeated entries without making an undocumented duplicate-count assumption.

#### 1.3. Retrieve one public object and handle unknown IDs

**File:** `Automation_practicee\tests\public-api\get-object.spec.ts`

**Steps:**
  1. Send GET /objects/7 without authentication.
    - expect: Response is successful JSON and contains one object with id "7", a string name, and a data property.
    - expect: When the documented sample remains available, its name and data follow the documented MacBook example; avoid asserting mutable values beyond the published example.
  2. Fetch a known object from GET /objects, then GET /objects/{id} using its returned ID.
    - expect: The single-object response ID matches the requested ID and its name and data agree with the list response.
  3. Send GET /objects/{id} with a unique nonexistent ID and with malformed path values such as an empty segment or URL-encoded special characters.
    - expect: Unknown or invalid IDs do not return a different object's data; response is a documented-style not-found/client error or otherwise clearly indicates no matching object.
    - expect: Response remains parseable and does not expose server internals.

#### 1.4. Create and read an object with flexible JSON data

**File:** `Automation_practicee\tests\public-api\create-object.spec.ts`

**Steps:**
  1. Generate a unique name (for example, api-auto-<run-id>) and POST /objects with Content-Type: application/json and a body containing that name plus nested data with string, number, boolean, null, array, and nested-object values.
    - expect: Response is successful (2xx), JSON, and contains a server-generated string id, the submitted name, data equivalent to the submitted JSON, and a createdAt timestamp.
    - expect: The response does not silently coerce or drop supported JSON values.
  2. GET /objects/{returnedId}, then GET /objects?id={returnedId}; do not rely on the predefined unfiltered list to include user-created objects.
    - expect: Both retrieval routes identify the created object and preserve its name and data.
    - expect: The query-filter route returns only the matching object (or the documented no-match behavior if created IDs are intentionally outside the public query scope; report any conflict with the documentation).
  3. In a finally block, DELETE /objects/{returnedId} and verify the object is no longer retrievable.
    - expect: Cleanup is attempted even if an assertion fails; report the created ID if cleanup fails.

#### 1.5. Reject invalid object creation requests safely

**File:** `Automation_practicee\tests\public-api\create-object-validation.spec.ts`

**Steps:**
  1. Independently POST /objects with an empty body, malformed JSON, a missing name, a null name, and a non-JSON Content-Type; issue a separate request for each case.
    - expect: Each invalid request is rejected or otherwise clearly reported as invalid; it must not produce a successful create response with a new object ID.
    - expect: Error responses are parseable where a body is returned and do not expose stack traces, secrets, or internal diagnostics.
  2. POST /objects with a valid name and data omitted, then read the created object and clean it up.
    - expect: The API handles omitted optional data consistently with its contract (commonly null or omitted); name is preserved and no invalid data shape is invented.
    - expect: Always clean up any object if the server accepted the request despite the test's expectation.

#### 1.6. Replace an object completely with PUT

**File:** `Automation_practicee\tests\public-api\replace-object.spec.ts`

**Steps:**
  1. Create a uniquely named object with at least two distinct data fields, record the returned ID, and send PUT /objects/{id} with a replacement name and a replacement data object containing one changed field and one new field.
    - expect: PUT returns a successful JSON response with the same ID, the replacement name and data, and an updatedAt timestamp.
    - expect: The response reflects full replacement: fields present only in the original data are absent rather than retained.
  2. GET the object by ID after PUT.
    - expect: Persisted state matches the PUT response and replacement payload.
  3. Send PUT to a unique nonexistent ID; then clean up the created test object in a finally block.
    - expect: The unknown-ID update does not create or mutate a different object and is clearly reported as not found/invalid.
    - expect: Cleanup is performed and any leaked ID is reported.

#### 1.7. Partially update an object with PATCH

**File:** `Automation_practicee\tests\public-api\patch-object.spec.ts`

**Steps:**
  1. Create a uniquely named object containing multiple data fields; PATCH /objects/{id} with only a new name.
    - expect: PATCH returns a successful JSON response with the same ID, updated name, unchanged data, and an updatedAt timestamp.
    - expect: Fields omitted from the PATCH body remain unchanged.
  2. PATCH the same object with only a partial data update, then GET it by ID.
    - expect: The API applies only the documented fields, retains unrelated object values where partial-update semantics apply, and persists the response state.
    - expect: No unrelated fields are silently reset or removed.
  3. PATCH a unique nonexistent ID and PATCH with an invalid JSON body; clean up the created object in a finally block.
    - expect: Invalid or nonexistent targets are not reported as successful updates to an unintended record.
    - expect: Any accepted test object is deleted during cleanup; leaked IDs are reported.

#### 1.8. Delete an object and verify it is gone

**File:** `Automation_practicee\tests\public-api\delete-object.spec.ts`

**Steps:**
  1. Create a uniquely named disposable object, record its ID, then DELETE /objects/{id}.
    - expect: DELETE returns a successful JSON response with a message identifying the deleted ID, consistent with the documented response example.
  2. GET /objects/{id} after deletion and repeat DELETE /objects/{id}.
    - expect: The deleted object is no longer returned by GET; the API clearly reports not found after deletion.
    - expect: Repeated DELETE does not recreate or affect another object; its status/body may follow the API's documented idempotency behavior.
  3. Ensure test cleanup runs if any earlier step fails.
    - expect: Never delete documented shared sample IDs; report the disposable ID if cleanup cannot be confirmed.

#### 1.9. Verify public access, HTTPS, JSON headers, and CORS

**File:** `Automation_practicee\tests\public-api\transport-and-cors.spec.ts`

**Steps:**
  1. Call each of the six documented public routes over HTTPS without Authorization or cookie credentials; for write routes use a disposable object and clean it up.
    - expect: All documented routes are reachable without authentication.
    - expect: Successful JSON responses use an application/json media type and valid JSON bodies.
  2. From a browser origin different from api.restful-api.dev, make a GET request and a JSON POST preflight/request with Origin set; inspect OPTIONS preflight where the browser sends it.
    - expect: CORS permits the documented cross-origin usage; the response includes suitable Access-Control-Allow-Origin and, for preflight, allowed method/header values.
    - expect: HTTPS calls succeed without redirecting to insecure HTTP.
  3. Send an unsupported method to a public object route without changing any object.
    - expect: The API rejects unsupported methods with a clear client error and does not mutate data.
