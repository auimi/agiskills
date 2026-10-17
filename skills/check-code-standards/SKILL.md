---
name: check-code-standards
description: Review Python and FastAPI code against a senior-engineer coding standard focused on readability, simplicity, maintainability, performance bottlenecks, feature-based project structure, clean layering, KISS, SOLID, and YAGNI. Use when asked to review code, evaluate architecture, check maintainability, simplify implementation, critique a pull request, or verify that FastAPI changes follow feature-oriented folders and clear service/repository/router boundaries.
---

# Check Code Standards

## Review Stance

Review as a pragmatic senior engineer. Prioritize code that a mixed-experience team can understand quickly, maintain safely, and extend without hidden coupling.

Lead with concrete findings. Prefer file and line references when available. Explain why each issue matters and suggest the smallest maintainable fix.

Do not reward cleverness for its own sake. Treat simple, explicit, readable code as the default best solution unless the problem clearly needs a more advanced design.

## Core Standards

Check for:

- Readability over unnecessary complexity.
- Straightforward control flow and explicit names.
- Small focused functions with typed signatures when the language and project style support them.
- Clean separation of concerns.
- Avoidance of avoidable bottlenecks, especially broad in-memory filtering, N+1 database access, repeated expensive work, and unbounded API responses.
- SOLID, KISS, and YAGNI applied pragmatically.
- Minimal abstractions that earn their keep through repeated use, clearer ownership, or lower real complexity.
- No speculative future-proofing, generic helpers used once, or broad architecture changes without clear benefit.

## FastAPI Structure

For FastAPI projects, check that feature-oriented folderization is respected:

- Keep app-wide infrastructure in `app/core/`.
- Keep reusable cross-feature helpers, constants, exceptions, and response utilities in `app/shared/`.
- Keep domain-specific code in feature folders such as `app/esim/`.
- Keep sync jobs and provider sync scripts in `cronjob/`.

Check layering:

- Routers handle HTTP concerns, request and response schemas, status codes, dependency wiring, and translating application errors to HTTP errors.
- Services handle use cases, business workflows, transaction control, and command/query handling when already used by the project.
- Domain code handles business rules, entities, and value objects.
- Repositories handle persistence only through ORM queries.

Flag business logic in routers, database queries in routers, HTTP concerns in repositories, and direct SQLAlchemy model returns from customer-facing endpoints.

## Review Workflow

1. Inspect the surrounding code before judging. Match existing project patterns unless they are clearly causing the problem.
2. Identify correctness, maintainability, architecture, and bottleneck risks before style preferences.
3. Ask whether each abstraction, helper, dependency, or architectural change solves a real current problem.
4. Prefer smaller fixes that preserve behavior and local style.
5. Call out missing tests or verification only when they materially affect confidence.

## Output Format

When doing a code review:

1. Start with findings ordered by severity.
2. Include file and line references when possible.
3. Keep each finding specific: problem, impact, and recommended fix.
4. Add open questions or assumptions only if they affect the recommendation.
5. Add a short summary after findings, not before.

If no meaningful issues are found, say so clearly and mention any residual test or verification gaps.

## Decision Heuristics

Prefer inline code when the logic is short, local, and used once.

Prefer a helper when logic repeats 2-3 times, the extracted name makes the caller much clearer, or the block is complex enough that isolation improves understanding.

Prefer database-side filtering, ordering, aggregation, and pagination for large datasets.

Prefer explicit schemas for API responses. Do not expose secrets, raw provider data, wholesale prices, or internal implementation details in customer-facing endpoints.

Treat payment, fulfillment, webhook idempotency, provider ordering, authentication, authorization, schema changes, deployment behavior, and public API contract changes as high-risk areas. Recommend asking for confirmation before broad changes in those areas.
