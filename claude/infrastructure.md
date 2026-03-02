# Infrastructure & Operations

> Reference file for CLAUDE.md - Queue/Horizon, maintenance, Sentry, deployment, Mollie sync

## Queue System & Laravel Horizon

### Queue Configuration
- **Queue Connection**: redis (NOT database for production)
- **Queue Names**: high, default, low, notifications, mollie, mollie-chargebacks, ponto, emails
- **Process Manager**: Laravel Horizon (sole queue processor - no standalone queue:work)
- **Redis Prefixes**: REDIS_PREFIX=payrequest_database_, HORIZON_PREFIX=payrequest_horizon:
- **CRITICAL**: All jobs must use queues defined in Horizon config - `mollie` NOT `mollie-sync`

### Horizon Supervisors (Production)
- supervisor-high-priority: handles high,default,low queues (15 max processes)
- supervisor-notifications: handles notifications queue (5 max processes)
- supervisor-mollie: handles mollie queue (8 max processes)
- supervisor-mollie-chargebacks: handles mollie-chargebacks queue (3 max processes)
- supervisor-ponto: handles ponto queue (3 max processes)
- supervisor-emails: handles emails queue (8 max processes)

### Job Configuration
- **Queue Assignment**: Use $this->onQueue('queue-name') in job constructor
- **Never use**: public $queue property (conflicts with Queueable trait)
- **Boolean Fields**: All job boolean database operations must use DB::raw('true'/'false')
- **Horizon Dashboard**: Available at /horizon with web+auth middleware

### Scheduled Jobs
- **RefreshExpiredTokens**: Every 6 hours - refresh Mollie & Ponto tokens
- **SyncMollieChargebacks**: Every 4 hours - sync chargeback data
- **SyncMollieTransactions**: Every 5 minutes - sync transaction data
- **SyncMollieMandates**: Every 8 hours - sync mandate data

### Horizon Commands
- `php artisan horizon` - Start Horizon
- `php artisan horizon:status` - Check status
- `php artisan horizon:terminate` - Stop gracefully
- `php artisan queue:failed` - View failed jobs
- `php artisan schedule:list` - View scheduled tasks

## Mollie Sync System

**Transaction Sync**: Every 5 minutes, 'mollie' queue, batch size 25
**Chargeback Sync**: Every 4 hours, 'mollie-chargebacks' queue, batch size 50
**Fixed**: PostgreSQL boolean casting with DB::raw('true'/'false')
**N+1 Fix**: Pre-load existing transactions with whereIn() (99.9% query reduction)

## Token Management

- **Background Refresh**: Tokens refreshed automatically when users visit provider page
- **Scheduled Refresh**: RefreshExpiredTokens job runs every 6 hours
- **Job-Level Refresh**: Sync jobs automatically refresh tokens before use
- **Proactive Timing**: Refresh occurs 10-30 minutes before expiry
- **Timezone Aware**: All token expiry checks use UTC for consistent comparison

## Automated Maintenance System

**Health Check**: Every 15 min, checks write permissions, root-owned files, Horizon status
**Permission Fix**: Every 30 min, fixes ownership to www-data:www-data, rebuilds caches
**Cron Jobs**: Horizon monitoring (5 min), log rotation (daily), cache cleanup (daily)
**Logs**: /var/log/payrequest-health.log, /var/log/payrequest-permissions.log
**Scripts**: /var/www/payrequest/scripts/health-check.sh, fix-permissions.sh

## Pre-Deployment Verification & Deployment Scripts

### Scripts Location
- `/var/www/payrequest/scripts/pre-deploy-check.sh` - Pre-deployment verification
- `/var/www/payrequest/scripts/deploy.sh` - Full deployment automation

### Pre-Deploy Check Script
```bash
./scripts/pre-deploy-check.sh              # Full verification
./scripts/pre-deploy-check.sh --quick      # Quick syntax check only
./scripts/pre-deploy-check.sh --dry-run    # Show what would be pulled
```

**What it checks**: New commits, PHP syntax errors, new env vars, pending migrations, dependency changes, route file changes, critical file changes, middleware changes, DB connection, Horizon status, config cachability.

### Deploy Script
```bash
./scripts/deploy.sh          # Interactive deployment with confirmation
./scripts/deploy.sh --force  # Skip confirmation prompt
```

Steps: Pre-deploy checks → Pull → Composer install → npm build → Warn migrations → Optimize → Fix permissions → Terminate Horizon

### Manual Deployment Steps
```bash
./scripts/pre-deploy-check.sh       # Step 1: Check
git pull origin main                 # Step 2: Pull
composer install --no-dev --optimize-autoloader  # Step 3: Dependencies
npm install && npm run build         # Step 4: Frontend
./optimize.sh                        # Step 5: Optimize
sudo /var/www/payrequest/scripts/fix-permissions.sh  # Step 6: Permissions
php artisan horizon:terminate        # Step 7: Restart Horizon
```

## Sentry Error Monitoring & MCP Integration

**Sentry Organization**: payrequest (`o456782`)
**Sentry Project ID**: `4508947505545216`
**Dashboard URL**: `https://payrequest.sentry.io/`

### SDK Integration

**Backend (PHP)**: `sentry/sentry-laravel@^4.15`, Config: `/config/sentry.php`, DSN via `SENTRY_LARAVEL_DSN`, Traces Sample Rate: 1.0
**Frontend (JavaScript)**: `@sentry/browser@^10.4.0`, Config: `/resources/js/sentry.js`, Integrations: Browser Tracing, Session Replay (10%), User Feedback
**Logging**: Log stack includes `sentry_logs` channel, minimum level: `warning`

### Sentry MCP Server (Claude Code Integration)

- Installed via: `claude mcp add -t http -H "Authorization: Bearer <token>" sentry https://mcp.sentry.dev/mcp`
- Scope: Local config (private per developer)
- Use to investigate production errors, check error trends, diagnose issues

### Scheduled Task Monitoring (Crons)

All critical cron jobs monitored via Sentry check-ins:
- subscriptions:process-billing (daily 05:00)
- subscriptions:resume (hourly)
- subscriptions:retry-failed (every 2h)
- subscriptions:process-trials (daily 06:00)
- sync-mollie-chargebacks (every 4h)
- sync-mollie-transactions (every 5min)
- mollie:sync-payment-statuses (every 15min)
- webhooks:process-pending (every 10min)

### Error Reporting UI

**500 Error Page** (`/resources/views/errors/500.blade.php`): Displays Sentry Event ID, modal for user error reports via `ReportErrorForm`.

### Key Files
- `/config/sentry.php` - Laravel SDK configuration
- `/resources/js/sentry.js` - Browser SDK initialization
- `/app/Exceptions/Handler.php` - Exception capture
- `/app/Console/Kernel.php` - Cron monitoring config
- `/app/Livewire/ReportErrorForm.php` - User error report form

## Route Optimization

- Use `./optimize.sh` script instead of `php artisan optimize`
- **Problem**: Closures in routes/web.php prevent route caching
- **Solution**: Custom optimize.sh caches config, events, views but skips routes
- **Command**: `./optimize.sh` (safe) vs `php artisan optimize` (fails)

## Production Environment

- **Environment**: Production (`APP_ENV=production`, `APP_DEBUG=false`)
- **Database**: `payrequest_live` (PostgreSQL via Neon)
- **Queue**: Laravel Horizon via Redis
- **Assets**: Vite production builds in `/public/build`
- **Permissions**: All application files owned by `www-data:www-data`
- **Timezone**: Europe/Amsterdam (NOT UTC) in config/app.php
