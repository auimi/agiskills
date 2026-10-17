# Python HTTP Client Patterns

Use this reference when implementing an external API client with `requests`, `httpx`, or a vendor SDK.

## Selection Guide

Prefer a vendor SDK when:

- It is official, actively maintained, and compatible with the project runtime.
- It handles required signing, OAuth, pagination, uploads, or webhook verification correctly.
- It has stable typed models or clear exceptions that simplify the integration.

Prefer direct HTTP with `requests` or `httpx` when:

- The SDK is stale, unofficial, too broad, or hard to test.
- The integration only needs a small number of REST endpoints.
- The project already uses a local HTTP client pattern.
- The SDK hides retry, timeout, idempotency, or error behavior that must be explicit.

Use `requests` for simple synchronous codebases that already depend on it. Use `httpx` when async support, a shared client, finer timeout control, or test transport support is useful.

## Client Shape

Keep provider details behind a small class:

```python
from __future__ import annotations

from dataclasses import dataclass
from decimal import Decimal
from typing import Any

import httpx


class ProviderError(Exception):
    pass


class ProviderAuthError(ProviderError):
    pass


class ProviderRateLimitError(ProviderError):
    pass


@dataclass(frozen=True)
class ProviderPackage:
    provider_id: str
    name: str
    price: Decimal


class ProviderClient:
    def __init__(self, base_url: str, api_key: str, timeout: float = 10.0) -> None:
        self._base_url = base_url.rstrip("/")
        self._api_key = api_key
        self._timeout = timeout

    async def list_packages(self, country_code: str) -> list[ProviderPackage]:
        async with httpx.AsyncClient(timeout=self._timeout) as client:
            response = await client.get(
                f"{self._base_url}/packages",
                headers=self._headers(),
                params={"country": country_code},
            )

        data = self._parse_response(response)
        items = data.get("packages")
        if not isinstance(items, list):
            raise ProviderError("Provider response missing packages list")

        return [self._parse_package(item) for item in items]

    def _headers(self) -> dict[str, str]:
        return {
            "Authorization": f"Bearer {self._api_key}",
            "Accept": "application/json",
        }

    def _parse_response(self, response: httpx.Response) -> dict[str, Any]:
        if response.status_code == 401:
            raise ProviderAuthError("Provider authentication failed")
        if response.status_code == 429:
            raise ProviderRateLimitError("Provider rate limit exceeded")
        if response.status_code >= 400:
            raise ProviderError(f"Provider request failed with {response.status_code}")

        try:
            data = response.json()
        except ValueError as exc:
            raise ProviderError("Provider returned invalid JSON") from exc

        if not isinstance(data, dict):
            raise ProviderError("Provider returned unexpected response type")
        return data

    def _parse_package(self, item: Any) -> ProviderPackage:
        if not isinstance(item, dict):
            raise ProviderError("Provider returned invalid package item")

        provider_id = item.get("id")
        name = item.get("name")
        price = item.get("price")
        if not isinstance(provider_id, str) or not isinstance(name, str):
            raise ProviderError("Provider package missing required fields")

        return ProviderPackage(
            provider_id=provider_id,
            name=name,
            price=Decimal(str(price)),
        )
```

Adapt the example to the local schema library. In Pydantic codebases, prefer explicit Pydantic models for response validation instead of dataclasses plus manual checks.

## Requests Pattern

Use a `requests.Session` when the integration makes repeated calls:

```python
from __future__ import annotations

from typing import Any

import requests


class ProviderClient:
    def __init__(self, base_url: str, api_key: str, timeout: float = 10.0) -> None:
        self._base_url = base_url.rstrip("/")
        self._timeout = timeout
        self._session = requests.Session()
        self._session.headers.update(
            {
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            }
        )

    def get_resource(self, resource_id: str) -> dict[str, Any]:
        response = self._session.get(
            f"{self._base_url}/resources/{resource_id}",
            timeout=self._timeout,
        )
        return self._parse_response(response)
```

Always pass an explicit timeout. Avoid module-level sessions unless the project already uses that pattern and lifecycle is clear.

## Retry Pattern

Keep retries narrow and explicit. Retry only safe operations, or operations protected by idempotency keys.

```python
import time
from collections.abc import Callable
from typing import TypeVar

T = TypeVar("T")


def retry_transient(operation: Callable[[], T], attempts: int = 3) -> T:
    last_error: Exception | None = None
    for index in range(attempts):
        try:
            return operation()
        except ProviderRateLimitError as exc:
            last_error = exc
            time.sleep(min(2**index, 8))
        except ProviderError as exc:
            last_error = exc
            if index == attempts - 1:
                break
            time.sleep(min(2**index, 8))

    if last_error is None:
        raise ProviderError("Provider operation failed")
    raise last_error
```

Prefer a well-maintained retry library only when the project already uses one or retry behavior is complex enough to justify the dependency.

## SDK Wrapper Pattern

Even with an SDK, wrap provider calls:

```python
class ProviderSdkClient:
    def __init__(self, sdk_client: object) -> None:
        self._sdk_client = sdk_client

    def create_order(self, payload: CreateOrderRequest) -> CreateOrderResponse:
        try:
            sdk_response = self._sdk_client.orders.create(payload.model_dump())
        except ProviderSdkAuthException as exc:
            raise ProviderAuthError("Provider authentication failed") from exc
        except ProviderSdkException as exc:
            raise ProviderError("Provider order creation failed") from exc

        return CreateOrderResponse.model_validate(sdk_response)
```

Do not let SDK exception types, SDK model objects, or raw SDK responses spread through the application.

## Test Pattern

Use mocked HTTP or SDK responses for unit tests:

- Mock success responses with all required fields.
- Mock missing fields and invalid types.
- Mock non-2xx responses by status code.
- Mock network timeout and JSON parse errors.
- Assert internal exceptions, not provider exception types.
- Assert idempotency keys and headers for sensitive create operations.

For `httpx`, prefer `MockTransport` when available in the project:

```python
import httpx


def test_client_parses_success_response() -> None:
    def handler(request: httpx.Request) -> httpx.Response:
        assert request.headers["authorization"] == "Bearer test-key"
        return httpx.Response(200, json={"packages": [{"id": "pkg_1", "name": "Test", "price": "12.50"}]})

    transport = httpx.MockTransport(handler)
    client = httpx.Client(transport=transport)
```

Inject the test client or transport when the production client design supports it. If not, keep mocks focused at the smallest provider boundary.
