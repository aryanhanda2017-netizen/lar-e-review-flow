# Smoke Tests — LAR-E

63 tests. All must pass before any backend push to Railway.

## Run (from `lare-backend-documents/`)

```bash
cd "Code — Backend/lare-backend-documents"
python3 -m pytest smoke_test.py -v --tb=short
# Single test:
python3 -m pytest smoke_test.py::test_name -v
```

Do not use `python` (wrong version) or `pytest` without `python3 -m`.

## Key Tests

| Test | Guards |
|------|--------|
| `test_jwt_secret_validator_rejects_default` | JWT secret not the insecure default |
| `test_webhook_idempotency_check_present` | Webhook dedup via `bsp_message_id` |
| `test_auth_rate_limit_headers_present` | slowapi on `/login` + `/signup` |
| `test_reminder_1h_window_bounds` | Reminder fires in 45–75 min window |
| `test_tier_b_threshold_at_15` | Tier B threshold = 15 (not 25) |
| `test_mock_mode_warning_in_main` | Startup warns when mock mode active |

## Common Failures

- **Import error:** Wrong directory. `cd lare-backend-documents/` first.
- **`ModuleNotFoundError: pydantic_settings`:** Run `pip3 install -r requirements.txt --break-system-packages`
- **Route change breaks test:** Check trailing slash convention (see `backend.md`).
- **All passing locally, Railway failing:** Run `alembic upgrade head` — missing migration.

## Pre-Push Checklist

```bash
python3 -m pytest smoke_test.py -v --tb=short
alembic check   # should say "up to date"
git add -p && git commit -m "..." && git push
```
