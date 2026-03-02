# Payments & Integrations

> Reference file for CLAUDE.md - Payment systems, smart links, bank transfers, multi-PSP, MCP

## Payment Providers

Businesses connect their own payment providers (Mollie, Stripe, etc.).

Integrations:
- Mollie API
- Stripe API
- Ponto API (Bank sync & Bank Payments)
- Postmark API (Transactional Email)

Payment options for customers:
- Card payments
- Pay by bank (Open Banking via Ponto)
- Manual payments

## Mollie Application Fees

- **Database-driven configuration**: `users.application_fee_percentage` and `users.application_fee_description`
- **Example**: amsterdam@watermansport.nl (user_id = 2) has 2% application fee configured
- Fee automatically deducted from payment and transferred to PayRequest account
- **Service**: `ApplicationFeeService::addToPaymentData()` handles all fee logic
- **Files using fees**: MollieService, MollieSubscriptionService, ShopCheckout, ShopCustomerInvoiceDetail, ShopCustomerMandates
- **To configure**: Set `application_fee_percentage` (e.g., 2.00 for 2%) and `application_fee_description` on user record
- Currency: Always EUR regardless of payment currency

## Bank Transfer Payment Method (via Ponto)

- **Purpose**: Accept bank transfers with automatic payment matching - zero PayRequest fees
- **How it works**: Customer gets IBAN + unique reference (PAY-XXXXX), transfers exact amount, Ponto syncs transactions, PayRequest auto-matches and marks invoice paid
- **Database Tables**:
  - `pending_bank_transfers`: Tracks pending transfers awaiting matching
  - `ponto_items.bank_transfer_enabled`: Toggle per business
  - `ponto_items.bank_transfer_instructions`: Custom instructions for customers
- **Model**: `PendingBankTransfer` with `generateReference()` for unique PAY-XXXXX codes
- **Service**: `BankTransferService` handles creation, matching, and processing
- **Matching Logic**: Scans Ponto transactions for PAY-XXXXX references, verifies exact amount match
- **Auto-Match Trigger**: `SyncPontoTransactions` job calls `BankTransferService::matchTransactions()` after sync
- **Available On**: Payment pages, invoice payments, smart links (NOT subscriptions/recurring)
- **Settings UI**: Settings → Ponto → Bank Transfer Payment Method toggle
- **Customer UI**: Shows IBAN, account name, reference with copy buttons
- **Status Flow**: `awaiting` → `matched` (when payment found) → Invoice marked `paid`
- **Documentation**: `/docs/payment-processing/bank-transfer.mdx`

## Payment Page Dynamic Links System

**URL Formats**:
- Path: `/{handle}/{amount|product}/{method}/{name}/{email}/{notes}`
- Query (recommended): `?product=1814&name=John&email=john@example.com&customfield_field=value`

**Query Parameters**: product, amount, method, name, email, address, city, postal, notes, customfield_*, lang
**Product Links**: `product:1649` or numeric ID >100. Auto-detects product, shows showcase, hides amount field.
**readonly_dynamic_fields**: Makes pre-filled fields read-only.
**Controller**: `PaymentPageController@showWithParams()`, View: `payment/page.blade.php`

## Smart Links System

**Purpose**: Reusable, branded payment links for marketing campaigns, social media, and QR codes.

### Core Concept
- **Pretty URLs**: `https://payrequest.me/{businessname}/{path}`
- **Two Types**: Product checkout or fixed amount payment
- **Analytics**: Automatic tracking of views, payments, revenue, conversion rates
- **Customization**: Custom titles, descriptions, after-payment redirects

### Database Schema

**smart_links Table**:
- `user_id`, `name`, `description`, `internal_notes`, `path` (unique URL slug)
- `type` ('product' or 'amount'), `product_id`, `amount`, `currency`
- `params` (JSON, future use), `on_success_action`, `on_success_url`
- `is_active` (boolean toggle), `views_count`, `payments_count`, `revenue`
- `last_used_at` (tracks last view)

**transactions Table**:
- `smart_link_id` (nullable foreign key) - Links payments to smart links

### Key Components

**Models**:
- `SmartLink.php` - Main model with analytics methods, tag support
- `Transaction.php` - Added `smartLink()` relationship
- `Tag.php` - Added 'SmartLink' type support

**Controllers**:
- `SmartLinkResolverController.php` - Resolves pretty URLs, increments views, stores session data
- `PaymentPageController@showWithParams()` - Renders payment page with smart link data
- `MollieWebhookController` - Tracks payments via `smart_link_id`

**Livewire Components**:
- `SmartLinkList` - Table with search, type filter, tag filter, toggle active, duplicate
- `SmartLinkCreate` - Minimal form (type, name, product/amount, description, tags)
- `SmartLinkEdit` - Full form including path, internal notes, after-payment actions
- `SmartLinkShow` - Detail page with QR code, stats, recent payments

**Routes**:
- Numeric constraint on amount routes: `'amount' => '[0-9]+\.?[0-9]*'`
- Smart link route: `/{businessname}/{path}` with alphabetic constraint
- Route precedence: Payment page amounts → Smart links → Default payment page

### URL Resolution Flow

1. User visits `https://payrequest.me/payrequest/testen`
2. `SmartLinkResolverController` finds smart link by path and business name
3. Increments `views_count`, stores `smart_link_data` in session
4. **Product links**: Redirect to `/shop/{handle}/product/{id}?smart_link={id}`
5. **Amount links**: Render payment page directly (keeps URL visible)

### Payment Page Customization

When accessed via smart link:
- **Title**: Shows `$smartLinkData['name']` instead of payment page title
- **Description**: Shows `$smartLinkData['description']` instead of default
- **Amount Field**: Disabled (readonly) with smart link amount pre-filled
- **Min/Max Text**: Hidden (smart link amount is fixed)

### Payment Tracking

**MollieWebhookController** (`handlePayment()` method):
```php
if ($newStatus === 'paid' && session('current_smart_link_id')) {
    $smartLink = SmartLink::find(session('current_smart_link_id'));
    $transaction->smart_link_id = $smartLinkId;
    $smartLink->recordPayment($transaction->amount);
}
```

### Analytics Methods

**SmartLink Model**:
- `incrementViews()` - Increments views_count
- `recordPayment($amount)` - Increments payments_count, revenue, updates last_used_at
- `getConversionRate()` - Returns (payments / views) * 100
- `getAveragePaymentValue()` - Returns revenue / payments_count
- `getFormattedRevenue()` - Returns formatted currency string

### Technical Notes

- **PostgreSQL Booleans**: Use `DB::raw('true'/'false')` for `is_active` updates
- **Route Constraints**: Numeric amounts vs alphabetic paths prevent conflicts
- **Session Management**: `smart_link_data` passed from resolver to payment page
- **Tag Support**: Polymorphic relationship via `morphToMany('tags', 'taggable')`
- **QR Code API**: External service `https://api.qrserver.com/v1/create-qr-code/`

## Multi-PSP (Payment Service Provider) Architecture (January 2026)

The platform supports a unified payment provider architecture allowing businesses to connect multiple payment processors (Mollie, Stripe, PayPal).

### Core Components
- **PaymentProviderInterface**: Contract for all payment providers (`app/Services/Payments/Contracts/`)
- **RecurringPaymentInterface**: Extension for providers supporting subscriptions/mandates
- **PaymentProviderFactory**: Creates provider instances based on type
- **PaymentService**: Unified API for payment operations with feature flag support
- **MolliePaymentProvider**: Wraps existing MollieService (`app/Services/Payments/Providers/`)

### DTOs (`app/Services/Payments/Contracts/DTOs/`)
- `PaymentRequest` - Payment input (amount, currency, method, metadata)
- `PaymentResponse` - Payment result (status, checkout URL, provider ID)
- `RefundRequest/RefundResponse` - Refund operations
- `MandateResponse` - Recurring payment mandates
- `PaymentMethod` - Saved payment method details

### Feature Flag
- **Config**: `config/payments.php` with `use_unified_service`
- **Env**: `PAYMENTS_USE_UNIFIED=true` enables new system
- **Fallback**: When disabled, uses legacy MollieService directly
- **Bridge Method**: `PaymentService::createInvoicePayment()` handles routing

### Database Schema Extensions
- `payment_providers.type` - Provider type (mollie, stripe, paypal)
- `payment_providers.is_default` - Default provider for user
- `saved_payment_methods` - New table for mandates/tokens across providers
- `transactions` - Added `provider_id`, `provider_type`, `provider_transaction_id`, `saved_payment_method_id`
- `subscriptions` - Added `provider_id`, `provider_type`, `saved_payment_method_id`

### Implementation Status (See `MULTI_PSP_IMPLEMENTATION_PLAN.md`)
- Phase 1: Foundation & Abstraction Layer - Complete
- Phase 2: Mollie Adapter - Complete (61 unit tests passing)
- Phase 3-6: Stripe, PayPal, Multi-Provider Checkout - Not started

### Usage Example
```php
// Using unified service (recommended)
$response = PaymentService::createInvoicePayment($user, $invoice, 'ideal');

// Using factory directly
$provider = PaymentProviderFactory::forUser($user, 'mollie');
$response = $provider->createPayment($request);
```

## MCP (Model Context Protocol) Integration (January 2026)

PayRequest exposes an MCP server for AI clients (like Claude Desktop) to interact with billing data.

### Components
- **BillingServer**: Main MCP server at `/mcp/billing` (`app/Mcp/Servers/BillingServer.php`)
- **OAuth Authentication**: Laravel Passport with `mcp:use` scope (via Dynamic Client Registration)
- **Routes**: Registered in `routes/ai.php` via `Mcp::web()` and `Mcp::oauthRoutes()`
- **PostgreSQL Fix**: Custom Passport models handle PostgreSQL boolean types

### Available Tools
- `GetInvoiceStatsTool` - Invoice statistics (total, paid, unpaid, overdue)
- `GetOverdueInvoicesTool` - List overdue invoices with pagination
- `ListInvoicesTool` - List invoices with filters (status, date range, customer)
- `SendInvoiceReminderTool` - Send payment reminder emails

### OAuth Flow (Dynamic Client Registration)
1. MCP client discovers OAuth endpoints at `/.well-known/oauth-authorization-server/mcp/billing`
2. Client registers itself via `POST /oauth/register` (RFC 7591)
3. User authorizes via browser at `/oauth/authorize`
4. Client receives access token with `mcp:use` scope
5. Client calls MCP endpoint with Bearer token

### Custom Passport Models (PostgreSQL Boolean Fix)
- `PassportClient` - Handles `revoked`, `personal_access_client`, `password_client` booleans
- `PassportToken` - Handles `revoked` boolean
- `PassportAuthCode` - Handles `revoked` boolean
- `PassportRefreshToken` - Handles `revoked` boolean
- Custom query builders intercept `where()` to use PostgreSQL `?::boolean` syntax
- Registered in `AuthServiceProvider` via `Passport::use*Model()`

### Key Files
- `app/Mcp/Servers/BillingServer.php` - MCP server definition
- `app/Mcp/Tools/*.php` - Individual MCP tools
- `app/Models/Passport*.php` - Custom Passport models with PostgreSQL fix
- `routes/ai.php` - MCP route registration
- `resources/views/passport/authorize.blade.php` - OAuth authorization view

### Claude Desktop Configuration (claude_desktop_config.json)
```json
{
  "mcpServers": {
    "payrequest-billing": {
      "url": "https://payrequest.app/mcp/billing"
    }
  }
}
```

## Saved Payment Method System

- **Purpose**: Allow customers to pay invoices using previously saved SEPA Direct Debit mandates
- **Architecture**: Integration with Mollie's SEPA mandate system
- **Key Components**:
  - `ShopCustomerInvoiceDetail` component handles saved payment method UI and processing
  - `loadAvailableMandates()` fetches customer's active SEPA mandates
  - `payWithSavedMethod()` creates payments using existing mandates
  - FluxUI modal for payment method selection between new vs saved payment methods
- **Duplicate Prevention**: System prevents multiple simultaneous payments for same invoice
- **Payment Flow**: Mandate → Payment Creation → Status Tracking → Transaction History
- **Technical Details**: Uses MollieService with reflection-based HTTP adapter access for raw API responses
