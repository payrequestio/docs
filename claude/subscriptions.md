# Subscription System

> Reference file for CLAUDE.md - Subscription architecture, billing, trials, migration mode

## Dual Subscription System

**DUAL SYSTEM**: PayRequest native (primary, subscriptions table) vs Legacy Mollie (mollie_subscriptions table)
**PayRequest Subscriptions**: Status types (active, pending_mandate, trial, paused, suspended, canceled, failed), uses Mollie mandates for payments
**Mollie Customer Sync**: Bi-directional sync between customers and mollie_customers tables when enabled
**CRITICAL**: Use subscriptions table directly, NOT MollieSubscriptionService for PayRequest subscriptions

### Architecture Details
- **PayRequest Subscriptions**: New native system (primary) - stored in `subscriptions` table
- **Mollie Subscriptions**: Legacy system (backup) - stored in `mollie_subscriptions` table
- **Migration Flow**: Mollie → PayRequest with tag/metadata conversion
- **Tag System**: Supports both subscription types via separate relationships
- **Routes**: `/subscriptions` (PayRequest) vs `/subscriptions/mollie` (Mollie legacy)
- **Display**: PayRequest subscriptions show names, Mollie shows Mollie IDs
- **Backward Compatibility**: All existing Mollie data preserved with "Legacy" badges

## PayRequest Billing System (Self-Subscription)

**REPLACED STRIPE**: PayRequest now uses its own subscription system for billing plans (migrated October 2025)

### Billing Products (on payrequest.shop)
- **Product 1860**: Freelancer Plan (€5/month) - Invoicing, customers, digital products, activity log
- **Product 1861**: Business Plan (€20/month) - Everything in Freelancer + subscriptions, shop, orders

### Architecture
- **User Model**: Added `customer_id` column (foreign key to customers table)
- **Service**: `UserSubscriptionService` - Centralized subscription logic for checking plans
- **Shop**: payrequest.shop (User ID 3) - PayRequest's own shop for billing plans
- **Customers**: Users become customers of payrequest.shop when they subscribe

### Key Components
- `UserSubscriptionService` - Main service for subscription checks:
  - `hasActiveSubscription($user)` - Check if user has any active plan
  - `getCurrentPlan($user)` - Returns 'Freelancer', 'Business', or null
  - `hasBusinessPlan($user)` - Check for Business plan
  - `requiresUpgrade($user, $feature)` - Check if feature needs upgrade
- `BillingSettings` Livewire component - Settings → Billing tab
- `ShopMagicLinkController` - Seamless authentication from main app to shop

### Middleware
- `RequireSubscription` - Ensures user has active subscription (registered as `subscription.required`)
- `RequireBusinessPlan` - Restricts Business-only features (registered as `business.plan`)

### Magic Link Authentication
- **Purpose**: Seamless login from payrequest.app → payrequest.shop
- **Route**: `/magic-login` (global shop route, no storename needed)
- **Flow**: Settings Billing → "Manage Subscription in Shop" button → Magic link → Auto-login to shop
- **Session**: Stores entire customer object as `session('customer')`
- **Security**: HMAC-SHA256 signed URLs with 30-minute expiry
- **CRITICAL**: `custom_domain_initialized` flag prevents session regeneration

### HandleCustomDomain Fix
- `/magic-login` route must be in skip list (line 49)
- Prevents custom domain middleware from interfering with authentication
- Session must have `custom_domain_initialized` flag to prevent regeneration

### Important Files
- `/app/Services/UserSubscriptionService.php` - Core subscription logic
- `/app/Livewire/BillingSettings.php` - Settings billing page
- `/app/Http/Controllers/ShopMagicLinkController.php` - Magic link auth
- `/app/Http/Middleware/RequireSubscription.php` - Subscription gate
- `/app/Http/Middleware/RequireBusinessPlan.php` - Business plan gate

## Trial System

**Database**: products.has_trial/trial_days/trial_description, subscriptions.is_trial/trial_days/trial_started_at/trial_ends_at
**Auto-Application**: Subscriptions automatically apply product trials during creation
**Methods**: isInTrial(), startTrial(), shouldBill() (respects trials)
**Processing**: ProcessTrialConversions command handles trial→paid conversion
**Display**: Green badges with gift icons throughout shop and customer portal

## Subscription Billing Control (Migration Mode)

**CRITICAL**: During subscription migration work, billing can be paused to prevent charges.

### To PAUSE Billing
1. Set in `.env` file: `PAUSE_SUBSCRIPTION_BILLING=true`
2. Run: `php artisan config:clear`
3. Verify: `php artisan billing:status`

### To RESUME Billing
1. Set in `.env` file: `PAUSE_SUBSCRIPTION_BILLING=false`
2. Run: `php artisan config:clear`
3. Verify: `php artisan billing:status`

### What Gets Paused
- Daily subscription billing (03:00)
- Hourly subscription resume checks
- Failed payment retries (every 2 hours)
- Trial conversions (06:00 daily)

### What Continues Running
- Mollie transaction syncing (every 5 minutes)
- Mollie chargeback syncing (every 4 hours)
- Mollie mandate syncing (every 8 hours)
- Mollie subscription syncing (every 6 hours)
- Token refresh jobs (every 6 hours)
- Business notifications (every 5 minutes)

### Billing Status Command
```bash
php artisan billing:status
```

## Enhanced iDEAL Payment & Mandate System

**Enhancement**: MollieWebhookController processes iDEAL mandates immediately (no queue delay)
**Detection**: Primary (mandateId from payment), fallback (time-based 5-min window)
**Flow**: iDEAL payment → mandate created → subscription activated (pending_mandate → active)
**Testing**: Check subscription status/mandate_id in tinker after iDEAL payment

## Webhook Reliability & Sync System (September 2025)

**Automatic Webhook Retry**:
- `php artisan webhooks:process-pending --hours=2` (every 10 min) - Updates invoices from transactions
- `php artisan mollie:sync-payment-statuses --hours=4` (every 15 min) - Syncs from Mollie API
- `php artisan webhooks:monitor-health --hours=24` - Health monitoring

**Livewire Auto-Refresh**: InvoiceTable has `wire:poll.30s` for automatic updates
**Database**: `payrequest_subscription_id` field links transactions to PayRequest subscriptions
**Status Flow**: open → pending → paid (via webhook or sync)
