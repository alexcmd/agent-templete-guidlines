# Authentication — Multi-Provider Auth Abstraction

> Section: Multi-Provider Auth Abstraction — ProviderDescriptor, registry, 30+ providers

Part of the [Authentication Guide](02-auth-overview-apikey.md) series.

## 14. Multi-Provider Auth Abstraction

When building an agent that supports many providers (30+), define a `ProviderDescriptor` and select providers by scanning environment variables.

### ProviderDescriptor Schema

```python
from dataclasses import dataclass

@dataclass
class ProviderDescriptor:
    id: str              # e.g. "anthropic", "openai", "ollama"
    name: str            # human display name
    env_var: str | None  # env var containing the API key (None if no key needed)
    base_url: str        # API base URL
    no_api_key_required: bool = False  # True for local providers (ollama, lm_studio)
```

### Provider Registry (Python)

```python
PROVIDERS: list[ProviderDescriptor] = [
    ProviderDescriptor("anthropic",  "Anthropic",        "ANTHROPIC_API_KEY",
                       "https://api.anthropic.com"),
    ProviderDescriptor("openai",     "OpenAI",           "OPENAI_API_KEY",
                       "https://api.openai.com/v1"),
    ProviderDescriptor("google",     "Google Gemini",    "GOOGLE_API_KEY",
                       "https://generativelanguage.googleapis.com/v1beta"),
    ProviderDescriptor("azure",      "Azure OpenAI",     "AZURE_OPENAI_API_KEY",
                       ""),  # base_url from config
    ProviderDescriptor("groq",       "Groq",             "GROQ_API_KEY",
                       "https://api.groq.com/openai/v1"),
    ProviderDescriptor("deepseek",   "DeepSeek",         "DEEPSEEK_API_KEY",
                       "https://api.deepseek.com/v1"),
    ProviderDescriptor("mistral",    "Mistral",          "MISTRAL_API_KEY",
                       "https://api.mistral.ai/v1"),
    ProviderDescriptor("openrouter", "OpenRouter",       "OPENROUTER_API_KEY",
                       "https://openrouter.ai/api/v1"),
    ProviderDescriptor("ollama",     "Ollama (local)",   None,
                       "http://localhost:11434",
                       no_api_key_required=True),
    ProviderDescriptor("lm_studio",  "LM Studio (local)", None,
                       "http://localhost:1234/v1",
                       no_api_key_required=True),
    # ... 25+ more
]
```

### Provider Selection Precedence

```python
import os

def select_provider(preferred: str | None = None) -> ProviderDescriptor | None:
    """
    Provider selection order:
      1. Explicit --provider flag or AGENT_PROVIDER env var
      2. First provider whose env_var is set in the environment
      3. First local provider (no_api_key_required=True)
      4. None — prompt user to configure
    """
    if preferred:
        for p in PROVIDERS:
            if p.id == preferred:
                return p

    # Check configured env var
    provider_id = os.environ.get("AGENT_PROVIDER")
    if provider_id:
        for p in PROVIDERS:
            if p.id == provider_id:
                return p

    # Auto-detect from env vars
    for p in PROVIDERS:
        if p.env_var and os.environ.get(p.env_var):
            return p

    # Fall back to local providers
    for p in PROVIDERS:
        if p.no_api_key_required:
            return p

    return None

def get_api_key(provider: ProviderDescriptor) -> str | None:
    if provider.no_api_key_required:
        return None
    return os.environ.get(provider.env_var or "") if provider.env_var else None
```

### Using Bearer Token Instead of API Key (OAuth Mode)

```python
def get_auth_headers(provider: ProviderDescriptor,
                     token_manager: TokenManager | None = None) -> dict:
    """Returns the correct auth headers for a provider."""
    if provider.id == "anthropic":
        if token_manager:
            # OAuth mode
            return {
                "Authorization": token_manager.auth_header(),
                "anthropic-beta": "oauth-2025-04-20",
                "anthropic-version": "2023-06-01",
            }
        else:
            return {
                "x-api-key": get_api_key(provider),
                "anthropic-version": "2023-06-01",
            }
    else:
        # OpenAI-compatible providers
        return {"Authorization": f"Bearer {get_api_key(provider)}"}
```

---

*Last updated: 2026-04-18. Model IDs and pricing reflect availability at that date. Check console.anthropic.com for the latest.*

---

*[← Cloud Providers & API Reference](02-auth-cloud-api-reference.md)*
