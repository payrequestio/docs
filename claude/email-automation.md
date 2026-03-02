# Email & Automation System

> Reference file for CLAUDE.md - Email architecture, automation triggers

## Email Automation System

**Components**: Automation Model, AutomationService, ResendWebhookController, EmailIssue Model
**Triggers**: email_bounced, email_complained, email_failed
**Actions**: add_tag, remove_tag, update_status
**UI**: /automations route, card-based interface
**Health Badges**: Healthy (green), Bounced (red), Complained (orange), Failed (yellow)

## Email System Architecture

**Configuration**: MAIL_MAILER=postmark, uses Postmark API (NOT SMTP)
**Template Service**: EmailTemplateService renders templates with {{variable}} replacement
**Email Types**: invoice_created, invoice_paid (auto-sent via jobs)
**Queue**: 'emails' queue (8 max processes in Horizon)
**PDF Attachments**: DomPDF generates invoices automatically
**Database**: email_templates table with is_active boolean
**CRITICAL**: Load relationships before job dispatch, use whereRaw() for boolean queries

## Internationalization (i18n) & Localization

**CRITICAL**: ALL user-visible text MUST use __() functions. NO EXCEPTIONS.

**Commands** (run after EVERY file change):
- `php artisan i18n:scan --write-missing=nl` - Scan for new keys
- `php artisan i18n:report` - Check coverage (must be 100%)
- `php artisan i18n:translate nl --source=en` - Auto-translate

**Usage**:
- Blade: `{{ __('text') }}`
- PHP: `__('text')` or `trans('text')`
- Variables: `__('Welcome :name', ['name' => $user->name])`
- Pluralization: `trans_choice('item.count', $count)`

**Files**: `/lang/nl.json`, `/lang/en.json` (JSON takes precedence), `/lang/_tracking.json` (source of truth)
**Do-Not-Translate**: PayRequest, Mollie, Stripe, SEPA, IBAN, API, JSON, EUR, USD
