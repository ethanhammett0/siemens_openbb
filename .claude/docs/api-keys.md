# API Key Management

Source: https://docs.openbb.co/odp/desktop/api-keys (fetched 2026-04-10)

## Storage Location

Central file: `~/.openbb_platform/user_settings.json` → `credentials` block.

## Credential Hierarchy (for your backend)

1. **ODP user_settings.json** — primary source
2. **Environment variable** — fallback
3. **`.env` file** — last resort, with `override=False`

**Critical rule:** Never let `.env` override ODP keys.

Reference implementation: `fintech_terminal/data_providers.py`

## Configuration Methods

### Direct Management
ODP Desktop provides a GUI: hover over entries → edit, save, delete.

### Bulk Import
"Import Keys" button accepts `.json` and `.env` files.

### File Access
Three key files (opens in default text editor):
- `user_settings.json` — credentials
- `system_settings.json` — system configuration
- `.env` — environment variables

## Important Notes

- **Restart required** after key changes — both backend servers and Python interpreters
- `.env` can be incorporated into backend configs for broader application access
- Missing credentials show as "Undefined" until values are added

## Reading Keys in Python

```python
import json
from pathlib import Path

settings_path = Path.home() / ".openbb_platform" / "user_settings.json"
with open(settings_path) as f:
    settings = json.load(f)
credentials = settings.get("credentials", {})
api_key = credentials.get("polygon_api_key")
```
