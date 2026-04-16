# ODP Credential Management

How to properly resolve API keys in backends and agents running inside
the OpenBB ODP ecosystem. This is the #1 source of auth failures.

For data pre-loading and copilot agent architecture, see
`openbb-app-builder/references/OPENBB-APP.md`.

---

## Where Keys Live

ODP stores credentials in `~/.openbb_platform/user_settings.json`:

```json
{
  "credentials": {
    "polygon_api_key": "abc123...",
    "coingecko_api_key": "xyz789...",
    "gemini_api_key": "...",
    "jina_api_key": "..."
  }
}
```

Written by the ODP API Keys UI (Workspace → Settings → API Keys).
This is the single source of truth.

## Resolution Order

Always resolve highest priority first:

1. **Environment variable** (ODP injection or shell)
2. **ODP user_settings.json** → `credentials` block
3. **Local .env file** (dev-only fallback, `override=False`)

Never let `.env` override ODP keys:
```python
load_dotenv(dotenv_path=Path(__file__).parent / ".env", override=False)
```

## Reference Implementation — Credential Loader

```python
import json, os
from pathlib import Path

_odp_credentials: dict | None = None

def _load_odp_credentials() -> dict:
    global _odp_credentials
    if _odp_credentials is not None:
        return _odp_credentials
    settings_path = Path.home() / ".openbb_platform" / "user_settings.json"
    try:
        with open(settings_path, "r", encoding="utf-8") as f:
            _odp_credentials = json.load(f).get("credentials", {})
    except (FileNotFoundError, Exception):
        _odp_credentials = {}
    return _odp_credentials
```

## Reference Implementation — Generic Resolver

For agents with multiple possible key names:

```python
def _resolve_key(odp_name: str, env_fallbacks: list[str] | None = None) -> str:
    val = _load_odp_credentials().get(odp_name, "")
    if val:
        return val
    for name in (env_fallbacks or []):
        val = os.getenv(name, "")
        if val:
            return val
    return ""

# Usage
polygon_key = _resolve_key("polygon_api_key", ["POLYGON_API_KEY", "massive_api_key"])
gemini_key = _resolve_key("gemini_api_key", ["GEMINI_API_KEY", "GOOGLE_API_KEY"])
```

For agents, inject resolved keys into env for the LLM client:
```python
if gemini_key:
    os.environ["GEMINI_API_KEY"] = gemini_key
```

## Debug Endpoint

Always include (never expose actual values):

```python
@app.get("/debug/keys")
def debug_keys():
    settings_path = Path.home() / ".openbb_platform" / "user_settings.json"
    odp_creds = {}
    if settings_path.exists():
        odp_creds = json.load(open(settings_path)).get("credentials", {})
    return {
        "odp_file_found": settings_path.exists(),
        "credential_keys": list(odp_creds.keys()),
        "polygon_resolved": bool(os.getenv("POLYGON_API_KEY") or odp_creds.get("polygon_api_key")),
    }
```

## Workspace Connection URLs

Widget/app backends and copilot agents use different URL formats
when connecting in OpenBB Workspace. Getting this wrong causes
silent connection failures.

| Backend type | URL format to enter in Workspace |
|-------------|--------------------------------|
| Widget/app backend | `http://localhost:{port}/` |
| Copilot agent | `http://127.0.0.1:{port}` |

Do NOT mix these up. Widget backends use `localhost`, agents use `127.0.0.1`.

### Port Registry

All port assignments are tracked in `PORT-REGISTRY.md` at the project root.
Before assigning a port to a new backend or agent, check the registry.
After assigning, update the registry immediately.

## Common Mistakes

- Using `.env` as primary key source instead of `user_settings.json`
- `load_dotenv()` without `override=False` (overwrites ODP-injected env vars)
- Not restarting backend after changing keys in ODP settings (loader caches on first read)
- Missing `/debug/keys` endpoint for diagnosing auth failures
- Forgetting `http://localhost:1420` in CORS origins (silent data failure)
- Using `127.0.0.1` for widget backends or `localhost` for agents (silent failure)
- Assigning a port that's already in use by another backend/agent
