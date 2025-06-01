# Changelog

## 2025-06-01
- Added debug logging to `app.py` `promptflow_request` function to emit Azure OpenAI parameters: system_message, model, temperature, max_tokens.
- Disabled authentication requirement by setting `auth_enabled` to `False` in `backend/settings.py`.