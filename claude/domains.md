# Domain Provisioning & Cloudflare

> Reference file for CLAUDE.md - Domain registration, Cloudflare forwarding & DNS

## Domain Provisioning System

**Purpose**: Automated domain registration, management, and annual renewals through provider APIs (OpenProvider)

### Architecture Overview

**Product Type**: Single "domain" product with dynamic TLD selection via domain search
**Provider System**: Extensible provision provider architecture supporting OpenProvider (and future providers)
**TLD Management**: Per-business TLD pricing with customizable profit margins
**Annual Subscriptions**: Automatic subscription creation for yearly domain renewals

### Database Schema

**provision_providers** (New Table):
- id, user_id, type (openprovider), name, credentials (encrypted), is_active, settings, last_sync_at

**domain_tlds** (New Table):
- id, user_id, provider_id, tld (.com, .nl, etc), cost_price, sell_price, transfer_price, renewal_price, is_active, metadata

**domain_registrations** (New Table):
- id, user_id, customer_id, product_id, subscription_id, tld_id, provider_id, domain_name, status, registered_at, expires_at, auto_renew, nameservers, contacts, provider_domain_id, auth_code, metadata
- **Status Types**: pending, active, expired, transferred, canceled

**products** (Enhanced):
- type (standard, domain, deposit), provider_id, provision_config

### Service Architecture

```
app/Services/Provision/
├── Contracts/
│   ├── ProvisionProviderInterface.php (base interface)
│   └── DomainProvisionInterface.php (domain-specific methods)
└── OpenProvider/
    ├── OpenProviderClient.php (API wrapper)
    └── OpenProviderService.php (implements DomainProvisionInterface)
```

**OpenProvider API Integration**:
- Base URL: `https://api.openprovider.eu` (production) or `https://api.cte.openprovider.eu` (test)
- Authentication: Token-based via `/v1beta/auth/login`
- Key Endpoints: domains/check, domains (register), products/extensions (TLD pricing)

### Models & Relationships

**ProvisionProvider**: user(), domainTlds(), domainRegistrations(), products(). Credentials encrypted via Crypt.
**DomainTld**: user(), provider(), domainRegistrations(). Helpers: isActive(), getMarginPercentage(), getMarginAmount()
**DomainRegistration**: user(), customer(), product(), subscription(), tld(), provider(). Helpers: isActive(), isPending(), isExpired(), daysUntilExpiry(). Scopes: active(), expiringWithin($days), autoRenew()

### Commands

**php artisan domains:sync-tlds**: Syncs TLD pricing from OpenProvider API. Options: `--provider={id}`, `--margin=20`

### Key Flows

**Domain Search**: Customer enters domain → System checks via OpenProvider API → Available TLDs shown with prices → Customer selects TLD → adds to cart

**Registration**: Checkout + payment → Domain registration job → OpenProviderService registers via API → DomainRegistration record → Subscription for annual renewal

**Renewal**: Subscriptions with 1-year interval → Billing job checks expiring → Auto-renew via OpenProvider API

### Provider Extensibility

1. Create provider service class implementing `DomainProvisionInterface`
2. Add provider type to `ProvisionProvider::getServiceClass()` match statement
3. Update sync command to support new provider type

## Cloudflare Domain Forwarding & DNS Editing

**Purpose**: Customers with domain subscriptions (registered via OpenProvider, using Cloudflare NS) can set up domain forwarding and manage DNS records from the customer portal.

### Forwarding Architecture

**Cloudflare API**: Uses Single Redirects via the Rulesets API (`/zones/{zone_id}/rulesets/phases/http_request_dynamic_redirect/entrypoint`)
**Rule Identification**: PayRequest-managed rules identified by description `"PayRequest Domain Forwarding"`. Non-PayRequest rules preserved.
**DNS Prerequisite**: A proxied A record pointing to `192.0.2.1` (RFC 5737 documentation IP) — required for redirects.
**Config Storage**: Forwarding config cached in `cloudflare_zones.metadata` JSON field.

### CloudflareService Methods

- `getRedirectRules(zoneId)` — GET redirect rules, handles 404 as empty result
- `createRedirectRule(zoneId, targetUrl, statusCode, preservePath, preserveQueryString)` — PUT (idempotent)
- `deleteRedirectRules(zoneId)` — Removes only PayRequest-managed rules
- `setupDomainForwarding(zoneId, ...)` — Orchestrator: ensures proxied root A record → creates redirect rule
- `ensureProxiedRootRecord(zoneId)` — (private) Creates/updates proxied A record to `192.0.2.1`

### Customer Portal (ShopCustomerSubscriptionDetail)

**Forwarding Tab**: Target URL, Redirect Type 301/302, Preserve Path/Query String toggles
**DNS Tab**: Full CRUD for DNS records using `flux:table`. Edit via modal, delete with `wire:confirm`, create via "Add Record" button.
**Domain Name Display**: Domain subscriptions show the domain name as primary heading instead of product name.

### Admin Dashboard (CloudflareApp)

"Set Up Forwarding" / "Edit Forwarding" button per zone. Purple "Forwarding" badge on zones with active forwarding.

### Required Cloudflare API Token Permissions

- `Zone:Read`, `DNS:Edit`, `Zone Rulesets:Edit` (required for forwarding)
- **Common Error**: `Authentication error` on forwarding = missing `Zone Rulesets:Edit` permission

### Key Files

- `/app/Services/CloudflareService.php` — All Cloudflare API methods
- `/app/Livewire/ShopCustomerSubscriptionDetail.php` — Customer portal: forwarding + DNS
- `/app/Livewire/Apps/CloudflareApp.php` — Admin forwarding management

### Expression Format

**Preserve path**: `concat("https://target.com/path", http.request.uri.path)`
**Static URL**: `"https://target.com/path"`
**Parsing**: Existing rules parsed back to form fields via `parseForwardingRule()`
