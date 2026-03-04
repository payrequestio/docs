# Sentry Error Monitoring

## Configuration

- **DSN**: Configured in `.env` as `SENTRY_LARAVEL_DSN`
- **Organization**: `payrequest`
- **Project**: `payrequest-laravel`
- **Auth Token**: `.env` `SENTRY_AUTH_TOKEN` (starts with `sntryu_`)
- **Logs**: Enabled (`SENTRY_ENABLE_LOGS=true`, level: warning)
- **Traces**: Sample rate 1.0 (100%)

## CLI Usage

sentry-cli is installed globally (`/usr/local/bin/sentry-cli`).

```bash
# List all unresolved issues
sentry-cli issues list -o payrequest -p payrequest-laravel -s unresolved

# Resolve an issue by ID
sentry-cli issues resolve -o payrequest -p payrequest-laravel -i <ISSUE_ID>

# Resolve multiple issues at once
sentry-cli issues resolve -o payrequest -p payrequest-laravel -i <ID1> -i <ID2>

# Mute an issue
sentry-cli issues mute -o payrequest -p payrequest-laravel -i <ISSUE_ID>
```

Auth token must be set via `SENTRY_AUTH_TOKEN` env var or `--auth-token` flag.

## Issue Short IDs

Sentry issues have short IDs like `PAYREQUEST-LARAVEL-9V`. The numeric Issue ID (e.g., `7309529067`) is needed for CLI commands. Use `issues list` to map between them.

## Common Issue Categories

| Category | Examples | Action |
|----------|----------|--------|
| PHP exceptions | ViewException, QueryException | Fix in code, resolve |
| JS/browser errors | TypeError, FluxUI race conditions | Usually transient, monitor |
| Redis errors | Connection refused, read error | Transient infra, auto-recovers |
| Permission errors | file_put_contents failed | Run `scripts/fix-permissions.sh`, resolve |
| N+1 queries | Performance warnings | Optimize with eager loading |
| Neon DB errors | SQLSTATE[08006] connection failed | Transient, Neon cold starts |

## Triage Workflow

1. `sentry-cli issues list -o payrequest -p payrequest-laravel -s unresolved` to see open issues
2. Prioritize by level: fatal > error > warning
3. Check last_seen date - recent = actively happening
4. Fix code issues, resolve in Sentry after deploying
5. Transient infra issues (Redis, Neon) - monitor, mute if recurring
6. JS/browser errors - usually FluxUI/Livewire race conditions, low priority
