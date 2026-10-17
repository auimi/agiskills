---
name: external-api-integration
description: Design, validate, and implement safe third-party API integrations. Use when you need to integrate an external REST API, vendor SDK, webhook, auth flow, API client, provider service, or Python HTTP implementation using requests or httpx; when adding or reviewing API error handling, retries, pagination, rate limits, schemas, or secure secret handling; or when current official API documentation must be checked before coding.
---

# External API Integration

## Core Rule

Never add or implement an external API integration without first confirming the integration plan with the user, unless the user has explicitly approved implementation in the current request.

Before writing integration code:

1. Find and review official provider documentation when web access is available.
2. Inspect the existing project architecture and integration patterns.
3. Summarize the proposed integration plan.
4. Ask the user to confirm before editing code.

If documentation is missing or unclear, ask the user for the official API documentation URL, sample request, and sample response.

## Discovery Checklist

Identify these details before implementation:

- Provider name and API purpose
- Official documentation URL and API version
- SDK availability, stability, and maintenance status
- Authentication method
- Required endpoints or SDK methods
- Request payloads, query parameters, and headers
- Expected response payloads and error response format
- Pagination, filtering, ordering, and rate limits
- Timeout, retry, idempotency, and concurrency requirements
- Webhook behavior and signature verification, if applicable
- Required environment variables and secret storage
- Security, logging, privacy, and compliance concerns
- Existing router, service, repository, and client boundaries

## Confirmation Format

Use this format before implementation:

```text
I found the API documentation and reviewed the required integration details.

Proposed integration:
- Provider:
- Endpoint(s) or SDK method(s):
- Authentication:
- Request payload:
- Expected response:
- Error handling:
- Retry/rate-limit strategy:
- Webhook handling:
- Files/modules to change:
- Environment variables required:

Please confirm before I add the integration.
```

Proceed only after the user confirms, unless the current request already contains explicit approval to implement.

## Implementation Workflow

Prefer official SDKs when they are current, well-maintained, compatible with the project, and reduce security or protocol risk. Prefer direct HTTP clients when the SDK is stale, too broad, hard to test, incompatible with the runtime, or only wraps simple REST endpoints.

Keep HTTP or SDK logic isolated in a dedicated client module. Do not call external providers directly from routers, controllers, commands, or repositories.

Use this layering unless the codebase already has a stronger local pattern:

```text
Route / Controller
  -> Service / Use case
    -> Integration client
      -> External API or SDK
```

For Python HTTP client implementation patterns, read `references/http-clients.md` when the task involves `requests`, `httpx`, retries, typed responses, or SDK-vs-HTTP selection.

## Client Standards

Create or update a dedicated integration client that:

- Uses typed request and response schemas where the project supports them
- Reads secrets from environment variables or a secret manager
- Defines method, URL, headers, query params, body, timeout, and expected status codes
- Validates response shape against official docs, user samples, sandbox responses, or mocks matching documented schema
- Converts provider errors into clean internal exceptions
- Handles timeouts, network errors, JSON parsing errors, and schema validation errors
- Keeps provider-specific details inside the integration boundary
- Avoids leaking raw provider responses, secrets, or implementation details into user-facing APIs
- Is easy to mock in focused tests

## Retry and Idempotency Rules

Retry only when safe:

- Network timeouts
- Rate limits, respecting `Retry-After`
- Temporary 5xx provider errors

Do not blindly retry non-idempotent operations such as payment creation, order placement, fulfillment, user creation, or message sending. Use idempotency keys when the provider supports them.

## Webhook Rules

For webhooks, validate:

- Signature verification
- Timestamp tolerance
- Replay prevention
- Event type allowlisting
- Idempotency and duplicate event handling
- Provider retry behavior
- Safe logging without secrets or sensitive payloads

Never trust webhook payloads without verification.

## Testing Expectations

Add focused tests with mocked HTTP or SDK responses unless the user explicitly requests live integration tests.

Cover the relevant cases:

- Successful request and response parsing
- Missing required response fields
- Invalid response schema
- 400, 401, 403, 404, 409, 422, 429, and 500+ errors
- Timeout and network error
- Retry and rate-limit behavior
- Pagination behavior
- Webhook signature validation and duplicate handling, if applicable

Before finalizing, run the smallest useful checks for the touched project: formatter, import or compile smoke check, focused tests, and any migration or configuration checks required by the change.
