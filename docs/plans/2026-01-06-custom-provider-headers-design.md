# Custom HTTP Headers for LLM Providers

## Overview

Add support for specifying custom HTTP headers that are forwarded with each request to LLM providers. Headers are user-configurable per provider, persisted in the YAML config, and included in all outgoing API calls.

## Requirements

- **Scope:** Per-provider configuration (not per-model or global)
- **Cardinality:** Multiple headers per provider (key-value map)
- **Values:** Static only (set in config, same for every request)
- **Storage:** Inline in YAML config file
- **Providers:** HTTPX-based only (OpenAI, Azure OpenAI, RHOAI VLLM, RHELAI VLLM)

## Configuration Schema

### Config file structure (`olsconfig.yaml`)

```yaml
llm_providers:
  - name: my_openai
    type: openai
    url: "https://api.openai.com/v1"
    credentials_path: openai_api_key.txt
    extra_headers:
      X-Tenant-ID: "my-org"
      X-Request-Source: "lightspeed"
      X-Environment: "production"
    models:
      - name: gpt-4
```

### Pydantic model changes (`ols/app/models/config.py`)

Add an `extra_headers` field to `ProviderConfig`:

```python
class ProviderConfig(BaseModel):
    # ... existing fields ...
    extra_headers: dict[str, str] = {}
```

### Validation rules

- Header names must be non-empty strings
- Header values must be strings (can be empty)
- No validation against reserved headers (like `Authorization`) - users are trusted
- Empty dict by default (no headers added)

## HTTP Client Implementation

### Where headers get injected (`ols/src/llms/providers/provider.py`)

The `_construct_httpx_client()` method already creates HTTPX clients with SSL and proxy config. Extend it to accept headers:

```python
def _construct_httpx_client(
    self,
    use_custom_certificate_store: bool,
    use_async: bool,
) -> httpx.Client | httpx.AsyncClient:
    # ... existing SSL/proxy setup ...

    # Get extra headers from provider config
    extra_headers = self.provider_config.extra_headers or {}

    if use_async:
        return httpx.AsyncClient(
            verify=ssl_context,
            proxies=proxy,
            mounts=mounts,
            headers=extra_headers,
        )
    else:
        return httpx.Client(
            verify=ssl_context,
            proxies=proxy,
            mounts=mounts,
            headers=extra_headers,
        )
```

### How HTTPX handles this

- Headers passed to the client constructor become "default headers"
- They're automatically included in every request made by that client
- Per-request headers (like `Authorization` set by LangChain) merge with and override defaults

### No changes needed in individual providers

OpenAI, Azure OpenAI, RHOAI VLLM, and RHELAI VLLM all call `_construct_httpx_client()`. The headers flow through automatically via the shared base class.

## Testing Strategy

### Config validation tests (`tests/unit/app/models/test_config.py`)

```python
def test_provider_config_extra_headers_valid():
    """Test extra_headers accepts valid dict."""
    config = ProviderConfig(
        name="test",
        type="openai",
        extra_headers={"X-Tenant": "abc", "X-Env": "prod"}
    )
    assert config.extra_headers == {"X-Tenant": "abc", "X-Env": "prod"}

def test_provider_config_extra_headers_default_empty():
    """Test extra_headers defaults to empty dict."""
    config = ProviderConfig(name="test", type="openai")
    assert config.extra_headers == {}

def test_provider_config_extra_headers_rejects_non_string_key():
    """Test extra_headers rejects non-string keys."""
    with pytest.raises(ValidationError):
        ProviderConfig(
            name="test",
            type="openai",
            extra_headers={123: "value"}
        )

def test_provider_config_extra_headers_rejects_non_string_value():
    """Test extra_headers rejects non-string values."""
    with pytest.raises(ValidationError):
        ProviderConfig(
            name="test",
            type="openai",
            extra_headers={"X-Custom": 123}
        )

def test_provider_config_extra_headers_rejects_empty_key():
    """Test extra_headers rejects empty string keys."""
    with pytest.raises(ValidationError):
        ProviderConfig(
            name="test",
            type="openai",
            extra_headers={"": "value"}
        )
```

### Header injection tests (`tests/unit/llms/providers/test_provider.py`)

```python
def test_construct_httpx_client_includes_extra_headers():
    """Test that extra_headers are passed to sync HTTPX client."""
    provider_config = ProviderConfig(
        name="test",
        type="openai",
        extra_headers={"X-Custom": "value"}
    )
    provider = OpenAIProvider(...)  # with mocked config

    client = provider._construct_httpx_client(False, False)

    assert client.headers["X-Custom"] == "value"

def test_construct_httpx_async_client_includes_extra_headers():
    """Test that extra_headers are passed to async HTTPX client."""
    provider_config = ProviderConfig(
        name="test",
        type="openai",
        extra_headers={"X-Async": "header"}
    )
    provider = OpenAIProvider(...)  # with mocked config

    async_client = provider._construct_httpx_client(False, True)

    assert async_client.headers["X-Async"] == "header"
```

## Documentation

### Update example config (`examples/olsconfig.yaml`)

```yaml
llm_providers:
  - name: my_openai
    type: openai
    url: "https://api.openai.com/v1"
    credentials_path: openai_api_key.txt

    # Optional: custom HTTP headers to include in all requests to this provider.
    # This is useful for multi-tenant routing, environment tags, or internal auditing.
    # extra_headers:
    #   X-Tenant-ID: "my-org"
    #   X-Request-Source: "lightspeed"
    #   X-Environment: "production"

    models:
      - name: gpt-4
```

### Update README.md (provider configuration section)

Table row:

| Field | Description |
|-------|-------------|
| `extra_headers` | Optional mapping of custom HTTP headers to include in **all** requests made to this provider. Keys and values must be strings. Useful for tenant IDs, environment tags, or internal auditing headers. Per-request headers (such as `Authorization`) can still override these if they use the same key. |

Inline snippet (if full provider example section exists):

```yaml
llm_providers:
  - name: my_openai
    type: openai
    url: "https://api.openai.com/v1"
    credentials_path: openai_api_key.txt
    extra_headers:
      X-Tenant-ID: "my-org"
      X-Environment: "staging"
```

## Files to Modify

| File | Change |
|------|--------|
| `ols/app/models/config.py` | Add `extra_headers: dict[str, str] = {}` field to `ProviderConfig` with validation |
| `ols/src/llms/providers/provider.py` | Pass `extra_headers` to HTTPX client constructor in `_construct_httpx_client()` |
| `examples/olsconfig.yaml` | Add commented example of `extra_headers` |
| `README.md` | Add table row and inline snippet for `extra_headers` |
| `tests/unit/app/models/test_config.py` | Add validation tests (valid, default, bad key/value) |
| `tests/unit/llms/providers/test_provider.py` | Add sync and async client header injection tests |

## Implementation Order

1. Add `extra_headers` field to `ProviderConfig` with validation
2. Update `_construct_httpx_client()` to pass headers to HTTPX
3. Add unit tests for config validation
4. Add unit tests for header injection (sync + async)
5. Update `examples/olsconfig.yaml`
6. Update `README.md`

## Out of Scope

Potential future work not included in this design:

- Dynamic header values (per-request computation)
- Headers for SDK-based providers (Watsonx, BAM)
- Per-model header overrides
