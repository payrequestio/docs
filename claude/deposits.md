# Security Deposit System

> Reference file for CLAUDE.md - Authorization holds on credit cards via Mollie manual capture

**Purpose**: Authorization holds on credit cards for hotels, car rental, equipment rental, Airbnb. Uses Mollie's `captureMode=manual` — amount is held but not charged. Business can capture (charge) or release (void) after inspection.

**Key constraint**: Only credit cards support manual capture. iDEAL, SEPA, bank transfer do NOT.

## Database Schema

**deposits** table: `id`, `user_id` (FK), `customer_id` (FK), `product_id` (FK), `mollie_payment_id` (unique), `booking_reference`, `amount` (decimal 10,2), `currency`, `status` (CHECK: pending/authorized/partially_captured/captured/released/expired/failed), `authorized_at`, `capture_before`, `captured_amount`, `released_at`, `check_in_date`, `check_out_date`, `internal_notes`, `customer_notes`, `metadata` (jsonb)

**deposit_captures** table: `id`, `deposit_id` (FK cascade), `mollie_capture_id` (unique), `amount`, `currency`, `status` (CHECK: pending/succeeded/failed), `description`, `captured_at`, `metadata` (jsonb)

**products.deposit_config** (jsonb): `hold_duration_days`, `auto_release_after_days`, `allow_partial_capture`, `deposit_instructions`
**products.type**: Added 'deposit' to CHECK constraint
**transactions.deposit_id**: Nullable FK to deposits
**tags.type**: Added 'Deposit' to CHECK constraint

## Models

- `Deposit.php` — Status constants, relationships (user, customer, product, captures, tags), helpers (canCapture, canRelease, getRemainingAmount, getHoursUntilExpiry), status badge colors/labels
- `DepositCapture.php` — Simple model for individual captures
- `Product.php` — Added `isDeposit()`, `getDepositConfig()`, `deposits()` relationship, `deposit_config` in fillable/casts
- `Transaction.php` — Added `deposit_id` fillable, `deposit()` relationship

## Services

**MollieService** (4 new methods):
- `createDepositPayment()` — Like `pay()` but with `captureMode=manual` and `method=creditcard`
- `capturePayment($paymentId, $amount, $description)` — Uses Mollie Captures API
- `releasePayment($paymentId)` — Cancels/voids authorization hold
- `getPaymentCaptures($paymentId)` — Lists captures for a payment

**DepositService** (orchestration):
- `createDepositPayment()` — Creates Deposit record + calls MollieService
- `handleAuthorized()` — Updates deposit status from webhook
- `captureDeposit()` — Validates + captures + creates DepositCapture record
- `releaseDeposit()` — Releases hold + updates status
- `processExpiredDeposits()` — Marks expired deposits (scheduled hourly)

## Webhook Integration

`MollieWebhookController` handles `authorized` status for deposit payments:
- Checks `metadata.type === 'deposit'`
- Calls `DepositService::handleAuthorized()`
- Dispatches `SendDepositAuthorizedEmail`

## Dashboard Routes & Components

- `GET /deposits` → `DepositTable` (search, status tabs, pagination)
- `GET /deposits/{deposit}` → `DepositDetail` (capture/release actions, timeline, captures list)
- Sidebar: Shows "Security Deposits" with active count badge (Business plan only)

## Customer Portal

- `GET /shop/{storename}/customer/deposits` → `ShopCustomerDeposits`
- `GET /shop/{storename}/customer/deposits/{depositId}` → `ShopCustomerDepositDetail`
- Custom domain variants also registered

## Payment Page

- Deposit products filter payment methods to credit card only
- Shows deposit info banner explaining authorization holds
- `isDepositProduct` and `depositConfig` passed to view

## Email

- `SendDepositAuthorizedEmail` job (emails queue)
- `DepositAuthorizedMail` mailable
- Template: `emails/deposit-authorized.blade.php`

## Scheduled Command

- `deposits:process-expired` — Runs hourly, marks expired authorizations
- Registered in `Console/Kernel.php`

## Product Create/Edit

- 'deposit' type added to ProductCreate wizard and ProductEdit
- Deposit settings: hold duration, auto-release, partial capture toggle, instructions
- Info callout explaining how deposits work
