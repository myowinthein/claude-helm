# Step 3 — Generate Postman Collection

Generate a Postman collection for this project if it exposes an API.

Read the prior step reports from `.archive/` — `step1-explore.md` and `step2-restore-and-freeze.md` — and use them as context. If a required report is missing, stop and report that the prior step must complete first.

Goal: Create reusable Postman files that future me can use to understand, test, troubleshoot, and demonstrate the project's API years later.

---

## Applicability

Generate a collection if the project exposes an API it owns — REST, GraphQL, or similar.

Skip if:
- The project has no API surface (pure frontend, static site, CLI tool, library, etc.)
- The only APIs present belong to third-party services (Stripe, SendGrid, etc.)

If skipping: explain why, output the report, and stop.

If an existing Postman collection is found (from Step 1): use it as the starting point — update, verify, and expand rather than generating from scratch.

---

## Rules

- Discover endpoints from source code, route definitions, or OpenAPI/Swagger specs. Do not speculate.
- Avoid deprecated, disabled, or commented-out endpoints.
- Use demo accounts, local tokens, or placeholder values only. No real credentials, secrets, or production URLs.
- If Step 2's report states "App running: yes": use it to verify requests.
- If Step 2's report states "App running: no" (or the field is absent from an older report): fall back to source-code discovery only and mark all requests as unverified.
- Do not commit or push.

## Verification

Verify representative requests where safe — GET requests, health endpoints, authentication flows, and local demo workflows. Verify at minimum one GET request per functional area and the primary authentication flow. If fewer than half of all endpoints have been verified, note this explicitly in Known Limitations.

Do not verify endpoints that may delete data, send emails or SMS, trigger webhooks, process payments, or perform bulk actions without explicit approval. Mark those as unverified with a note explaining why.

---

## Output Files

Use the project name from Step 1 for the filenames:

- `recovery/postman/{project-name}.postman_collection.json`
- `recovery/postman/{project-name}.postman_environment.json`

Organize requests by functional area (Authentication, Users, Admin, Products, Orders, etc.). Use collection variables, environment variables, request descriptions, and authentication notes. Use localhost URLs and placeholder values. Avoid duplicate endpoints.

Use **Postman Collection Format v2.1** (`schema: "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"`). This format is supported by Postman, Bruno, and most modern API clients. Do not use v2.0 — it is not importable by Bruno without conversion.

---

## Tasks

1. Determine whether a Postman collection is applicable for this project
2. Check for existing Postman files from Step 1 — use as starting point if found
3. Discover all API endpoints from source code, routes, or OpenAPI specs
4. Determine authentication method and required environment variables
5. Generate or update the collection and environment files
6. Organize requests by functional area
7. Verify representative requests where safe
8. Mark unverified requests with a reason

---

Output the report in this format:

## Applicability
Whether a collection was generated or skipped and why.

## Existing Collection
Whether an existing Postman collection was found and how it was used.

## Endpoints Discovered
Total endpoints found. How they were discovered (routes, OpenAPI spec, source code). Any endpoints excluded and why.

## Authentication
Auth method used. How it is configured in the collection.

## Files Created or Updated
File paths and what changed if updating an existing collection.

## Verified Requests
Requests that were tested against the running app and their results.

## Unverified Requests
Requests that could not be safely verified. Reason for each.

## Known Limitations
Endpoints that could not be covered, external dependencies, or gaps in coverage.

---

Do not commit or push. Output the report and stop. Wait for explicit approval before proceeding to Step 4.
