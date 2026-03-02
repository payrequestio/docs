# Customer Portal & Shop

> Reference file for CLAUDE.md - Portal architecture, shared components, cart, customer types

## Portal Architecture

PayRequest is a web-based Dashboard where business owners manage their entire billing workflow:
- Customer Portal & Products (portal with built-in shop)
- Invoices & Orders
- Subscriptions
- Customers
- Activity Log
- Tag Management
- Email Automations

Each business gets:
- An internal dashboard (for managing everything)
- An external customer portal (used by their customers to browse products, pay invoices, and manage subscriptions)

### UI Terminology (February 2026 rename)
- Sidebar: "Portal & Products" group → "Customer Portal" item
- Profile dropdown: "View Portal" (was "View Shop")
- /shop management page: "Customer Portal" headings, "Portal Settings" tab, "Portal Logo", etc.
- **Routes/URLs still use `/shop/` prefix** — only UI labels changed, no route renames
- Internal code (models, controllers, blade filenames) still uses "shop" naming — only user-facing text updated

## Key Product Concepts

### Business Model
- Multi-Tenant: every user (business owner) has their own environment (portal, products, customers).
- Customers of those businesses use the external customer portal to interact.

### Important Concepts
- **Customer Portal** = the primary external-facing experience per business. Portal with built-in shop.
- Invoice = can contain products and/or custom invoice items.
- Subscription = recurring payment for a product.
- Order = one-off purchase of products.

## Tags System

All main entities (Invoice, Customer, Subscription, Product, Order) can have tags (labels). Tags help with segmentation and automation (internal usage, not visible to end customers).

## Customer Types (Individual vs Business)

- **Two customer types**: `customer_type` field ('individual' or 'business')
- **Individual customers**: Use `name` field (full name), standard contact details
- **Business customers**: Use `company` field (company name), additional fields:
  - `company_number` (KVK number for Dutch businesses)
  - `company_country`, `legal_form`, `industry_code`, `industry_description`
  - `established_at`, `website_url`, `trade_names` (JSON array)
  - `kvk_status`, `kvk_source`, `kvk_last_synced_at` (KVK.nl API integration)
  - `contact_name`, `contact_email`, `contact_phone` (business contact person)
- **CRITICAL**: Always use `$customer->getDisplayName()` in views, NOT `$customer->name` directly
- **Display Email Helper**: `getDisplayEmail()` handles Mollie imported customers with placeholder emails

## Main Models

- User → Business Owner (has business_details JSON field)
- Customer → End Customer of Business Owner (session-based auth in shop)
- Product (supports custom_fields JSON and custom_fields_config JSON)
- Subscription, Invoice, Order, OrderItem (has options JSON)
- Transaction (linked to invoices/orders/subscriptions)
- Tag, Taggable (polymorphic linking table)
- Shop (business storefront configuration)

## Custom Fields System

- Products can have dynamic custom fields (text, email, url, number, textarea)
- Fields have visibility settings (dashboard-only vs public in shop)
- Customer data flows from product page → cart → checkout → order
- Custom field data stored in OrderItem.options JSON field

## Cart Management

- Session-based cart with custom field support
- Cart items can be edited/removed during checkout
- CartService handles all cart operations with index-based management

## Setup Fee Customization

**Fields**: products.setup_fee_name/setup_fee_description (custom names instead of generic "Setup Fee")
**Accessors**: Default to __('Setupkosten') and "One-time setup fee"
**Display**: Cart, checkout, invoices use custom names from product
**Use Cases**: "Borg" (deposit), "Onboarding Fee", "Installatiekosten"

```php
public function getSetupFeeNameAttribute($value): string
{
    return $value ?: __('Setupkosten');
}
```

## Product Sharing System

**Purpose**: Share product links with QR codes via modal (similar to invoice sharing).
**Component**: ProductDetail component with `shareProductLink()` method
**URL Priority**: Custom domain first, then PayRequest subdomain
**QR Code**: `https://api.qrserver.com/v1/create-qr-code/?size=200x200&data={url}`
**Modal**: FluxUI modal with copy-to-clipboard

## Shared Blade Components

### Product Custom Fields Component
`resources/views/components/product-custom-fields.blade.php`

Reusable for product configuration UI (custom fields + quantity selector). Used by both shop product page and payment/sales page.

```blade
<x-product-custom-fields
    :fields="$visibleCustomFields"
    :values="$customFieldValues"
    :disabled="$customFieldsDisabled"
    wireModel="customFieldValues"
    :showQuantity="$product->allow_quantity_selection"
    :quantity="$quantity"
    theme="auto"
    :hiddenInputs="false"
/>
```

**Used By**: `product-view.blade.php` (shop), `payment-page-custom-fields.blade.php` (payment page)
**Supported Field Types**: text, textarea, email, url, number (with slider for per-unit markup), select, toggle

### Domain Search Component
`resources/views/components/domain-search.blade.php`

Reusable for domain search and configuration UI. Used by both shop product page (cart mode) and payment/sales page (direct payment mode).

**Used By**: `shop/domain-search.blade.php` (cart mode), `payment-page-domain-search.blade.php` (direct mode)
**Flow Differences**:
- **Cart mode**: Search → Select → Configure nameservers → "Add to Cart" → Redirect to checkout
- **Direct mode**: Search → Select → Configure nameservers → "Confirm Domain" → Hidden inputs → Pay button

## UX Principles

- The Dashboard is built for efficiency: single-page app feel (Livewire + FluxUI).
- The external Customer Portal must be clean, minimal, mobile-friendly.
- The system should never crash or give fatal errors on customer-visible pages.
- Payment status and transaction tracking must be fully accurate and up-to-date in the dashboard.

## Page Header Standards

**MANDATORY Pattern**: @section('header') with flux:main, header-content max-w-6xl mx-auto, flux:heading size="xl" level="1", flux:breadcrumbs, flux:separator
**Breadcrumb Patterns**: Dashboard → Entity → Instance → Action
**Content**: Main content uses max-w-6xl container with py-10

## Card Design System

**Base Card**: `<div class="entity-card bg-white dark:bg-zinc-900 rounded-xl p-6 shadow-sm border border-zinc-200 dark:border-zinc-800">`
**Hover CSS**: `.entity-card:hover { transform: translateY(-1px); box-shadow: 0 8px 25px -5px rgba(0,0,0,0.1); }`
**Grid**: `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6`
**Colors**: Blue (primary), Purple (actions), Green (success), Red (danger), Zinc (neutral)
**NEVER use flux:card** - Always use div with modern card classes
