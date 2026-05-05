# Bravo Benefits — Standardized Database Schema Documentation

> **Generated:** May 1, 2026
> **Project:** Bravo Benefits Platform
> **Database:** PostgreSQL (via SQLAlchemy / Flask-SQLAlchemy)
> **Source Schema:** `bravo_benefits_standardized.md` (April 28, 2026) — post-architecture-review refactor
> **Standard Applied:** Flask Boilerplate `DB_SCHEMA.md` architectural conventions
> **Scope:** Domain 1 — Core Identity & Auth · Domain 2 — Benefits & TRS · Domain 3 — Reward & Recognition · Domain 4 — Content, Support & Marketing · Domain 5 — Platform Config & Scheduling · Domain 6 — Analytics

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture: Multi-Tenancy](#architecture-multi-tenancy)
3. [Base Model & Common Fields](#base-model--common-fields)
4. [Mixins](#mixins)
5. [Table Definitions](#table-definitions)

   **Domain 1 — Core Identity & Auth**
   - [groups](#groups)
   - [sectors](#sectors)
   - [lead_sources](#lead_sources)
   - [organisations](#organisations)
   - [users](#users)
   - [user_org_memberships](#user_org_memberships)
   - [org_teams](#org_teams)
   - [user_org_teams](#user_org_teams)
   - [org_internal_notes](#org_internal_notes)
   - [image](#image)

   **Domain 2 — Benefits & TRS**
   - [master_benefits](#master_benefits)
   - [benefit_levels](#benefit_levels)
   - [org_benefits](#org_benefits)
   - [org_benefit_team_levels](#org_benefit_team_levels)
   - [custom_tiles](#custom_tiles)
   - [benefit_requests](#benefit_requests)
   - [employee_benefit_status](#employee_benefit_status)
   - [trs_config](#trs_config)
   - [trs_component_types](#trs_component_types)
   - [trs_components](#trs_components)
   - [trs_employee_data](#trs_employee_data)
   - [trs_statements](#trs_statements)
   
   **Domain 3 — Reward & Recognition**
   - [rnr_config](#rnr_config)
   - [rnr_teams](#rnr_teams)
   - [rnr_user_teams](#rnr_user_teams)
   - [rnr_values](#rnr_values)
   - [rnr_nominations](#rnr_nominations)
   - [rnr_rewards](#rnr_rewards)
   - [rnr_budget_individual](#rnr_budget_individual)
   - [rnr_budget_team](#rnr_budget_team)

   **Domain 4 — Content, Support & Marketing**
   - [documents](#documents)
   - [notices](#notices)
   - [notice_likes](#notice_likes)
   - [support_tickets](#support_tickets)
   - [support_ticket_messages](#support_ticket_messages)
   - [notifications](#notifications)
   - [templates](#templates)
   - [marketing_categories](#marketing_categories)
   - [marketing_assets](#marketing_assets)
   - [adverts](#adverts)
   - [advert_user_suppression](#advert_user_suppression)
   - [fact_sheets](#fact_sheets)
   - [internal_conversations](#internal_conversations)
   - [internal_messages](#internal_messages)

   **Domain 5 — Platform Config & Scheduling**
   - [scheduled_jobs](#scheduled_jobs)

   **Domain 6 — Analytics**
   - [analytics_events](#analytics_events)

6. [Deleted Tables — Migration Notes](#deleted-tables--migration-notes)
7. [Entity Relationship Summary](#entity-relationship-summary)
8. [Migration Order](#migration-order)

---

## Overview

The **Bravo Benefits Platform** is a multi-tenant SaaS application providing employee benefits management and Total Reward Statements (TRS) to client organisations. This document covers the two foundational domains:

| Domain | Tables | Description |
|---|---|---|
| **Core Identity & Auth** | `groups`, `sectors`, `lead_sources`, `organisations`, `users`, `user_org_memberships`, `org_teams`, `user_org_teams`, `org_internal_notes`, `image` | Identity, authentication, org structure, branding, and SSO |
| **Benefits & TRS** | `master_benefits`, `benefit_levels`, `org_benefits`, `org_benefit_team_levels`, `custom_tiles`, `benefit_requests`, `employee_benefit_status`, `trs_config`, `trs_component_types`, `trs_components`, `trs_employee_data`, `trs_statements` | Benefit catalogue, org offerings, and Total Reward Statements |
| **Reward & Recognition** | `rnr_config`, `rnr_teams`, `rnr_user_teams`, `rnr_values`, `rnr_nominations`, `rnr_rewards`, `rnr_budget_individual`, `rnr_budget_team` | Peer recognition, wall of fame, reward budgets, and perk catalogue |
| **Content, Support & Marketing** | `documents`, `notices`, `notice_likes`, `support_tickets`, `support_ticket_messages`, `notifications`, `templates`, `marketing_categories`, `marketing_assets`, `adverts`, `advert_user_suppression`, `fact_sheets`, `internal_conversations`, `internal_messages` | Employee communications, support workflow, internal broadcast messaging, marketing library, adverts, and health content |
| **Platform Config & Scheduling** | `scheduled_jobs` | Background job queue for all time-based platform operations |
| **Analytics** | `analytics_events` | Range-partitioned raw event log powering self-serve analytics dashboards |


---

## Architecture: Multi-Tenancy

Multi-tenancy in the Bravo Benefits platform is implemented using a **shared-database, shared-schema** strategy with two complementary isolation mechanisms:

### Primary Isolation — `org_uuid`

1. **Every `Base`-derived model** contains an `org_uuid` column (FK → `organisations.uuid`), automatically stamped on `save()` from `request.org_uuid`.
2. `Base.get_base_query()` automatically injects `.filter(org_uuid == <current_org>)` on every query, derived from the request context set by the authentication middleware.
3. If no `org_uuid` is present (background jobs, Bravo staff operations), queries return all records — enabling cross-organisation superadmin access.
4. Admin bypass methods (`get_by_uuid_global`, `get_by_org_uuid`) allow deliberate cross-organisation lookups.

### Secondary Isolation — `user_org_memberships`

The `user_org_memberships` table provides a second layer: **one user can belong to multiple organisations** with different roles. Every business operation that involves a user should resolve their active membership (`is_active=True`) for the current `org_uuid` before applying logic.

```
Request → Auth Middleware (sets request.org_uuid + request.user_uuid)
        → Base.get_base_query() → org_uuid filtered Results
        → UserOrgMembership.get_active() → role-gated logic
```

### Bravo Staff Bypass

Users flagged `is_bravo_user=True` on the `users` table are Bravo platform staff. They bypass the standard `org_uuid` filter (no membership required) and operate across all organisations.

> **Key Design Rules:**
> - Never query a business table without an `org_uuid` filter unless explicitly performing a Bravo-staff or admin operation.
> - `org_teams` (benefit-access teams) are entirely distinct from any R&R team concept — never merge them.

---

## Base Model & Common Fields

### `Base` (abstract — `app/models/base.py`)

All business models in this project inherit from `Base`. It provides the following standard columns on every table:

| Column | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Internal surrogate primary key — never exposed in API responses |
| `uuid` | `String` | Unique | Public-facing universally unique identifier (UUID4) — used in all API references |
| `org_uuid` | `String` | FK → `organisations.uuid`, Nullable | Tenant isolation key; auto-stamped from request context on `save()` |
| `created_at` | `DateTime (tz)` | NOT NULL, default=now | Timestamp of record creation |
| `updated_at` | `DateTime (tz)` | NOT NULL, default=now, onupdate=now | Timestamp of last update |
| `deleted_at` | `DateTime (tz)` | Nullable | Soft-delete timestamp; `NULL` = active record |
| `created_by` | `String` | FK → `users.uuid`, Nullable | UUID of user who created the record (via `UserAuditMixin`) |
| `updated_by` | `String` | FK → `users.uuid`, Nullable | UUID of user who last updated the record (via `UserAuditMixin`) |
| `deleted_by` | `String` | FK → `users.uuid`, Nullable | UUID of user who soft-deleted the record (via `UserAuditMixin`) |

> **Note on `is_active`:** The `is_active` boolean is **retained** where it carries business semantics distinct from deletion (e.g., `org_teams.is_active` means "team is operational", not "deleted"). The standard `deleted_at` field handles true soft-deletes. Both may coexist on the same table.

### Abstract Sub-Bases

#### `PincodeBase` (extends `Base`)
Available for any location-aware models:

| Column | Type | Description |
|---|---|---|
| `pincode` | `String` | Postal / ZIP code |
| `country` | `String` | Country name or ISO code |
| `state` | `String` | State / province |
| `city` | `String` | City name |

#### `FileBase` (extends `Base`)
Used by `image` and document-style models:

| Column | Type | Constraints | Description |
|---|---|---|---|
| `module_uuid` | `String` | Nullable | UUID of the owning module record |
| `module_type` | `String` | NOT NULL | Discriminator for the owning module type |
| `name` | `String` | Nullable | File name |
| `path` | `String` | Nullable | Storage path (S3 prefix) |
| `size` | `String` | Nullable | File size (stored as string) |
| `file_format` | `String` | Nullable | File extension / MIME type |
| `description` | `Text` | Nullable | Optional description |

---

## Mixins

### `UserAuditMixin` (`app/models/base.py`)

Injects `created_by`, `updated_by`, `deleted_by` FK columns (→ `users.uuid`) and their SQLAlchemy relationships (`created_by_user`, `updated_by_user`, `deleted_by_user`) via `@declared_attr`. Included in `Base` and `organisations` (which does not extend `Base` to avoid circular FK).

### `AuditableEvent` (`app/models/audit_event.py`)

Registers SQLAlchemy `after_insert`, `after_update`, `after_delete` listeners on any inheriting model. On each event, automatically writes a row to `audit_log` capturing:
- Table name, object ID, and action (`CREATE` / `UPDATE` / `DELETE`)
- `state_before` / `state_after` JSON snapshots of changed columns
- Request metadata (HTTP method, URL, IP) from Flask's request context

**Sensitive fields excluded from audit snapshots:** `password_hash`, `two_fa_secret`, `client_secret`, `access_token`, `refresh_token`.

All `Base`-derived models inherit `AuditableEvent` automatically.

### `SqlAlchemyEventMixin` (`app/models/event_mixin.py`)

Provides overridable lifecycle hooks for fine-grained SQLAlchemy event handling:

| Hook | Trigger |
|---|---|
| `on_before_insert` / `on_after_insert` | Row insert |
| `on_before_update` / `on_after_update` | Row update |
| `on_before_delete` / `on_after_delete` | Row delete |
| `on_before_flush` / `on_after_flush` | Session flush |
| `on_before_commit` | Session commit |

Recommended for: `users`, `user_org_memberships` — models with complex domain lifecycle logic (e.g., triggering notifications, enforcing business rules on membership state changes).

---

## Table Definitions

---

### Domain 1 — Core Identity & Auth

---

### `groups`

**Business Purpose:** Org group entities managed by Bravo staff. Enables group admins to push documents and notices across multiple linked organisations simultaneously. Does **not** extend `Base` — it is a platform-level lookup with no org tenancy scope.

**Base Class:** `db.Model` + `UserAuditMixin`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Internal surrogate primary key — never exposed in API responses |
| `uuid` | `String` | Unique | Public UUID used as the FK reference (`group_uuid`) in `organisations` |
| `group_name` | `String(200)` | NOT NULL | Human-facing display name of the group, shown in the Bravo staff dashboard group-picker |
| `is_active` | `Boolean` | NOT NULL, default=True | Controls whether the group appears as selectable in the admin UI; inactive groups still serve their linked orgs |
| `created_at` | `DateTime (tz)` | NOT NULL, default=now | Record creation timestamp |
| `updated_at` | `DateTime (tz)` | NOT NULL, default=now | Last update timestamp |
| `created_by` | `String` | FK → `users.uuid`, Nullable | UUID of the Bravo staff user who created this group (via `UserAuditMixin`) |
| `updated_by` | `String` | FK → `users.uuid`, Nullable | UUID of the Bravo staff user who last modified this group (via `UserAuditMixin`) |

---

### `sectors`

**Business Purpose:** Industry sector lookup table used purely for CRM classification of organisations. Managed exclusively by Bravo staff — client orgs cannot create or modify sectors. Platform-level lookup with no org tenancy scope.

**Base Class:** `db.Model` + `UserAuditMixin`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Internal surrogate primary key |
| `uuid` | `String` | Unique | Public UUID referenced as `sector_uuid` on `organisations` |
| `sector_name` | `String(150)` | NOT NULL | Human-facing sector label shown in the organisation creation form (e.g. `Financial Services`, `Healthcare`) |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether this sector is selectable when creating or editing an organisation |
| `created_at` | `DateTime (tz)` | NOT NULL, default=now | Record creation timestamp |
| `updated_at` | `DateTime (tz)` | NOT NULL, default=now | Last update timestamp |

---

### `lead_sources`

**Business Purpose:** Hierarchical lead source lookup for CRM tracking, recording how each client organisation was acquired. Supports self-referential parent-child groupings (e.g. a `Digital` header with `LinkedIn`, `Google Ads` children). Managed by Bravo staff only.

**Base Class:** `db.Model` + `UserAuditMixin`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Internal surrogate primary key |
| `uuid` | `String` | Unique | Public UUID referenced as `lead_source_uuid` on `organisations` |
| `parent_uuid` | `String` | FK → `lead_sources.uuid`, Nullable | Self-referential parent UUID; NULL = root-level node. Allows grouping sub-sources under a common header |
| `source_name` | `String(200)` | NOT NULL | Display name of the lead source shown in the org CRM form |
| `is_subheader` | `Boolean` | NOT NULL, default=False | When True, this row renders as a non-selectable display subheader in the dropdown — used purely for visual grouping |
| `display_order` | `Integer` | NOT NULL, default=0 | Controls sort order within a parent group; lower = higher in the list |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether this source is selectable on the org form |
| `created_at` | `DateTime (tz)` | NOT NULL, default=now | Record creation timestamp |
| `updated_at` | `DateTime (tz)` | NOT NULL, default=now | Last update timestamp |

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `parent` | `LeadSource` | Self Many-to-One | Parent lead source category | `ON DELETE SET NULL` — orphaned child becomes a root node |
| `children` | `LeadSource` | Self One-to-Many | Child lead source entries | `ON DELETE SET NULL` |

---

### `organisations`

**Business Purpose:** Root tenant entity representing a client company. Every piece of business data in the platform is scoped to an organisation via `org_uuid`. Does **not** extend `Base` — avoids circular FK dependencies since `Base` itself has `org_uuid → organisations.uuid`. Branding, feature flags, and SSO configuration are all consolidated into the `settings` JSONB column to reduce table width and group related configuration logically.

> **2FA Policy Note:** The `force_2fa_for_all` column has been **removed**. Two-factor authentication is now exclusively a user-level choice managed via `users.two_fa_enabled`. There is no org-level 2FA enforcement — individual users opt in or out via their own account settings.

**Base Class:** `db.Model` + `UserAuditMixin`

> ⚡ **Naming Note:** The Bravo schema uses `organisations` (British English). The public FK reference throughout the codebase is `org_uuid` (standardized from the original `org_id`). The internal `id` (BigInteger PK) and `uuid` (String, unique) follow standard boilerplate conventions.

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Internal surrogate primary key |
| `uuid` | `String` | Unique, default=uuid4 | Public identifier; used as the `org_uuid` FK value in every child table across the platform |
| `group_uuid` | `String` | FK → `groups.uuid`, Nullable | Links this org to a Bravo group, enabling group admins to push content across all member orgs simultaneously |
| `sector_uuid` | `String` | FK → `sectors.uuid`, Nullable | Industry sector classification used for CRM reporting and segmentation |
| `lead_source_uuid` | `String` | FK → `lead_sources.uuid`, Nullable | Records how this client was acquired; used for marketing attribution in the Bravo staff CRM view |
| `consultant_user_uuid` | `String` | FK → `users.uuid`, Nullable | UUID of the assigned Bravo consultant responsible for onboarding this org |
| `account_manager_user_uuid` | `String` | FK → `users.uuid`, Nullable | UUID of the assigned Bravo account manager for ongoing commercial relationship |
| `company_name` | `String(255)` | NOT NULL | Official company trading name; displayed as the org's identity throughout the platform |
| `sage_id` | `String(100)` | Nullable | Billing reference code in Sage accounting; used for invoice matching |
| `platform_type` | `String(10)` | NOT NULL, default=`full`, CHECK IN (`full`, `basic`) | Platform tier — `full` has all modules enabled; `basic` is a lighter offering with a restricted feature set |
| `launch_date` | `Date` | Nullable | The date the platform formally launched for this org's employees; used in onboarding scheduling |
| `employee_count` | `Integer` | Nullable | Expected or actual headcount; used for license sizing and bulk upload validation |
| `self_registration_url` | `Text` | Unique, Nullable | Unique URL slug used for employee self-registration without an admin invite |
| `settings` | `JSONB` | NOT NULL, default=`{}` | Consolidated org configuration blob. Contains three sub-objects: `branding` (visual identity), `features` (module feature flags), and `sso` (single sign-on configuration). See **`settings` JSONB Structure** below |
| `status` | `String(20)` | NOT NULL, default=`active`, CHECK IN (`active`, `suspended`, `closed`) | Organisation lifecycle status — `suspended` blocks employee logins while retaining data; `closed` triggers GDPR retention clock |
| `closed_at` | `DateTime (tz)` | Nullable | Timestamp when this org was formally closed; triggers the GDPR data retention deadline calculation |
| `retain_until` | `DateTime (tz)` | Nullable | GDPR data retention deadline, computed as `closed_at + 7 years`; set by the application on closure |
| `created_at` | `DateTime (tz)` | NOT NULL, default=now | Record creation timestamp |
| `updated_at` | `DateTime (tz)` | NOT NULL, default=now | Last update timestamp |
| `deleted_at` | `DateTime (tz)` | Nullable | Soft-delete timestamp; NULL = active |
| `created_by` | `String` | FK → `users.uuid`, Nullable | Bravo staff user who created this org record (via `UserAuditMixin`) |
| `updated_by` | `String` | FK → `users.uuid`, Nullable | Bravo staff user who last modified this org record (via `UserAuditMixin`) |
| `deleted_by` | `String` | FK → `users.uuid`, Nullable | Bravo staff user who soft-deleted this org record (via `UserAuditMixin`) |

> **SQLAlchemy `__table_args__` Note:** The following string ENUM columns require explicit `CheckConstraint` entries in the model's `__table_args__`:
> - `platform_type`: `CheckConstraint("platform_type IN ('full', 'basic')", name='ck_organisations_platform_type')`
> - `status`: `CheckConstraint("status IN ('active', 'suspended', 'closed')", name='ck_organisations_status')`

#### `settings` JSONB Structure


The `settings` column stores three logically grouped sub-objects. Developers should access nested values via `org.settings.get('branding', {}).get('logo_url')` pattern with safe fallbacks.

```json
{
  "branding": {
    "primary_hex":       "<string|null>",
    "secondary_hex":     "<string|null>",
    "logo_url":          "<string|null>",
    "header_image_url":  "<string|null>",
    "welcome_message":   "<string|null>"
  },
  "features": {
    "show_adverts":                "<boolean>",
    "show_fact_sheets":            "<boolean>",
    "is_bravo_broker":             "<boolean>",
    "is_trs_enabled":              "<boolean>",
    "is_rnr_enabled":              "<boolean>",
    "ni_number_mandatory":         "<boolean>",
    "is_teams_enabled":            "<boolean>",
    "is_notices_enabled":          "<boolean>",
    "is_documents_enabled":        "<boolean>",
    "is_analytics_enabled":        "<boolean>",
    "is_internal_messages_enabled":"<boolean>"
  },
  "sso": {
    "provider":      "<string|null>",
    "is_enabled":    "<boolean>",
    "client_id":     "<string|null>",
    "client_secret": "<string|null>",
    "tenant_id":     "<string|null>",
    "metadata_url":  "<string|null>"
  }
}
```

| Key Path | Type | Default | Description |
|---|---|---|---|
| `branding.primary_hex` | `String(7)` | `null` | Primary brand colour in hex format (e.g. `#FF5733`); used as the main header and button colour throughout the employee portal. *Migrated from the removed top-level `primary_hex` column.* |
| `branding.secondary_hex` | `String(7)` | `null` | Secondary brand colour in hex format; used for accents, hover states, and secondary UI elements. *Migrated from removed `secondary_hex`.* |
| `branding.logo_url` | `String` | `null` | S3 URL of the organisation's logo image; rendered in the portal header on every page. *Migrated from removed `logo_url`.* |
| `branding.header_image_url` | `String` | `null` | S3 URL of the platform header/banner image displayed on the employee homepage. *Migrated from removed `header_image_url`.* |
| `branding.welcome_message` | `String` | `null` | Custom welcome message shown on the employee login/landing page. *Migrated from removed `welcome_message`.* |
| `features.show_adverts` | `Boolean` | `false` | When True, Bravo-managed adverts are shown to employees in this org's portal. *Migrated from removed `show_adverts`.* |
| `features.show_fact_sheets` | `Boolean` | `true` | When True, Module 9 fact sheets are visible to this org's employees (all-or-nothing — no selective team assignment). *Migrated from removed `show_fact_sheets`.* |
| `features.is_bravo_broker` | `Boolean` | `false` | When True, Bravo acts as the broker for this organisation's benefits — affects broker-specific token visibility in `master_benefits.required_tokens`. *Migrated from removed `is_bravo_broker`.* |
| `features.is_trs_enabled` | `Boolean` | `false` | Master feature flag enabling the Total Reward Statement module (chargeable add-on). When False, all TRS routes return 403 for this org's users. *Migrated from removed `is_trs_enabled`.* |
| `features.is_rnr_enabled` | `Boolean` | `false` | Master feature flag enabling the Reward & Recognition module. When False, all R&R routes return 403 for this org's users. *Migrated from removed `is_rnr_enabled`.* |
| `features.ni_number_mandatory` | `Boolean` | `true` | When True, NI number is a required field on employee bulk upload and the profile form; set to False for orgs with non-UK employees who do not have NI numbers. *Migrated from removed `ni_number_mandatory`.* |
| `features.is_teams_enabled` | `Boolean` | `false` | Master feature flag enabling the Org Teams module for this organisation. When False, team management routes and benefit team-level assignment UIs are hidden. |
| `features.is_notices_enabled` | `Boolean` | `false` | Master feature flag enabling the Notices module. When False, the notices section is hidden from the employee portal and org admin UI. |
| `features.is_documents_enabled` | `Boolean` | `false` | Master feature flag enabling the Documents module. When False, document upload, listing, and download routes return 403 for this org's users. |
| `features.is_analytics_enabled` | `Boolean` | `false` | Master feature flag enabling the self-serve Analytics dashboard for this organisation. When False, analytics routes and the dashboard tab are hidden from org admins. |
| `features.is_internal_messages_enabled` | `Boolean` | `false` | Master feature flag enabling the Internal Conversations (broadcast messaging) module. When False, `internal_conversations` and `internal_messages` routes are disabled for this org. |
| `sso.provider` | `String` | `null` | SSO provider type. Allowed values: `google`, `microsoft`, `okta`, `saml`, `other`. *Migrated from deleted `sso_config.provider`.* |
| `sso.is_enabled` | `Boolean` | `false` | Whether SSO login is currently active for this organisation. When True, password-based login is suppressed. *Migrated from deleted `sso_config.is_enabled`.* |
| `sso.client_id` | `String` | `null` | OAuth client ID issued by the SSO provider. *Migrated from deleted `sso_config.client_id`.* |
| `sso.client_secret` | `String` | `null` | OAuth client secret issued by the SSO provider. **Encrypted at rest; excluded from all audit log snapshots.** *Migrated from deleted `sso_config.client_secret`.* |
| `sso.tenant_id` | `String` | `null` | Azure / Microsoft tenant UUID; required when `provider = microsoft`. |
| `sso.metadata_url` | `String` | `null` | Provider metadata URL used for SAML federation setup. *Migrated from deleted `sso_config.metadata_url`.* |

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `group` | `Group` | Many-to-One | Parent org group | `ON DELETE SET NULL` — org continues without a group |
| `sector` | `Sector` | Many-to-One | Industry sector | `ON DELETE SET NULL` |
| `consultant` | `User` | Many-to-One | Assigned Bravo consultant | `ON DELETE SET NULL` |
| `account_manager` | `User` | Many-to-One | Assigned account manager | `ON DELETE SET NULL` |
| `memberships` | `UserOrgMembership` | One-to-Many | All user memberships in this org | `ON DELETE RESTRICT` — org cannot be deleted while memberships exist |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_orgs_group` | `group_uuid` | Index |
| `idx_orgs_status` | `status` | Index |
| *(standard)* | `uuid` | Unique |

---

### `users`

**Business Purpose:** Single identity record for every human who can log in — Bravo staff, org admins, or employees. PII is held here and subject to GDPR anonymisation on account closure (`gdpr_anonymised_at`). A user may belong to multiple organisations via `user_org_memberships` rows; this table holds only identity-level data that is consistent across all org contexts.

**Base Class:** `Base` + `SqlAlchemyEventMixin`

> ⚡ **Naming Note:** Original schema used `user_id VARCHAR(12)` as PK. Standardized to `id` (BigInteger PK) + `uuid` (String, unique) pattern. References to `user_id` in child tables are updated to `user_uuid`.

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `primary_email` | `String(255)` | Unique, NOT NULL | Primary login email address; globally unique across the platform — one account per email |
| `password_hash` | `Text` | Nullable | Bcrypt password hash; NULL for SSO-only users who authenticate entirely via their IdP |
| `first_name` | `String(100)` | NOT NULL, default=`""` | User's first name; replaced with `"Name Removed"` on GDPR anonymisation at account closure |
| `last_name` | `String(100)` | NOT NULL, default=`""` | User's last name; scrambled on GDPR anonymisation |
| `is_bravo_user` | `Boolean` | NOT NULL, default=False | Marks this account as a Bravo platform staff member; bypasses all `org_uuid` filter injection in `Base.get_base_query()` |
| `two_fa_enabled` | `Boolean` | NOT NULL, default=False | Whether the user has personally enabled 2FA via TOTP setup. This is the sole 2FA control — there is no org-level override; 2FA enforcement is a user-level decision only |
| `two_fa_secret` | `Text` | Nullable | Encrypted TOTP 2FA secret; excluded from all audit log snapshots |
| `gdpr_anonymised_at` | `DateTime (tz)` | Nullable | Timestamp when PII was anonymised in compliance with GDPR on org closure; signals downstream services to exclude this user from communications |
| `suppress_all_adverts` | `Boolean` | NOT NULL, default=False | User-level advert suppression override; when True, Bravo adverts are never shown to this user regardless of org settings |
| `is_active` | `Boolean` | NOT NULL, default=True | Global account active flag; False blocks all logins across every org membership |
| `last_login_at` | `DateTime (tz)` | Nullable | Timestamp of most recent successful login; used for inactive user reporting |

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `memberships` | `UserOrgMembership` | One-to-Many | All org memberships for this user | `ON DELETE RESTRICT` — user cannot be deleted while memberships exist |
| `oauth_accounts` | `OAuthAccount` | One-to-Many | Linked OAuth provider accounts | `ON DELETE CASCADE` — OAuth accounts are destroyed with the user |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_users_primary_email` | `primary_email` | Unique |
| `idx_users_active` | `is_active` | Index |
| *(Base)* | `uuid` | Unique |

---

### `user_org_memberships`

**Business Purpose:** The heart of role-based access in the platform. One row represents one user's relationship to one organisation — a user can hold multiple memberships across different orgs with different roles. Employment data that varies per-org context (salary, NI number, dates) lives here. Notification preferences and bookmarked benefits are consolidated as JSONB columns here to eliminate the previously separate `notification_preferences` and `employee_favourite_benefits` tables.

For inter-company moves, the old membership is deactivated (`is_active=False`, `deleted_at` set) and a new one is created; `source_membership_uuid` links the lineage.

**Base Class:** `Base` + `SqlAlchemyEventMixin`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `user_uuid` | `String` | FK → `users.uuid`, NOT NULL | The user holding this membership; links org-specific employment data back to the platform identity |
| `role` | `String(20)` | NOT NULL, default=`employee`, CHECK IN (`bravo_admin`, `bravo_staff`, `group_admin`, `org_admin`, `rnr_admin`, `employee`, `consultant`, `hr_team`, `freelancer`) | Role within this specific org context; controls which views, actions, and data the user can access |
| `is_primary_org` | `Boolean` | NOT NULL, default=True | Whether this is the user's primary organisation; determines which org is loaded on default login when the user belongs to multiple orgs |
| `ni_number` | `String(20)` | Nullable | UK National Insurance number; unique among active memberships per org when present — enforced via partial unique index |
| `employee_number` | `String(50)` | Nullable | Employer-assigned employee reference number; used for HR system correlation and bulk upload matching |
| `gross_annual_salary` | `Numeric(12,2)` | Nullable | Gross annual salary in GBP; feeds TRS salary component calculations |
| `pensionable_salary` | `Numeric(12,2)` | Nullable | Pensionable salary (may differ from gross); used specifically in TRS pension component calculations |
| `allowances` | `Numeric(12,2)` | Nullable | Annual allowances value (e.g. car allowance); available as a TRS component input |
| `bonus` | `Numeric(12,2)` | Nullable | Annual bonus value; available as a TRS component input |
| `date_of_birth` | `Date` | Nullable | Required when the org enables R&R birthday acknowledgement; never exposed directly in employee-facing APIs |
| `joining_date` | `Date` | Nullable | Required when the org enables R&R long-service acknowledgement; drives the anniversary scheduler sweep |
| `go_live_at` | `DateTime (tz)` | Nullable | Scheduled future onboarding datetime; when set, the background scheduler sends an invite email at this time |
| `invite_link_expires_at` | `DateTime (tz)` | Nullable | Invite link expiry — set to `go_live_at` if scheduled, otherwise `invited_at + 7 days` |
| `last_reminder_sent_at` | `DateTime (tz)` | Nullable | Timestamp of the last invite reminder email; prevents scheduler from spamming reminders |
| `has_logged_in` | `Boolean` | NOT NULL, default=False | Whether the user has ever logged in under this membership; used to distinguish truly inactive invited users from those who registered |
| `is_active` | `Boolean` | NOT NULL, default=True | Membership active flag; False means the user has left this org — historical data is retained |
| `deactivated_at` | `DateTime (tz)` | Nullable | Timestamp of deactivation — set on inter-company moves or departures |
| `source_membership_uuid` | `String` | FK → `user_org_memberships.uuid`, Nullable | UUID of the previous membership when a user moves between orgs; provides continuity lineage |
| `notification_preferences` | `JSONB` | Nullable, default=`{}` | Per-membership notification opt-in/opt-out toggles. Replaces the deleted `notification_preferences` table. See **`notification_preferences` JSONB Structure** below |
| `favourite_benefit_uuids` | `JSONB` | Nullable, default=`[]` | Ordered array of `org_benefit_uuid` strings representing the benefits this employee has pinned/bookmarked. Replaces the deleted `employee_favourite_benefits` join table. Order of UUIDs in the array determines display order on the employee homepage |

> **SQLAlchemy `__table_args__` Note:** The `role` column requires a `CheckConstraint`:
> ```python
> CheckConstraint(
>     "role IN ('bravo_admin','bravo_staff','group_admin','org_admin',"
>     "'rnr_admin','employee','consultant','hr_team','freelancer')",
>     name='ck_user_org_memberships_role'
> )
> ```

#### `notification_preferences` JSONB Structure

Stores the employee's opt-in preferences for each notification channel within this org membership context. Defaults to opt-in for transactional notifications (R&R) and explicit opt-in required for marketing-style notifications (GDPR).

```json
{
  "notify_documents":       false,
  "notify_notices":         false,
  "notify_messages":        false,
  "notify_rnr_nominations": true,
  "notify_rnr_wall":        true,
  "push_enabled":           true
}
```

| Key | Type | Default | Description |
|---|---|---|---|
| `notify_documents` | `Boolean` | `false` | Email notification when a new document is uploaded to the portal. GDPR: explicit opt-in required — default is False. *Migrated from deleted `notification_preferences.notify_documents`.* |
| `notify_notices` | `Boolean` | `false` | Email notification when a new notice is published. GDPR: explicit opt-in required — default is False. *Migrated from deleted `notification_preferences.notify_notices`.* |
| `notify_messages` | `Boolean` | `false` | Email notification when a new support ticket message is received. GDPR: explicit opt-in — default is False. *Migrated from deleted `notification_preferences.notify_messages`.* |
| `notify_rnr_nominations` | `Boolean` | `true` | Email notification when this user receives an R&R nomination. Transactional — default opt-in. *Migrated from deleted `notification_preferences.notify_rnr_nominations`.* |
| `notify_rnr_wall` | `Boolean` | `true` | Email notification when this user's nomination appears on the Wall of Fame. Transactional — default opt-in. *Migrated from deleted `notification_preferences.notify_rnr_wall`.* |
| `push_enabled` | `Boolean` | `true` | Mobile push notifications enabled for this membership context. *Migrated from deleted `notification_preferences.push_enabled`.* |

#### `favourite_benefit_uuids` JSONB Structure

```json
["<org_benefit_uuid_1>", "<org_benefit_uuid_2>", "..."]
```

| Key | Type | Description |
|---|---|---|
| *(array element)* | `String (UUID)` | UUID string referencing an `org_benefits.uuid` value. The order of elements in the array reflects the employee's preferred display order on the pinned benefits shelf. An empty array `[]` means no pinned benefits. *Migrated from deleted `employee_favourite_benefits` join table.* |

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `user` | `User` | Many-to-One | The platform identity record for this member | `ON DELETE RESTRICT` — membership blocks user deletion |
| `organisation` | `Organisation` | Many-to-One | The organisation this membership belongs to | `ON DELETE RESTRICT` — membership blocks org deletion |
| `org_teams` | `UserOrgTeam` | One-to-Many | Benefit-access team assignments for this membership | `ON DELETE CASCADE` — team slots removed with the membership |
| `source_membership` | `UserOrgMembership` | Self Many-to-One | Previous membership lineage on inter-company move | `ON DELETE SET NULL` — lineage reference cleared if source is purged |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_uom_ni_active` | `org_uuid`, `ni_number` WHERE `is_active=True AND ni_number IS NOT NULL` | Partial Unique — enforces NI uniqueness among active members per org |
| `idx_uom_user` | `user_uuid` | Index |
| `idx_uom_org` | `org_uuid` | Index |
| `idx_uom_role` | `org_uuid`, `role` | Index |
| `idx_uom_go_live` | `go_live_at` WHERE `go_live_at IS NOT NULL` | Partial Index — used by the onboarding scheduler sweep |

---

### `org_teams`

**Business Purpose:** Benefit-access teams within an organisation. Controls which employees can access which org benefits via `org_benefit_team_levels`. Entirely distinct from R&R teams — these teams exist purely to segment benefit visibility and TRS calculation tiers within a client org.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `team_name` | `String(150)` | NOT NULL | Human-facing display name for this benefit-access team (e.g. `Directors`, `Sales`, `All Staff`) |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether the team is currently operational; inactive teams are hidden from benefit assignment UIs but retained for historical TRS records |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_org_teams_org_name` | `org_uuid`, `team_name` | Unique — team names must be unique within an org |
| `idx_org_teams_org` | `org_uuid` | Index |

---

### `user_org_teams`

**Business Purpose:** Maps a user's org membership to one or more benefit-access teams. An employee can belong to multiple teams within the same org, receiving the union of all team-level benefit assignments. This is the mechanism by which TRS calculation levels are assigned per-employee.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | The org membership being assigned to a team; scopes the assignment to both user and org |
| `team_uuid` | `String` | FK → `org_teams.uuid`, NOT NULL | The benefit-access team the membership is being added to |
| `email_opt_in` | `Boolean` | NOT NULL, default=True | Whether the user opts in to email notifications specific to this team's benefit updates |

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `membership` | `UserOrgMembership` | Many-to-One | The user's org membership record | `ON DELETE CASCADE` — team slot removed when membership is deleted |
| `team` | `OrgTeam` | Many-to-One | The benefit-access team | `ON DELETE RESTRICT` — team cannot be deleted while members exist |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_user_org_teams_membership_team` | `membership_uuid`, `team_uuid` | Unique — prevents duplicate team assignments |
| `idx_user_org_teams_team` | `team_uuid` | Index |

---

### `org_internal_notes`

**Business Purpose:** Bravo-staff-only internal notes attached to an organisation record. Used for two purposes: tracking service interactions (service trail) and recording implementation milestones. Supports dated reminders that surface as a badge on the Bravo staff dashboard, with a dismissal flag to mark them as actioned.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `note_type` | `String(20)` | NOT NULL, default=`implementation`, CHECK IN (`service`, `implementation`) | Categorises the note: `service` = ongoing service trail entry; `implementation` = one-time setup milestone |
| `note_text` | `Text` | NOT NULL | Full text content of the note; supports plain text or Markdown |
| `reminder_date` | `Date` | Nullable | Optional future date when this note should resurface as a reminder badge on the Bravo staff dashboard |
| `reminder_dismissed` | `Boolean` | NOT NULL, default=False | Whether the reminder has been actioned and dismissed by a Bravo staff member; False = still active reminder |

> **SQLAlchemy `__table_args__` Note:** `note_type` requires:
> `CheckConstraint("note_type IN ('service', 'implementation')", name='ck_org_internal_notes_note_type')`

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_notes_org` | `org_uuid` | Index |
| `idx_notes_reminder` | `reminder_date` WHERE `reminder_date IS NOT NULL AND reminder_dismissed = False` | Partial Index — used by the Bravo staff dashboard reminder sweep |

---

### `image`

**Business Purpose:** Centralised image store for the platform. Provides a shared library of stock images (Bravo-managed, available to all orgs) and per-org bespoke uploads. Referenced by `master_benefits.image_uuid` and `custom_tiles.image_uuid` for benefit tile cover images. A hard cap of 100 images per org is enforced at the application layer to control storage costs.

**Base Class:** `Base`

> ⚡ **Naming Note:** Renamed from `image_library` to `image` across all table references, FK columns, and index names.

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid` (Nullable = Bravo stock image, visible to all orgs), `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `filename` | `String(255)` | NOT NULL | Original filename as provided during upload; used for display in the image picker |
| `s3_key` | `Text` | Unique, NOT NULL | Globally unique S3 object key; used to construct the CDN URL for rendering |
| `mime_type` | `String(50)` | NOT NULL | MIME type of the image (e.g. `image/jpeg`, `image/png`); validated on upload |
| `size_bytes` | `Integer` | NOT NULL | File size in bytes; used in the admin UI storage display and upload limit enforcement |
| `width_px` | `Integer` | Nullable | Image width in pixels; captured on upload for aspect ratio rendering hints |
| `height_px` | `Integer` | Nullable | Image height in pixels; captured on upload for aspect ratio rendering hints |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_image_s3_key` | `s3_key` | Unique |
| `idx_image_org` | `org_uuid` | Index |
| *(App-layer)* | Max 100 images per non-null `org_uuid` | Application-enforced cap — checked before every org upload |

---

### Domain 2 — Benefits & TRS

---

### `master_benefits`

**Business Purpose:** Global benefit catalogue managed exclusively by Bravo staff. Defines the master identity and default display configuration of every benefit available on the platform. Organisation-specific configurations extend this via `org_benefits`. All copy/display fields (descriptions, context variants, icon, resources, and token placeholders) are consolidated into JSONB columns to reduce table width and allow flexible content structure without schema migrations.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid` (NULL — platform-level record), `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `name` | `String(200)` | NOT NULL | Primary benefit display name shown as the tile heading throughout the platform |
| `benefit_type` | `String(20)` | NOT NULL, default=`standard`, CHECK IN (`standard`, `rnr`, `trs`, `other`) | Benefit category — `standard` = regular benefit offering; `rnr` = R&R-specific; `trs` = TRS component benefit; `other` = miscellaneous |
| `default_status` | `String(10)` | NOT NULL, default=`market`, CHECK IN (`offer`, `market`) | Default offering status applied when a benefit is first assigned to an org via `org_benefits` |
| `image_uuid` | `String` | FK → `image.uuid`, Nullable | UUID of the cover image from the `image` table; rendered as the benefit tile background (`ON DELETE SET NULL`) |
| `supports_arbitrary_levels` | `Boolean` | NOT NULL, default=False | When True, this benefit supports multiple TRS calculation levels via `benefit_levels`; required for any benefit used in TRS per PRD Module 6 |
| `hide_for_employee` | `Boolean` | NOT NULL, default=False | Master-level default for employer-only tiles (e.g. Employment Law benefit); when True, the benefit is hidden from employee views. Overridable at org level on `org_benefits` |
| `is_bravo_broker` | `Boolean` | NOT NULL, default=False | When True, Bravo acts as broker for this benefit — used for internal reporting on clients using Bravo vs alternative brokers |
| `default_employee_offer_type` | `String(30)` | Nullable, CHECK IN (`link`, `form`, `document`, `none`) | Default employee-facing interaction type when the benefit status is `offer`; controls which CTA renders on the employee tile |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether this benefit is available for assignment to organisations; inactive benefits are hidden from the org admin benefit picker |
| `data` | `JSONB` | Nullable, default=`{}` | Consolidated copy and display fields. Contains all rich text, context variant content, icon, and provider information. See **`data` JSONB Structure** below |
| `resources` | `JSONB` | Nullable, default=`[]` | Ordered array of resource objects (videos, PDFs, links) attached to this benefit for employee reference. Replaces the deleted `benefit_resources` table. See **`resources` JSONB Structure** below |
| `required_tokens` | `JSONB` | Nullable, default=`[]` | Array of token/substitution variable definitions for this benefit (e.g. `{{SCHEME_URL}}`). Org-specific answers are stored in `org_benefits.token_answers`. Replaces the deleted `benefit_tokens` table. See **`required_tokens` JSONB Structure** below |

> **SQLAlchemy `__table_args__` Note:** The following ENUM columns require `CheckConstraint` entries:
> - `benefit_type`: `CheckConstraint("benefit_type IN ('standard', 'rnr', 'trs', 'other')", name='ck_master_benefits_benefit_type')`
> - `default_status`: `CheckConstraint("default_status IN ('offer', 'market')", name='ck_master_benefits_default_status')`
> - `default_employee_offer_type`: `CheckConstraint("default_employee_offer_type IN ('link', 'form', 'document', 'none')", name='ck_master_benefits_default_employee_offer_type')`

#### `data` JSONB Structure

Contains all copy, display, and context-variant fields for this benefit. Context variants follow a 2×2 matrix: audience (`company` vs `employee`) × status (`market` vs `offer`).

```json
{
  "short_desc":        "<string|null>",
  "description":       "<string|null>",
  "email_snippet":     "<string|null>",
  "partner_provider":  "<string|null>",
  "font_awesome_icon": "<string|null>",

  "ctx_company_market_summary":   "<string|null>",
  "ctx_company_market_desc":      "<string|null>",
  "ctx_company_market_video_url": "<string|null>",

  "ctx_company_offer_summary":    "<string|null>",
  "ctx_company_offer_desc":       "<string|null>",
  "ctx_company_offer_video_url":  "<string|null>",

  "ctx_employee_market_summary":   "<string|null>",
  "ctx_employee_market_desc":      "<string|null>",
  "ctx_employee_market_video_url": "<string|null>",

  "ctx_employee_offer_summary":    "<string|null>",
  "ctx_employee_offer_desc":       "<string|null>",
  "ctx_employee_offer_video_url":  "<string|null>"
}
```

| Key | Type | Description |
|---|---|---|
| `short_desc` | `String` | Short description (up to ~500 chars) shown on listing/card views before a user opens the full benefit page. *Migrated from removed top-level `short_desc` column.* |
| `description` | `Text` | Full benefit description rendered as block-builder HTML on the benefit detail page. *Migrated from removed `description`.* |
| `email_snippet` | `Text` | HTML snippet inserted into email templates when this benefit is referenced in comms. *Migrated from removed `email_snippet`.* |
| `partner_provider` | `String` | Name of the third-party provider who delivers this benefit (e.g. `Vitality`, `Cycle Solutions`). *Migrated from removed `partner_provider`.* |
| `font_awesome_icon` | `String` | Font Awesome icon class used as fallback tile icon when no `image_uuid` is set (e.g. `fa-heartbeat`). *Migrated from removed `font_awesome_icon`.* |
| `ctx_company_market_summary` | `Text` | Summary shown to org admin when marketing this benefit to the company (status = `market`, audience = employer). *Migrated from removed `ctx_company_market_summary`.* |
| `ctx_company_market_desc` | `Text` | Full description for the company-market context. *Migrated from removed `ctx_company_market_desc`.* |
| `ctx_company_market_video_url` | `String` | Optional video URL for the company-market context. *Migrated from removed `ctx_company_market_video_url`.* |
| `ctx_company_offer_summary` | `Text` | Summary shown to org admin once the company has taken up the benefit (status = `offer`, audience = employer). *Migrated from removed `ctx_company_offer_summary`.* |
| `ctx_company_offer_desc` | `Text` | Full description for the company-offer context. *Migrated from removed `ctx_company_offer_desc`.* |
| `ctx_company_offer_video_url` | `String` | Optional video URL for the company-offer context. *Migrated from removed `ctx_company_offer_video_url`.* |
| `ctx_employee_market_summary` | `Text` | Summary shown to employee when the benefit is being marketed (status = `market`, audience = employee). *Migrated from removed `ctx_employee_market_summary`.* |
| `ctx_employee_market_desc` | `Text` | Full description for the employee-market context. *Migrated from removed `ctx_employee_market_desc`.* |
| `ctx_employee_market_video_url` | `String` | Optional video URL for the employee-market context. *Migrated from removed `ctx_employee_market_video_url`.* |
| `ctx_employee_offer_summary` | `Text` | Summary shown to employee once the benefit is active (status = `offer`, audience = employee). *Migrated from removed `ctx_employee_offer_summary`.* |
| `ctx_employee_offer_desc` | `Text` | Full description for the employee-offer context. *Migrated from removed `ctx_employee_offer_desc`.* |
| `ctx_employee_offer_video_url` | `String` | Optional video URL for the employee-offer context. *Migrated from removed `ctx_employee_offer_video_url`.* |

#### `resources` JSONB Structure

An ordered array of supplementary resource objects displayed on the benefit detail page.

```json
[
  {
    "resource_type": "video|pdf|link",
    "title":         "<string>",
    "resource_url":  "<string>",
    "display_order": 0
  }
]
```

| Key | Type | Description |
|---|---|---|
| `resource_type` | `String` | Format of the resource. Allowed values: `video`, `pdf`, `link`. *Migrated from deleted `benefit_resources.resource_type`.* |
| `title` | `String` | Human-facing title displayed as the resource link label on the benefit page. *Migrated from deleted `benefit_resources.title`.* |
| `resource_url` | `String` | Full URL to the resource — external video, S3 PDF, or external link. *Migrated from deleted `benefit_resources.resource_url`.* |
| `display_order` | `Integer` | Sort order within the resources section; lower = displayed first. *Migrated from deleted `benefit_resources.display_order`.* |

#### `required_tokens` JSONB Structure

An array of substitution token definitions. When an org admin configures an `org_benefits` row, they are prompted to provide answers for each token defined here; answers are stored in `org_benefits.token_answers`.

```json
[
  {
    "token_key":            "{{SCHEME_URL}}",
    "token_label":          "Scheme URL",
    "usage_type":           "page|pdf|email|internal",
    "is_required":          true,
    "is_bravo_broker_flag": false
  }
]
```

| Key | Type | Description |
|---|---|---|
| `token_key` | `String` | The substitution placeholder string (e.g. `{{SCHEME_URL}}`); used in content templates for find-and-replace. *Migrated from deleted `benefit_tokens.token_key`.* |
| `token_label` | `String` | Human-readable label shown to org admins in the token answer form. *Migrated from deleted `benefit_tokens.token_label`.* |
| `usage_type` | `String` | Where this token is used. Allowed values: `internal`, `page`, `pdf`, `email`. *Migrated from deleted `benefit_tokens.usage_type`.* |
| `is_required` | `Boolean` | When True, the org must supply an answer before the benefit can be set to `offer` status. *Migrated from deleted `benefit_tokens.is_required`.* |
| `is_bravo_broker_flag` | `Boolean` | When True, this token only applies to orgs where `settings.features.is_bravo_broker = true`; hidden for non-broker orgs. *Migrated from deleted `benefit_tokens.is_bravo_broker_flag`.* |

---

### `benefit_levels`

**Business Purpose:** Named calculation-tier definitions for a master benefit (e.g. `Director`, `Employee`, `Senior Manager`). These levels drive the TRS calculation hierarchy: master benefit level → org benefit override → team grid assignment (`org_benefit_team_levels`). Each level defines a calculation method and value; the actual team-to-level mapping lives in `org_benefit_team_levels`.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid` (NULL — platform-level), `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `benefit_uuid` | `String` | FK → `master_benefits.uuid`, NOT NULL | The parent master benefit this level belongs to |
| `level_name` | `String(100)` | NOT NULL | Display name of the level shown to org admins in the TRS configuration UI (e.g. `Director`, `Employee`) |
| `display_order` | `Integer` | NOT NULL, default=0 | Sort order within the benefit's level list; lower = shown first |
| `calc_method` | `String(30)` | NOT NULL, default=`not_set`, CHECK IN (`salary_multiple`, `salary_percentage`, `fixed_amount`, `rate_per_thousand`, `individual`, `not_set`) | Calculation method applied at this tier: `salary_multiple` = gross × value; `salary_percentage` = gross × value/100; `fixed_amount` = value directly; `rate_per_thousand` = (gross / 1000) × value; `individual` = per-employee manual entry; `not_set` = not yet configured |
| `calc_value` | `Numeric(12,4)` | Nullable | The multiplier, percentage, rate, or fixed amount used by `calc_method`; NULL when `calc_method = not_set` |

> **SQLAlchemy `__table_args__` Note:** `calc_method` requires:
> `CheckConstraint("calc_method IN ('salary_multiple','salary_percentage','fixed_amount','rate_per_thousand','individual','not_set')", name='ck_benefit_levels_calc_method')`

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `benefit` | `MasterBenefit` | Many-to-One | Parent master benefit | `ON DELETE CASCADE` — levels are intrinsic to the benefit definition and removed if the master benefit is deleted |

---

### `org_benefits`

**Business Purpose:** A master benefit configured and activated for a specific organisation. The `status` column drives employee-facing visibility. Org-specific copywritten content (4 context variants), the bespoke attachment URL, and token substitution answers are consolidated into two JSONB columns (`data` and `token_answers`) to eliminate the previously separate `org_benefit_token_answers` table and reduce top-level column count.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `benefit_uuid` | `String` | FK → `master_benefits.uuid`, NOT NULL | The master benefit definition this org is configuring |
| `status` | `String(10)` | NOT NULL, default=`market`, CHECK IN (`offer`, `market`, `hidden`, `refused`, `draft`) | Employee-facing display status: `offer` = benefit is available; `market` = being marketed, not yet active; `hidden` = hidden from all views; `refused` = org declined; `draft` = being configured |
| `hide_for_employee` | `Boolean` | NOT NULL, default=False | Org-level override — when True, the benefit is visible to org admins only, hidden from all employee views regardless of `status` |
| `is_featured` | `Boolean` | NOT NULL, default=False | When True, this benefit is promoted to the featured section on the employee benefits homepage |
| `display_order` | `Integer` | NOT NULL, default=0 | Sort order on the benefits listing page; lower = appears first |
| `go_live_at` | `DateTime (tz)` | Nullable | Scheduled datetime (UK timezone) for the background scheduler to automatically flip `status` from `market` → `offer` |
| `data` | `JSONB` | Nullable, default=`{}` | Org-specific copywritten content overrides and attachment URL. See **`data` JSONB Structure** below |
| `token_answers` | `JSONB` | Nullable, default=`{}` | Key-value map of token substitution answers provided by the org admin. Replaces the deleted `org_benefit_token_answers` table. See **`token_answers` JSONB Structure** below |

> **SQLAlchemy `__table_args__` Note:** `status` requires:
> `CheckConstraint("status IN ('offer', 'market', 'hidden', 'refused', 'draft')", name='ck_org_benefits_status')`

#### `data` JSONB Structure

Stores org-level content overrides for the 4-context display matrix, plus the bespoke attachment URL. When a key is present and non-null, it overrides the corresponding value from `master_benefits.data`. When null/absent, the master benefit's value is used as the fallback.

```json
{
  "ctx_market_company":   "<string|null>",
  "ctx_offer_company":    "<string|null>",
  "ctx_market_employee":  "<string|null>",
  "ctx_offer_employee":   "<string|null>",
  "bespoke_attachment_url": "<string|null>"
}
```

| Key | Type | Description |
|---|---|---|
| `ctx_market_company` | `Text` | Org-specific copywritten content for the employer view when benefit status is `market`. Overrides `master_benefits.data.ctx_company_market_summary` when present. *Migrated from removed top-level `ctx_market_company` column.* |
| `ctx_offer_company` | `Text` | Org-specific copywritten content for the employer view when benefit status is `offer`. *Migrated from removed `ctx_offer_company`.* |
| `ctx_market_employee` | `Text` | Org-specific copywritten content for the employee view when benefit status is `market`. *Migrated from removed `ctx_market_employee`.* |
| `ctx_offer_employee` | `Text` | Org-specific copywritten content for the employee view when benefit status is `offer`. *Migrated from removed `ctx_offer_employee`.* |
| `bespoke_attachment_url` | `String` | URL of an org-specific document attachment (e.g. their own scheme brochure) shown only to this org's employees. *Migrated from removed `bespoke_attachment_url`.* |

#### `token_answers` JSONB Structure

A flat key-value map where each key is a `token_key` string from `master_benefits.required_tokens` and the value is the org's substitution answer.

```json
{
  "{{SCHEME_URL}}":  "https://orgname.schemeprovider.co.uk",
  "{{UNIQUE_CODE}}": "ORG123XYZ"
}
```

| Key | Type | Description |
|---|---|---|
| `<token_key>` (e.g. `{{SCHEME_URL}}`) | `String` | The substitution token placeholder from `master_benefits.required_tokens[*].token_key` as the map key; the org's answer string as the value. Used by the content rendering engine to substitute placeholders in `data` fields and email templates. *Replaces the deleted `org_benefit_token_answers` table.* |

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `benefit` | `MasterBenefit` | Many-to-One | The parent master benefit definition | `ON DELETE RESTRICT` — master benefit cannot be deleted while org configs exist |
| `team_levels` | `OrgBenefitTeamLevel` | One-to-Many | Team-specific benefit level assignments | `ON DELETE CASCADE` — level assignments removed with the org benefit |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_org_benefits_org_benefit` | `org_uuid`, `benefit_uuid` | Unique — one offering per benefit per org |
| `idx_org_benefits_org` | `org_uuid` | Index |
| `idx_org_benefits_status` | `status` | Index |
| `idx_org_benefits_golive` | `go_live_at` WHERE `go_live_at IS NOT NULL` | Partial Index — used by the go-live scheduler sweep |

---

### `org_benefit_team_levels`

**Business Purpose:** Tier 3 of the TRS calculation hierarchy — assigns a specific `benefit_level` to a team within an org benefit configuration. This is the junction between org teams and benefit levels, determining which TRS calculation rate applies to employees in a given team for a given benefit.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `org_benefit_uuid` | `String` | FK → `org_benefits.uuid`, NOT NULL | The org benefit configuration being team-tiered |
| `team_uuid` | `String` | FK → `org_teams.uuid`, NOT NULL | The team receiving this level assignment |
| `level_uuid` | `String` | FK → `benefit_levels.uuid`, NOT NULL | The benefit level (and its calc method/value) assigned to this team |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_org_benefit_team_levels` | `org_benefit_uuid`, `team_uuid` | Unique — one level per team per org benefit |

---

### `custom_tiles`

**Business Purpose:** Org-created benefit tiles for non-Bravo offerings — competitions, TypeForm surveys, seasonal promotions, or any external content the org wants to surface to employees. Renamed from `custom_benefits` to better reflect that these are UI tile entries, not benefit catalogue items. The `visible_to_team_uuids` JSONB array replaces the deleted `custom_benefit_team_access` many-to-many join table, simplifying team visibility management into a single column.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `group_uuid` | `String` | FK → `groups.uuid`, Nullable | When set, this tile was pushed by a group admin and is visible to all orgs in the group (not just the creating org) |
| `title` | `String(255)` | NOT NULL | Display title of the tile shown on the employee benefits/homepage view |
| `short_desc` | `String(500)` | Nullable | Short description shown beneath the tile title in listing views |
| `description` | `Text` | Nullable | Full content description (block-builder HTML) shown on the tile detail page |
| `image_uuid` | `String` | FK → `image.uuid`, Nullable | Cover image UUID from the `image` table used as the tile background (`ON DELETE SET NULL`) |
| `embed_type` | `String(20)` | NOT NULL, default=`external_url`, CHECK IN (`external_url`, `html_embed`) | Controls how the tile content is delivered: `external_url` = redirect or new tab; `html_embed` = iFrame/inline HTML |
| `embed_content` | `Text` | Nullable | The URL or HTML snippet depending on `embed_type` |
| `open_in_new_tab` | `Boolean` | NOT NULL, default=True | When True and `embed_type = external_url`, the link opens in a new browser tab |
| `is_featured` | `Boolean` | NOT NULL, default=False | When True, this tile is promoted to the featured section on the employee homepage |
| `display_order` | `Integer` | NOT NULL, default=0 | Sort order on the benefits/tiles listing page; lower = appears first |
| `status` | `String(10)` | NOT NULL, default=`active`, CHECK IN (`active`, `disabled`, `deleted`) | Lifecycle status: `active` = visible; `disabled` = hidden (e.g. seasonal promotion off-season); `deleted` = soft-deleted |
| `visible_to_team_uuids` | `JSONB` | Nullable, default=`[]` | Array of `org_teams.uuid` strings restricting visibility to specific teams. When the array is empty (`[]`) or null, the tile is visible to **all teams** in the org. Replaces the deleted `custom_benefit_team_access` many-to-many join table. See **`visible_to_team_uuids` JSONB Structure** below |

> **SQLAlchemy `__table_args__` Note:** The following ENUM columns require `CheckConstraint` entries:
> - `embed_type`: `CheckConstraint("embed_type IN ('external_url', 'html_embed')", name='ck_custom_tiles_embed_type')`
> - `status`: `CheckConstraint("status IN ('active', 'disabled', 'deleted')", name='ck_custom_tiles_status')`

#### `visible_to_team_uuids` JSONB Structure

```json
["<org_team_uuid_1>", "<org_team_uuid_2>"]
```

| Key | Type | Description |
|---|---|---|
| *(array element)* | `String (UUID)` | UUID string referencing an `org_teams.uuid` value. When the array is populated, only employees who are members of one of these teams (via `user_org_teams`) will see this tile. An empty array or null means all teams in the org can see it. *Replaces the deleted `custom_benefit_team_access` join table.* |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_custom_tiles_org` | `org_uuid` | Index |
| `idx_custom_tiles_group` | `group_uuid` | Index — used when resolving group-pushed tiles across member orgs |

---

### `benefit_requests`

**Business Purpose:** Created when an employee or org admin clicks "I want this" on a marketed benefit tile. Acts as a lead-capture mechanism, surfacing demand signals on the Bravo staff dashboard as actionable items so consultants can follow up and progress the benefit to `offer` status.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `org_benefit_uuid` | `String` | FK → `org_benefits.uuid`, NOT NULL | The org benefit being requested; the benefit must have `status = market` at time of request |
| `requested_by_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | Membership UUID of the user who clicked "I want this"; links request to both user and org |
| `status` | `String(20)` | NOT NULL, default=`pending`, CHECK IN (`pending`, `actioned`, `removed`) | Handling status: `pending` = visible on Bravo dashboard; `actioned` = consultant followed up; `removed` = dismissed without action |
| `actioned_at` | `DateTime (tz)` | Nullable | Timestamp when a Bravo staff member actioned this request |
| `actioned_by_uuid` | `String` | FK → `users.uuid`, Nullable | UUID of the Bravo staff user who actioned the request |

> **SQLAlchemy `__table_args__` Note:** `status` requires:
> `CheckConstraint("status IN ('pending', 'actioned', 'removed')", name='ck_benefit_requests_status')`

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_benefit_requests_pending` | `status` WHERE `status = 'pending'` | Partial Index — drives the Bravo dashboard pending requests badge |

---

### `employee_benefit_status`

**Business Purpose:** Tracks per-employee enrolment in a specific benefit (e.g. cycle-to-work scheme enrolment). One record per membership per org benefit. Used by org admins and Bravo staff to manage and report on who is enrolled in each offered benefit.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | The employee org membership being tracked |
| `org_benefit_uuid` | `String` | FK → `org_benefits.uuid`, NOT NULL | The benefit whose enrolment is being tracked |
| `status` | `String(20)` | NOT NULL, default=`enrolled`, CHECK IN (`enrolled`, `unenrolled`) | Current enrolment state |
| `enrolled_at` | `DateTime (tz)` | NOT NULL, default=now | Timestamp when the employee was enrolled |
| `unenrolled_at` | `DateTime (tz)` | Nullable | Timestamp when the employee was unenrolled; NULL if still enrolled |
| `notes` | `Text` | Nullable | Admin notes on the enrolment or unenrolment reason (e.g. "left cycle-to-work scheme mid-year") |

> **SQLAlchemy `__table_args__` Note:** `status` requires:
> `CheckConstraint("status IN ('enrolled', 'unenrolled')", name='ck_employee_benefit_status_status')`

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_employee_benefit_status` | `membership_uuid`, `org_benefit_uuid` | Unique — one enrolment record per employee per benefit |

---

### `trs_config`

**Business Purpose:** Total Reward Statement configuration for an organisation. One record per org when the TRS feature is enabled (controlled by `organisations.settings.features.is_trs_enabled`). Defines the presentation settings for the TRS module — chart style, salary visibility toggle permission, and statement copy.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `is_enabled` | `Boolean` | NOT NULL, default=False | Whether the TRS module is actively enabled for this org; a secondary flag under the master `organisations.settings.features.is_trs_enabled` |
| `allow_salary_visibility` | `Boolean` | NOT NULL, default=True | When True, employees are given a UI toggle to show/hide their salary on their own TRS PDF |
| `chart_type` | `String(5)` | NOT NULL, default=`pie`, CHECK IN (`pie`, `bar`) | Chart visualisation type used for the TRS component breakdown on the employee view |
| `intro_text` | `Text` | Nullable | Introductory copy displayed above the component breakdown on the employee TRS page |
| `statement_title` | `String(255)` | Nullable | Custom title displayed at the top of the generated TRS PDF (e.g. `"Your Total Reward Statement 2026"`) |

> **SQLAlchemy `__table_args__` Note:** `chart_type` requires:
> `CheckConstraint("chart_type IN ('pie', 'bar')", name='ck_trs_config_chart_type')`

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_trs_config_org_uuid` | `org_uuid` | Unique — one TRS config per org |

---

### `trs_component_types`

**Business Purpose:** System-defined TRS component taxonomy — the master list of component categories (salary, pension, bonus, etc.) that Bravo staff define at platform level. Each type defines a default calculation method. Org-level configurations (`trs_components`) reference these types. Managed exclusively by Bravo staff — not editable by client orgs.

**Base Class:** `db.Model` + `UserAuditMixin` (platform lookup — no org scope)

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Internal surrogate primary key |
| `uuid` | `String` | Unique | Public UUID referenced as `component_type_uuid` in `trs_components` |
| `type_name` | `String(30)` | Unique, NOT NULL, CHECK IN (`salary`, `pension`, `bonus`, `benefit_value`, `allowance`, `holiday_value`, `insurance`, `other`) | System identifier for the component type; unique across the platform |
| `display_label` | `String(150)` | NOT NULL | Human-readable default label (e.g. `"Base Salary"`) shown on TRS if the org does not provide an override |
| `calc_method` | `String(30)` | NOT NULL, default=`not_set`, CHECK IN (`salary_multiple`, `salary_percentage`, `fixed_amount`, `rate_per_thousand`, `individual`, `not_set`) | Default calculation method for this component type; can be overridden at org level in `trs_components` |
| `is_system_defined` | `Boolean` | NOT NULL, default=True | Whether this type was created by Bravo (True) or is a custom addition (False); system-defined types cannot be deleted |
| `color_hex` | `String(7)` | Nullable | Default display colour used for this component's segment in TRS pie/bar charts (e.g. `#4A90E2`) |

> **SQLAlchemy `__table_args__` Note:** The following ENUM columns require `CheckConstraint` entries:
> - `type_name`: `CheckConstraint("type_name IN ('salary','pension','bonus','benefit_value','allowance','holiday_value','insurance','other')", name='ck_trs_component_types_type_name')`
> - `calc_method`: `CheckConstraint("calc_method IN ('salary_multiple','salary_percentage','fixed_amount','rate_per_thousand','individual','not_set')", name='ck_trs_component_types_calc_method')`

---

### `trs_components`

**Business Purpose:** Per-org TRS component configuration — specifies which system component types are active for this org, how they are labelled, and what calculation parameters apply. Acts as the org-level customisation layer over the platform-defined `trs_component_types`.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `component_type_uuid` | `String` | FK → `trs_component_types.uuid`, NOT NULL | The system component type this org config is based on |
| `display_name` | `String(200)` | Nullable | Org-specific label override (e.g. `"Pension (employer contribution)"`); falls back to `trs_component_types.display_label` when null |
| `calc_value` | `Numeric(14,4)` | Nullable | The multiplier, percentage, rate, or fixed amount for this org's component calculation; NULL if the component type's default method applies |
| `display_order` | `Integer` | NOT NULL, default=0 | Sort order for this component in the TRS employee view and PDF |
| `is_visible_to_employee` | `Boolean` | NOT NULL, default=True | When False, this component is included in the total calculation but hidden from the employee-facing TRS view |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether this component is currently active for this org; inactive components are excluded from TRS calculations and the employee view |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_trs_components_org_type` | `org_uuid`, `component_type_uuid` | Unique — one config per component type per org |
| `idx_trs_components_org` | `org_uuid` | Index |

---

### `trs_employee_data`

**Business Purpose:** Per-employee values for each active TRS component, keyed to `membership_uuid`. One record per employee per active component. These values feed directly into TRS statement generation and PDF rendering.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | The employee org membership this value belongs to |
| `component_uuid` | `String` | FK → `trs_components.uuid`, NOT NULL | The org TRS component this value corresponds to |
| `value_amount` | `Numeric(14,2)` | NOT NULL | The monetary value for this component for this employee (in GBP) |
| `effective_date` | `Date` | Nullable | Date from which this value takes effect; used for point-in-time statement accuracy |
| `entered_by_uuid` | `String` | FK → `users.uuid`, Nullable | UUID of the org admin or Bravo staff user who entered or last updated this value |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_trs_employee_data` | `membership_uuid`, `component_uuid` | Unique — one value per employee per component |

---

### `trs_statements`

**Business Purpose:** Point-in-time PDF snapshot of a user's Total Reward Statement. Stores the full component breakdown as JSONB (`snapshot_json`) so that historical statements can be retrieved and re-rendered instantly without recalculation from live data. Generated on demand or scheduled annually.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | The employee this statement was generated for |
| `generated_at` | `DateTime (tz)` | NOT NULL, default=now | Timestamp when the statement was generated; used as the statement date displayed on the PDF |
| `snapshot_json` | `JSONB` | NOT NULL | Full point-in-time component breakdown captured at generation time. See **`snapshot_json` JSONB Structure** below |
| `total_value` | `Numeric(14,2)` | NOT NULL | Total reward value at the time of generation; stored as a scalar for fast listing/sorting without parsing the JSON |
| `salary_shown` | `Boolean` | NOT NULL, default=True | Whether the employee chose to display their salary on this statement at the time it was generated |

#### `snapshot_json` JSONB Structure

Captures the complete state of the TRS at generation time so that historical statements are immutable and reproducible. All values are stored as-calculated — do not re-derive from live `trs_employee_data` or `trs_components` rows.

```json
{
  "statement_title": "Your Total Reward Statement 2026",
  "generated_for": {
    "membership_uuid": "<string>",
    "employee_name":   "<string>",
    "employee_number": "<string|null>"
  },
  "total_value": 65000.00,
  "salary_visible": true,
  "components": [
    {
      "component_type": "salary",
      "display_name":   "Base Salary",
      "calc_method":    "individual",
      "value_amount":   45000.00,
      "color_hex":      "#4A90E2",
      "is_visible":     true
    }
  ]
}
```

| Key | Type | Description |
|---|---|---|
| `statement_title` | `String` | The statement title at generation time, copied from `trs_config.statement_title`; stored here so historical PDFs are not affected by future config changes |
| `generated_for.membership_uuid` | `String` | UUID of the membership this statement was generated for — audit reference |
| `generated_for.employee_name` | `String` | Full name of the employee at generation time; stored here to survive GDPR anonymisation of the live `users` record |
| `generated_for.employee_number` | `String\|null` | Employer-assigned employee number at generation time, from `user_org_memberships.employee_number` |
| `total_value` | `Numeric` | Total reward value summed across all visible components at generation time; mirrors the top-level `trs_statements.total_value` scalar |
| `salary_visible` | `Boolean` | Whether the employee chose to display their salary on this statement; mirrors `trs_statements.salary_shown` |
| `components[*].component_type` | `String` | The `type_name` from `trs_component_types` at generation time (e.g. `salary`, `pension`) |
| `components[*].display_name` | `String` | The org's display name for this component at generation time, from `trs_components.display_name` or `trs_component_types.display_label` |
| `components[*].calc_method` | `String` | The calculation method applied at generation time |
| `components[*].value_amount` | `Numeric` | The calculated monetary value for this component for this employee |
| `components[*].color_hex` | `String\|null` | The chart colour for this component at generation time, from `trs_component_types.color_hex` |
| `components[*].is_visible` | `Boolean` | Whether this component was visible to the employee at generation time, from `trs_components.is_visible_to_employee` |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_trs_statements_membership` | `membership_uuid` | Index — used when retrieving all statements for an employee |

---

### Domain 3 — Reward & Recognition

---

### `rnr_config`

**Business Purpose:** R&R module configuration for an organisation — one record per org when the R&R feature is enabled (controlled by `organisations.settings.features.is_rnr_enabled`). Defines budget mode, branding, behavioural flags, and the org's perk catalogue. The `predefined_perks` JSONB array replaces the deleted `rnr_perks` standalone table; the `rules_data` JSONB supplements (but does not replace) the flat control columns.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `rnr_title` | `String(200)` | NOT NULL, default=`Reward & Recognition` | Custom display name for the R&R module shown throughout the employee portal |
| `wall_name` | `String(200)` | NOT NULL, default=`Wall of Fame` | Custom name for the Wall of Fame tab visible to employees |
| `email_header_image_uuid` | `String` | FK → `image.uuid`, Nullable | UUID of the header image used in R&R email communications (`ON DELETE SET NULL`) |
| `logo_url` | `Text` | Nullable | S3 URL of the R&R module logo shown at the top of the R&R section |
| `color_main` | `String(7)` | Nullable | Primary R&R brand colour in hex format (e.g. `#FF5733`) |
| `color_secondary` | `String(7)` | Nullable | Secondary R&R brand colour in hex format; used for accents within the R&R module |
| `reward_pot_mode` | `String(10)` | Nullable, CHECK IN (`per_user`, `per_team`) | Budget allocation model — mutually exclusive; `per_user` assigns budgets to individual R&R admins; `per_team` pools budget at the R&R team level. Cannot mix within an org |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether the R&R module is currently active for this org; False suppresses all R&R routes for the org's users |
| `predefined_perks` | `JSONB` | Nullable | The org's catalogue of redeemable predefined perks. Replaces the deleted `rnr_perks` table. See **`predefined_perks` JSONB Structure** below |
| `rules_data` | `JSONB` | Nullable | Supplementary R&R behavioural rules. Supplements (does NOT replace) flat control columns — provides a grouped view for UI serialisation and future extensibility. See **`rules_data` JSONB Structure** below |

> **SQLAlchemy `__table_args__` Note:** `reward_pot_mode` requires:
> `CheckConstraint("reward_pot_mode IN ('per_user', 'per_team')", name='ck_rnr_config_reward_pot_mode')`

#### `predefined_perks` JSONB Structure

An array of predefined perk objects representing the org's available non-monetary reward catalogue. Replaces the deleted `rnr_perks` standalone table. When an employee is awarded a predefined perk, the name and emoji are captured as an immutable snapshot on `rnr_rewards.predefined_perk_name` / `predefined_perk_emoji` so that historical reward records remain accurate even if the org later edits or removes a perk from this catalogue.

```json
[
  {
    "name":      "Extra Day Off",
    "emoji":     "🏖️",
    "is_active": true
  }
]
```

| Key | Type | Description |
|---|---|---|
| `name` | `String` | Display name of the perk shown in the reward selection UI (e.g. `"Extra Day Off"`, `"Team Lunch"`). *Migrated from deleted `rnr_perks.perk_name`.* |
| `emoji` | `String` | Optional emoji icon displayed alongside the perk name (e.g. `"🏖️"`). *Migrated from deleted `rnr_perks.perk_emoji`.* |
| `is_active` | `Boolean` | When False, the perk is hidden from the reward selection UI but retained in the array so that existing historical snapshots on `rnr_rewards` remain unaffected. *Migrated from deleted `rnr_perks.is_active`.* |

#### `rules_data` JSONB Structure

Supplements the flat behavioural columns — does **not** replace them. The flat columns remain the authoritative source; `rules_data` provides a grouped representation for UI serialisation. When reading rules, always prefer the flat column over the JSONB key; the JSONB is kept in sync as a convenience projection.

```json
{
  "approve_before_publish":    false,
  "allow_money_rewards":       false,
  "allow_perk_rewards":        false,
  "acknowledge_birthdays":     false,
  "acknowledge_long_service":  false
}
```

| Key | Flat Column | Type | Default | Description |
|---|---|---|---|---|
| `approve_before_publish` | `approve_before_publish` *(flat)* | `Boolean` | `false` | When True, nominations require admin approval before appearing on the Wall of Fame. *Supplemented into JSONB for UI grouping.* |
| `allow_money_rewards` | `allow_money_rewards` *(flat)* | `Boolean` | `false` | Whether monetary rewards can be given within this org. *Supplemented into JSONB for UI grouping.* |
| `allow_perk_rewards` | `allow_perk_rewards` *(flat)* | `Boolean` | `false` | Whether predefined or custom perk rewards can be given. *Supplemented into JSONB for UI grouping.* |
| `acknowledge_birthdays` | `acknowledge_birthdays` *(flat)* | `Boolean` | `false` | Drives the birthday scheduler sweep that auto-generates nominations for `user_org_memberships.date_of_birth` matches. *Supplemented into JSONB for UI grouping.* |
| `acknowledge_long_service` | `acknowledge_long_service` *(flat)* | `Boolean` | `false` | Drives the long-service scheduler sweep based on `user_org_memberships.joining_date` anniversaries. *Supplemented into JSONB for UI grouping.* |

> **Implementation Note:** The five keys in `rules_data` mirror the five flat Boolean columns that are retained on this table. When updating rules via the API, write to both the flat column and the JSONB key atomically in a single `UPDATE`. The flat columns are the indexed, query-authoritative values; `rules_data` is a convenience grouping for the frontend serialiser.

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_rnr_config_org_uuid` | `org_uuid` | Unique — one R&R config per org |

---

### `rnr_teams`

**Business Purpose:** R&R-specific teams within an organisation. **Entirely separate from `org_teams`** (benefit-access teams) — distinct memberships, budget ledgers, and admin roles. R&R teams exist solely to pool reward budgets and scope nomination visibility in the R&R module; they have no effect on benefit access or TRS calculations. Cannot be deleted if financial budget rows exist; use `is_active=False` to retire a team while preserving its ledger history.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `team_name` | `String(150)` | NOT NULL | R&R team display name shown in the admin UI and on the Wall of Fame filter |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether the team is active; set False instead of deleting if budget ledger rows (`rnr_budget_team`) exist to preserve financial history |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_rnr_teams_org_name` | `org_uuid`, `team_name` | Unique — team names must be unique within an org |
| `idx_rnr_teams_org` | `org_uuid` | Index |

---

### `rnr_user_teams`

**Business Purpose:** Maps a user's org membership to an R&R team. `can_give_money` and `can_give_perks` permission fields are set by Bravo staff only and control which reward types the R&R admin can award within this team context.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | The user's org membership being assigned to an R&R team |
| `rnr_team_uuid` | `String` | FK → `rnr_teams.uuid`, NOT NULL | The R&R team the membership is being added to |
| `is_rnr_admin` | `Boolean` | NOT NULL, default=False | Whether this user acts as an R&R admin (team leader) for this team — can nominate and give rewards on behalf of the team |
| `can_give_money` | `Boolean` | NOT NULL, default=False | Whether this user can give monetary rewards within this team — **set by Bravo staff only** |
| `can_give_perks` | `Boolean` | NOT NULL, default=False | Whether this user can give perk rewards within this team — **set by Bravo staff only** |
| `email_opt_in` | `Boolean` | NOT NULL, default=True | Whether the user opts in to R&R notification emails for this team context |
| `assigned_at` | `DateTime (tz)` | NOT NULL, default=now | Timestamp of when this membership was assigned to the team |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_rnr_user_teams_membership_team` | `membership_uuid`, `rnr_team_uuid` | Unique — prevents duplicate team assignments |
| `idx_rnr_user_teams_team` | `rnr_team_uuid` | Index |

---

### `rnr_values`

**Business Purpose:** Org-defined value cards — the nomination types available for selection when an employee submits a nomination. Retired cards are retained for historical nomination display; they cannot be selected for new nominations. Birthday and long-service auto-nominations reference dedicated system cards of the corresponding `card_type`.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `value_name` | `String(200)` | NOT NULL | Display name of the value / nomination type shown in the nomination selection UI |
| `image_uuid` | `String` | FK → `image.uuid`, Nullable | Value card image UUID from the `image` table (`ON DELETE SET NULL`) |
| `card_type` | `String(20)` | NOT NULL, default=`nomination`, CHECK IN (`nomination`, `birthday`, `long_service`, `rnr_admin`) | Type of card: `nomination` = standard employee-to-employee; `birthday` = auto-birthday card; `long_service` = auto-anniversary card; `rnr_admin` = admin-initiated |
| `show_on_wall` | `Boolean` | NOT NULL, default=True | Whether nominations using this value card appear on the Wall of Fame |
| `is_retired` | `Boolean` | NOT NULL, default=False | When True, the card is hidden from new nomination selection but retained for historical display |
| `display_order` | `Integer` | NOT NULL, default=0 | Sort order in the nomination UI; lower = displayed first |

> **SQLAlchemy `__table_args__` Note:** `card_type` requires:
> `CheckConstraint("card_type IN ('nomination', 'birthday', 'long_service', 'rnr_admin')", name='ck_rnr_values_card_type')`

---

### `rnr_nominations`

**Business Purpose:** Core nomination record — the anchor of every R&R event, including automated birthday and long-service recognitions. Bravo staff can correct typos on existing nominations. Cannot be hard-deleted or soft-deleted (`is_deleted=True`) if a linked `rnr_rewards` row exists — the reward is the financial ledger anchor.

> **1:1 Relationship with `rnr_rewards`:** A nomination and its reward are in a strict one-to-one relationship. `rnr_rewards.nomination_uuid` carries a `UNIQUE` constraint — there can never be more than one reward per nomination. Automated rewards (birthday, long-service) still require the scheduler to generate a system nomination row first, which acts as the ledger anchor, before a reward row is created.

> **Soft-Delete Guard:** A nomination with a linked `rnr_rewards` row must **never** have `is_deleted=True` set. The `ON DELETE RESTRICT` FK on the reward row blocks physical deletion at the DB layer; however, the `is_deleted` boolean bypasses that constraint and must be guarded at the service layer (see guard below).

**Base Class:** `Base` + `SqlAlchemyEventMixin`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `nominator_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | Membership of the user making the nomination (or the system scheduler for automated nominations) |
| `nominee_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | Membership of the user being nominated |
| `value_uuid` | `String` | FK → `rnr_values.uuid`, NOT NULL | The R&R value card selected for this nomination |
| `title` | `String(255)` | NOT NULL | Nomination headline shown on the Wall of Fame |
| `description` | `Text` | Nullable | Full nomination reason / story; shown on wall if `rnr_config.show_desc_on_wall = True` |
| `status` | `String(15)` | NOT NULL, default=`pending`, CHECK IN (`pending`, `approved`, `rejected`, `auto_approved`) | Approval workflow status: `pending` = awaiting review (when `rnr_config.approve_before_publish = True`); `approved` / `rejected` = admin decision; `auto_approved` = instantly published without review |
| `show_on_wall` | `Boolean` | NOT NULL, default=True | Whether this nomination is visible on the Wall of Fame |
| `approved_by_uuid` | `String` | FK → `users.uuid`, Nullable | UUID of the user who approved or rejected the nomination |
| `approved_at` | `DateTime (tz)` | Nullable | Timestamp of the approval or rejection decision |
| `is_deleted` | `Boolean` | NOT NULL, default=False | Bravo-only soft-delete flag; service layer must block this if a linked `rnr_rewards` row exists (see Soft-Delete Guard below) |

> **SQLAlchemy `__table_args__` Note:** `status` requires:
> `CheckConstraint("status IN ('pending', 'approved', 'rejected', 'auto_approved')", name='ck_rnr_nominations_status')`

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `nominator` | `UserOrgMembership` | Many-to-One | The nominating employee or system | `ON DELETE RESTRICT` — membership cannot be deleted while nominations exist |
| `nominee` | `UserOrgMembership` | Many-to-One | The nominated employee | `ON DELETE RESTRICT` — membership cannot be deleted while nominations exist |
| `value` | `RnrValue` | Many-to-One | The R&R value card selected | `ON DELETE RESTRICT` — value card cannot be deleted while nominations reference it |
| `reward` | `RnrReward` | One-to-One | The associated reward record (if any) | `ON DELETE RESTRICT` — **nomination cannot be deleted while a reward record exists** |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_rnr_nominations_org` | `org_uuid` | Index |
| `idx_rnr_nominations_nominee` | `nominee_uuid` | Index |
| `idx_rnr_wall` | `org_uuid`, `show_on_wall`, `status` WHERE `is_deleted = False` | Partial Index — used by Wall of Fame queries |

#### Soft-Delete Guard

> ⚠️ **Critical Business Rule:** A nomination that has an associated `rnr_rewards` row **must never be flagged `is_deleted=True`**. The reward relationship carries `ON DELETE RESTRICT` at the DB layer (blocks physical deletion), but the `is_deleted` soft flag bypasses that constraint — guard this at the **service layer**.

```python
from app.models.rnr_nominations import RnrNomination
from app.models.rnr_rewards import RnrReward

def soft_delete_nomination(nomination_uuid: str) -> None:
    """
    Soft-delete a nomination. Raises if a financial reward is linked —
    a nomination with a reward is immutable (financial audit trail).
    Only callable by Bravo staff.
    """
    nomination = RnrNomination.get_by_uuid(nomination_uuid)
    if nomination is None:
        raise ValueError(f"Nomination {nomination_uuid} not found.")

    # Guard: block soft-delete if a reward record exists
    linked_reward = RnrReward.get_by_nomination_uuid(nomination_uuid)
    if linked_reward is not None:
        raise PermissionError(
            f"Nomination {nomination_uuid} cannot be deleted because a "
            f"financial reward (uuid={linked_reward.uuid}) is linked to it. "
            "Retract the reward first."
        )

    nomination.update_multiple_property_by_uuid(
        nomination_uuid, {'is_deleted': True}
    )
```

---

### `rnr_rewards`

**Business Purpose:** Monetary or perk reward linked to a nomination — strict **1:1** relationship enforced by the `UNIQUE` constraint on `nomination_uuid`. One reward per nomination, never many-to-one. Automated rewards (birthdays, long-service) still require the scheduler to generate a system nomination row first as the ledger anchor before this row is created.

`predefined_perk_name` and `predefined_perk_emoji` capture an **immutable snapshot** of the perk at reward creation time. This means historical reward records remain accurate even if the org later edits or removes the perk from `rnr_config.predefined_perks` — the snapshot is self-contained and does not reference the live catalogue. `custom_perk_text` is retained for free-text ad-hoc perks that are not drawn from the predefined catalogue.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `nomination_uuid` | `String` | FK → `rnr_nominations.uuid`, **Unique**, NOT NULL | The associated nomination. **UNIQUE constraint enforces the strict 1:1 rule — one reward per nomination, never more.** |
| `reward_type` | `String(10)` | NOT NULL, CHECK IN (`money`, `perk`) | Whether this is a monetary (`money`) or perk-based (`perk`) reward |
| `amount` | `Numeric(10,2)` | Nullable | Monetary reward value in GBP (must be a multiple of £5; enforced at the application layer); NULL for perk rewards |
| `gift_card_url` | `Text` | Nullable, Globally Unique | HTTPS gift card redemption URL — globally unique one-time-use voucher link; NULL until Bravo staff issues the voucher |
| `gift_card_sent_at` | `DateTime (tz)` | Nullable | Timestamp when Bravo sent the gift card voucher link to the nominee |
| `predefined_perk_name` | `String(200)` | Nullable | **Immutable snapshot** of the perk name at reward time, copied from `rnr_config.predefined_perks[*].name`. Historical records remain unaffected if the org later edits the perk catalogue. *Replaces the deleted FK `perk_uuid → rnr_perks.uuid`.* |
| `predefined_perk_emoji` | `String(10)` | Nullable | **Immutable snapshot** of the perk emoji at reward time, copied from `rnr_config.predefined_perks[*].emoji`. Stored alongside the name so the reward card can be re-rendered without looking up the live catalogue. *Replaces the deleted FK `perk_uuid → rnr_perks.uuid`.* |
| `custom_perk_text` | `Text` | Nullable | Free-text description for ad-hoc perks not drawn from the predefined catalogue (e.g. `"Team lunch at a restaurant of your choice"`) |
| `is_claimed` | `Boolean` | NOT NULL, default=False | Whether the gift card has been claimed/redeemed by the nominee |
| `claimed_at` | `DateTime (tz)` | Nullable | Timestamp when the nominee redeemed the gift card |
| `given_by_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | Membership UUID of the R&R admin who gave this reward |
| `given_at` | `DateTime (tz)` | NOT NULL, default=now | Timestamp when the reward was formally given |

> **SQLAlchemy `__table_args__` Note:** `reward_type` requires:
> `CheckConstraint("reward_type IN ('money', 'perk')", name='ck_rnr_rewards_reward_type')`

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `nomination` | `RnrNomination` | One-to-One | The parent nomination this reward is attached to | `ON DELETE RESTRICT` — nomination cannot be deleted while a reward row exists |
| `given_by` | `UserOrgMembership` | Many-to-One | The R&R admin who gave the reward | `ON DELETE RESTRICT` — membership cannot be deleted while rewards exist |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_rnr_rewards_nomination` | `nomination_uuid` | Unique — enforces strict 1:1 with `rnr_nominations`; one reward per nomination |
| `idx_rnr_rewards_gift_url` | `gift_card_url` WHERE `gift_card_url IS NOT NULL` | Partial Unique — globally unique gift card URLs |

---

### `rnr_budget_individual`

**Business Purpose:** **Append-only budget ledger** for `per_user` reward pot mode. Budget is assigned to a specific R&R admin membership. Corrections are made via reversal entries — **never UPDATE or DELETE committed rows**. Summing all entries for a membership gives the current spendable balance.

**Base Class:** `db.Model` + `UserAuditMixin`

> ⚡ **Ledger Rule:** This table deliberately does **not** extend `Base` to prevent accidental `save()` / `update_multiple_property_by_uuid()` mutations via the standard ORM helpers. All writes must go through a dedicated `BudgetService` that enforces append-only semantics. Apply a `SqlAlchemyEventMixin.on_before_update` guard or a PostgreSQL rule to block UPDATEs at the DB layer.

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Surrogate primary key |
| `uuid` | `String` | Unique | Public identifier |
| `org_uuid` | `String` | FK → `organisations.uuid`, NOT NULL | Tenant scoping |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | R&R admin whose budget this ledger entry belongs to |
| `entry_type` | `String(15)` | NOT NULL, CHECK IN (`credit`, `debit`, `adjustment`, `correction`, `employee_return`) | Ledger entry type: `credit` = budget added; `debit` = reward given; `adjustment` = admin correction; `correction` = reversal entry; `employee_return` = unused budget returned |
| `amount` | `Numeric(10,2)` | NOT NULL | Monetary value — positive for credits, negative for debits |
| `notes` | `Text` | Nullable | Optional note explaining the entry (required for `correction` and `adjustment` types) |
| `reference_uuid` | `String` | Nullable | UUID of the related `rnr_nominations` or `rnr_rewards` row for audit traceability |
| `created_by` | `String` | FK → `users.uuid`, NOT NULL | User who created the entry; `credit` entries must be created by `bravo_staff` only |
| `created_at` | `DateTime (tz)` | NOT NULL, default=now | Entry creation timestamp |

> **SQLAlchemy `__table_args__` Note:** `entry_type` requires:
> `CheckConstraint("entry_type IN ('credit','debit','adjustment','correction','employee_return')", name='ck_rnr_budget_individual_entry_type')`

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_rnr_budget_ind_org` | `org_uuid` | Index |
| `idx_rnr_budget_ind_membership` | `membership_uuid` | Index |
| *(App/DB rule)* | No UPDATE or DELETE | Append-only enforcement |

#### Query Pattern

> **Balance Query — `per_user` mode:**

```python
from sqlalchemy import func
from app.models.rnr_budget_individual import RnrBudgetIndividual

def get_individual_balance(db_session, membership_uuid: str, org_uuid: str) -> float:
    """Return current R&R budget balance for a given membership (per_user mode)."""
    result = db_session.query(
        func.coalesce(func.sum(RnrBudgetIndividual.amount), 0)
    ).filter(
        RnrBudgetIndividual.membership_uuid == membership_uuid,
        RnrBudgetIndividual.org_uuid == org_uuid,
    ).scalar()
    return float(result)
```

#### Append-Only Guard

```python
from sqlalchemy import event

@event.listens_for(RnrBudgetIndividual, 'before_update')
def prevent_budget_individual_update(mapper, connection, target):
    """Append-only ledger guard — raises immediately on any attempted UPDATE."""
    raise RuntimeError(
        f"[RnrBudgetIndividual] Immutable ledger violation: attempted UPDATE on "
        f"budget_id={target.id} (uuid={target.uuid}). "
        "Use a reversal entry (entry_type='correction') instead."
    )
```

> For an additional DB-layer safety net:
> ```sql
> CREATE RULE no_update_rnr_budget_individual
>     AS ON UPDATE TO rnr_budget_individual DO INSTEAD NOTHING;
> ```

---

### `rnr_budget_team`

**Business Purpose:** **Append-only budget ledger** for `per_team` reward pot mode. Budget is held against the R&R team (shared pool across all admins in the team). Same append-only rules apply as `rnr_budget_individual`. A company cannot mix pot modes — `rnr_config.reward_pot_mode` is org-wide and mutually exclusive.

**Base Class:** `db.Model` + `UserAuditMixin`

> ⚡ **Ledger Rule:** Same append-only constraints as `rnr_budget_individual`. Apply a `before_update` event listener and/or PostgreSQL rule. All writes must go through `BudgetService`.

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Surrogate primary key |
| `uuid` | `String` | Unique | Public identifier |
| `org_uuid` | `String` | FK → `organisations.uuid`, NOT NULL | Tenant scoping |
| `rnr_team_uuid` | `String` | FK → `rnr_teams.uuid`, NOT NULL | R&R team whose budget this ledger entry belongs to |
| `entry_type` | `String(15)` | NOT NULL, CHECK IN (`credit`, `debit`, `adjustment`, `correction`, `employee_return`) | Ledger entry type — same semantics as `rnr_budget_individual.entry_type` |
| `amount` | `Numeric(10,2)` | NOT NULL | Monetary value — positive for credits, negative for debits |
| `notes` | `Text` | Nullable | Optional note explaining the entry |
| `reference_uuid` | `String` | Nullable | UUID of the related `rnr_nominations` or `rnr_rewards` row |
| `created_by` | `String` | FK → `users.uuid`, NOT NULL | User who created the entry; `credit` entries must be created by `bravo_staff` only |
| `created_at` | `DateTime (tz)` | NOT NULL, default=now | Entry creation timestamp |

> **SQLAlchemy `__table_args__` Note:** `entry_type` requires:
> `CheckConstraint("entry_type IN ('credit','debit','adjustment','correction','employee_return')", name='ck_rnr_budget_team_entry_type')`

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_rnr_budget_team_org` | `org_uuid` | Index |
| `idx_rnr_budget_team_team` | `rnr_team_uuid` | Index |
| *(App/DB rule)* | No UPDATE or DELETE | Append-only enforcement |

#### Query Pattern

> **Balance Query — `per_team` mode:**

```python
from sqlalchemy import func
from app.models.rnr_budget_team import RnrBudgetTeam

def get_team_balance(db_session, rnr_team_uuid: str, org_uuid: str) -> float:
    """Return current R&R budget balance for a given R&R team (per_team mode)."""
    result = db_session.query(
        func.coalesce(func.sum(RnrBudgetTeam.amount), 0)
    ).filter(
        RnrBudgetTeam.rnr_team_uuid == rnr_team_uuid,
        RnrBudgetTeam.org_uuid == org_uuid,
    ).scalar()
    return float(result)
```

#### Append-Only Guard

```python
from sqlalchemy import event

@event.listens_for(RnrBudgetTeam, 'before_update')
def prevent_budget_team_update(mapper, connection, target):
    """Append-only ledger guard — raises immediately on any attempted UPDATE."""
    raise RuntimeError(
        f"[RnrBudgetTeam] Immutable ledger violation: attempted UPDATE on "
        f"budget_id={target.id} (uuid={target.uuid}). "
        "Use a reversal entry (entry_type='correction') instead."
    )
```

> For an additional DB-layer safety net:
> ```sql
> CREATE RULE no_update_rnr_budget_team
>     AS ON UPDATE TO rnr_budget_team DO INSTEAD NOTHING;
> ```

---

### Domain 4 — Content, Support & Marketing

---

### `documents`

**Business Purpose:** Employer-uploaded document library. Org admins and group admins upload documents (policy handbooks, benefit guides, etc.) that employees can download. Group admins can push a document across all linked orgs simultaneously via `group_uuid`. Team-scoped visibility is managed directly via the `visible_team_uuids` JSONB array — an empty array means the document is visible to all teams in the org (replacing the deleted `document_team_visibility` join table).

> **Implementation Note:** The `visible_team_uuids` array requires an `after_delete` application hook on the `OrgTeam` model to remove any deleted team UUID from this array across all document rows in the affected org, preventing stale UUID references.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `group_uuid` | `String` | FK → `groups.uuid`, Nullable | When set, this document was pushed by a group admin and is visible to all orgs in the group (`ON DELETE SET NULL`) |
| `title` | `String(255)` | NOT NULL | Document display title shown in the employee document library |
| `file_url` | `Text` | NOT NULL | S3 URL of the uploaded document file |
| `mime_type` | `String(100)` | NOT NULL, CHECK IN allowed types | File MIME type — validated on upload. Allowed values: `application/pdf`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`, `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`, `text/csv`, `application/vnd.ms-excel`, `application/vnd.openxmlformats-officedocument.presentationml.presentation` |
| `is_group_document` | `Boolean` | NOT NULL, default=False | Whether this document was pushed by a group admin across all linked orgs |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether the document is visible to employees; False hides it without soft-deleting |
| `visible_team_uuids` | `JSONB` | NOT NULL, default=`[]` | Array of `org_teams.uuid` strings restricting visibility to specific teams. An empty array `[]` means the document is visible to **all teams** in the org. Replaces the deleted `document_team_visibility` join table. See **`visible_team_uuids` JSONB Structure** below |

> **SQLAlchemy `__table_args__` Note:** `mime_type` requires a `CheckConstraint`. Given the long values, define it as a named constant:
> ```python
> _ALLOWED_MIME_TYPES = (
>     'application/pdf',
>     'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
>     'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
>     'text/csv',
>     'application/vnd.ms-excel',
>     'application/vnd.openxmlformats-officedocument.presentationml.presentation',
> )
> CheckConstraint(f"mime_type IN {_ALLOWED_MIME_TYPES}", name='ck_documents_mime_type')
> ```

#### `visible_team_uuids` JSONB Structure

```json
["<org_team_uuid_1>", "<org_team_uuid_2>"]
```

| Key | Type | Description |
|---|---|---|
| *(array element)* | `String (UUID)` | UUID string referencing an `org_teams.uuid` value. When the array is populated, only employees who are members of one of these teams (via `user_org_teams`) will see this document. An empty array or `null` means all teams in the org can see it. *Replaces the deleted `document_team_visibility` join table.* |

> **`after_delete` Hook on `OrgTeam`:** When an `OrgTeam` row is deleted, a service-layer hook must iterate all `documents` in that org and remove the deleted team's UUID from every `visible_team_uuids` array. Use a single `UPDATE documents SET visible_team_uuids = visible_team_uuids - '<team_uuid>'::text WHERE org_uuid = '<org_uuid>'` PostgreSQL array-remove expression.

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_documents_org` | `org_uuid` | Index |
| `idx_documents_group` | `group_uuid` | Index — used when resolving group-pushed documents |

---

### `notices`

**Business Purpose:** Org or group-level rich HTML notices published to the employee newsfeed. Employees can like notices (tracked separately in `notice_likes`). Group admins can push notices across all linked orgs simultaneously. Team-scoped visibility is managed via the `visible_team_uuids` JSONB array. `like_count` is a denormalised counter maintained by the application for fast, read-optimised feed generation — avoids a COUNT aggregation query on the hot `notice_likes` table on every feed load.

> **Implementation Note:** The `visible_team_uuids` array requires an `after_delete` application hook on the `OrgTeam` model (same pattern as `documents`) to clean up deleted team UUIDs.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `group_uuid` | `String` | FK → `groups.uuid`, Nullable | When set, this notice was pushed by a group admin and is visible to all orgs in the group (`ON DELETE SET NULL`) |
| `title` | `String(255)` | NOT NULL | Notice headline shown on the employee newsfeed listing |
| `short_desc` | `String(500)` | Nullable | Summary text shown beneath the title in listing/card views before the employee opens the full notice |
| `html_content` | `Text` | Nullable | Full rich HTML body of the notice rendered on the notice detail page |
| `image_uuid` | `String` | FK → `image.uuid`, Nullable | Hero image UUID from the `image` table displayed as the notice card cover (`ON DELETE SET NULL`) |
| `is_group_notice` | `Boolean` | NOT NULL, default=False | Whether this notice was pushed by a group admin across all linked orgs |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether the notice is visible to employees; False suppresses it without soft-deleting |
| `like_count` | `Integer` | NOT NULL, default=0 | Denormalised like counter — incremented/decremented by the application when employees like/unlike. Maintained for fast, read-optimised feed generation; avoids a COUNT query on `notice_likes` on every feed render |
| `visible_team_uuids` | `JSONB` | NOT NULL, default=`[]` | Array of `org_teams.uuid` strings restricting visibility to specific teams. An empty array `[]` means the notice is visible to **all teams** in the org. Replaces the deleted `notice_team_visibility` join table. See **`visible_team_uuids` JSONB Structure** below |

#### `visible_team_uuids` JSONB Structure

```json
["<org_team_uuid_1>", "<org_team_uuid_2>"]
```

| Key | Type | Description |
|---|---|---|
| *(array element)* | `String (UUID)` | UUID string referencing an `org_teams.uuid` value. When the array is populated, only employees who are members of one of these teams (via `user_org_teams`) will see this notice. An empty array means all teams in the org can see it. *Replaces the deleted `notice_team_visibility` join table.* |

> **`after_delete` Hook on `OrgTeam`:** Same pattern as `documents.visible_team_uuids` — when an `OrgTeam` is deleted, a service hook removes the deleted UUID from all notice `visible_team_uuids` arrays in the affected org.

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_notices_org` | `org_uuid` | Index |
| `idx_notices_group` | `group_uuid` | Index — used when resolving group-pushed notices |

---

### `notice_likes`

**Business Purpose:** Records which employees have liked a notice — one like row per employee per notice. Enforced via composite primary key. The like count itself is stored as the denormalised `notices.like_count` column for fast feed generation; `notice_likes` is the authoritative source of truth for the unique-per-employee constraint and for "did this user like this notice?" queries. This table is designed to handle **concurrent writes** safely — the composite PK provides an implicit unique constraint, and the application uses INSERT … ON CONFLICT DO NOTHING to avoid race conditions when two sessions try to record the same like simultaneously.

**Base Class:** `db.Model` (interaction log — no full `Base` fields required)

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `notice_uuid` | `String` | FK → `notices.uuid`, NOT NULL | The notice being liked (`ON DELETE CASCADE` — likes removed with the notice) |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | The employee who liked the notice |
| `liked_at` | `DateTime (tz)` | NOT NULL, default=now | Timestamp when the like was recorded |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| *(composite PK)* | `notice_uuid`, `membership_uuid` | Primary Key — enforces one like per employee per notice |

---

### `support_tickets`

**Business Purpose:** Employee-raised support tickets providing a structured messaging channel between employees and Bravo staff. Supports context-specific tickets (`benefit`, `general`, `system`) and SLA tracking fields (`assigned_at`, `resolved_at`) surfaced on the Bravo staff dashboard. One thread per ticket; individual messages live in `support_ticket_messages`.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `raised_by_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | Membership of the employee who raised the ticket |
| `context_type` | `String(10)` | NOT NULL, default=`general`, CHECK IN (`benefit`, `general`, `system`) | Ticket context category: `benefit` = related to a specific benefit; `general` = general enquiry; `system` = platform/technical issue |
| `related_benefit_uuid` | `String` | FK → `org_benefits.uuid`, Nullable | The specific benefit this ticket concerns — populated when `context_type = benefit` |
| `subject` | `String(300)` | NOT NULL | Ticket subject line displayed in the Bravo staff ticket list |
| `status` | `String(15)` | NOT NULL, default=`open`, CHECK IN (`open`, `in_progress`, `resolved`, `cleared`) | Ticket workflow status: `open` = new; `in_progress` = assigned and being worked; `resolved` = answer provided; `cleared` = closed without action |
| `assigned_to_uuid` | `String` | FK → `users.uuid`, Nullable | Bravo staff member currently assigned to handle this ticket |
| `assigned_at` | `DateTime (tz)` | Nullable | Timestamp when the ticket was assigned to a Bravo staff member (SLA start) |
| `resolved_at` | `DateTime (tz)` | Nullable | Timestamp when the ticket was resolved (SLA end); used for resolution time reporting on the Bravo staff dashboard |

> **SQLAlchemy `__table_args__` Note:** The following ENUM columns require `CheckConstraint` entries:
> - `context_type`: `CheckConstraint("context_type IN ('benefit', 'general', 'system')", name='ck_support_tickets_context_type')`
> - `status`: `CheckConstraint("status IN ('open', 'in_progress', 'resolved', 'cleared')", name='ck_support_tickets_status')`

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `messages` | `SupportTicketMessage` | One-to-Many | All messages in this ticket thread | `ON DELETE CASCADE` — messages removed with the ticket |
| `raised_by` | `UserOrgMembership` | Many-to-One | The employee who raised the ticket | `ON DELETE RESTRICT` |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_tickets_org` | `org_uuid` | Index |
| `idx_tickets_status` | `status` | Index |
| `idx_tickets_assigned` | `assigned_to_uuid` WHERE `status != 'resolved'` | Partial Index — drives the Bravo staff open-ticket assignment list |

---

### `support_ticket_messages`

**Business Purpose:** Individual messages within a support ticket thread. Both the employee and Bravo staff write messages here; the `sender_uuid` links to the `users` table (not membership) to capture the Bravo staff identity. Supports optional file attachments. `is_read` drives the unread-message badge on the employee and Bravo staff UIs.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `ticket_uuid` | `String` | FK → `support_tickets.uuid`, NOT NULL | The parent ticket this message belongs to (`ON DELETE CASCADE`) |
| `sender_uuid` | `String` | FK → `users.uuid`, NOT NULL | The platform user who sent this message — links to `users` (not membership) to support Bravo staff as senders |
| `body` | `Text` | NOT NULL | Full message content displayed in the ticket thread |
| `attachment_url` | `Text` | Nullable | Optional S3 URL of a file attached to this message |
| `is_read` | `Boolean` | NOT NULL, default=False | Whether the recipient has read this message; drives the unread-message notification badge |
| `sent_at` | `DateTime (tz)` | NOT NULL, default=now | Timestamp when the message was sent; displayed in the thread view |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_ticket_messages_ticket` | `ticket_uuid` | Index — used to retrieve all messages for a ticket in thread order |

---

### `notifications`

**Business Purpose:** In-app and push notification feed per user (renamed from `newsfeed_events` to better reflect its dual in-app/push role and allow future SMS/push expansion). Uses a polymorphic `reference_uuid` + `reference_table` pattern to point to the source entity without hard FK constraints. One row per recipient per event — the feed is assembled by querying unread rows for the requesting membership.

**Base Class:** `Base`

> ⚡ **Rename Note:** Renamed from `newsfeed_events`. All FK references, model names, and index names updated accordingly. The rename allows future SMS and push notification channels to be documented under the same table without the name implying a feed-only context.

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `recipient_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | The membership receiving this notification |
| `event_type` | `String(30)` | NOT NULL, CHECK IN (`notice`, `document`, `message`, `rnr_nomination`, `rnr_wall`, `birthday`, `long_service`, `advert`) | Type of notification event — drives the icon and routing behaviour in the employee notification centre |
| `reference_uuid` | `String` | Nullable | UUID of the referenced source entity (e.g. the `notice_uuid`, `rnr_nominations.uuid`) |
| `reference_table` | `String(100)` | Nullable | Table name of the referenced entity — used by the frontend to construct the deep-link URL |
| `title` | `String(255)` | Nullable | Notification title displayed in the notification centre |
| `body_preview` | `Text` | Nullable | Short preview text shown beneath the title in the notification list |
| `is_read` | `Boolean` | NOT NULL, default=False | Whether the employee has read/dismissed this notification; False = drives the unread badge count |
| `push_sent_at` | `DateTime (tz)` | Nullable | Timestamp when the push notification was dispatched to the mobile device; NULL if no push was sent |

> **SQLAlchemy `__table_args__` Note:** `event_type` requires:
> ```python
> CheckConstraint(
>     "event_type IN ('notice','document','message','rnr_nomination',"
>     "'rnr_wall','birthday','long_service','advert')",
>     name='ck_notifications_event_type'
> )
> ```

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_notifications_recipient` | `recipient_uuid`, `is_read`, `created_at DESC` | Composite Index — used for paginated feed queries and unread count |

---

### `templates`

**Business Purpose:** Global HTML email templates managed exclusively by Bravo staff. One record per template type; each `template_type` is unique across the platform. Renamed from `email_templates` to allow future expansion to SMS and push notification templates without a disruptive schema rename.

> ⚡ **Rename Note:** Renamed from `email_templates` to `templates`. All FK references, model names, index names, and constraint names updated accordingly. The rename future-proofs the table for SMS/push template types without schema migration.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid` (NULL — platform-level record), `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `template_name` | `String(200)` | NOT NULL | Human-readable template name displayed in the Bravo staff template management UI |
| `template_type` | `String(50)` | Unique, NOT NULL, CHECK IN allowed types | System template identifier — uniquely identifies the template used by email dispatch workers. Allowed values: `welcome`, `self_reg`, `invite_reminder`, `rnr_approval`, `rnr_monthly_roundup`, `rnr_birthday`, `rnr_long_service`, `rnr_team_leader_fortnightly`, `new_document`, `new_notice`, `gift_card`, `trs_ready` |
| `html_content` | `Text` | NOT NULL | Full HTML email template body including token placeholders for dynamic substitution |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether this template is active and available for dispatch; inactive templates fall back to the platform default |

> **SQLAlchemy `__table_args__` Note:** `template_type` requires:
> ```python
> CheckConstraint(
>     "template_type IN ('welcome','self_reg','invite_reminder','rnr_approval',"
>     "'rnr_monthly_roundup','rnr_birthday','rnr_long_service',"
>     "'rnr_team_leader_fortnightly','new_document','new_notice','gift_card','trs_ready')",
>     name='ck_templates_template_type'
> )
> ```

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_templates_type` | `template_type` | Unique — one template per type across the platform |

---

### `marketing_categories`

**Business Purpose:** Platform-level taxonomy categories for grouping marketing assets (e.g. `Benefits`, `R&R`, `Wellbeing`). Managed by Bravo staff. Client orgs cannot create or modify categories. Referenced by `marketing_assets.category_uuids` JSONB array.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid` (NULL — platform-level), `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `category_name` | `String(200)` | NOT NULL | Display name of the category shown in the marketing asset filter UI (e.g. `Benefits`, `Wellbeing`, `R&R`) |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether this category is active and available as a filter option; inactive categories are hidden from the filter UI but retained for historical asset classification |

---

### `marketing_assets`

**Business Purpose:** Static content library for org launch materials (posters, email templates, graphics, videos). Filterable by launch phase and platform type. Renamed from `marketing_materials` to better reflect that this table stores downloadable creative assets, not generic marketing data. Benefit associations and category assignments are managed as JSONB arrays on this table (replacing the deleted `marketing_benefit_links` and `marketing_category_materials` join tables), with GIN indexes for fast reverse-lookups.

> ⚡ **Rename Note:** Renamed from `marketing_materials`. All FK references, model names, index names, and constraint names updated accordingly.

> **`after_delete` Hook on `MarketingCategory`:** When a `MarketingCategory` is deleted, a service-layer hook must remove the deleted category UUID from `marketing_assets.category_uuids` for all affected rows.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid` (NULL — platform-level), `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `title` | `String(255)` | NOT NULL | Asset display title shown in the marketing library |
| `material_type` | `String(10)` | NOT NULL, CHECK IN (`poster`, `email`, `graphic`, `video`) | Format type of the asset |
| `launch_phase` | `String(5)` | NOT NULL, CHECK IN (`pre`, `post`, `both`) | Whether this asset is for pre-launch, post-launch, or both phases |
| `platform_type` | `String(10)` | NOT NULL, default=`both`, CHECK IN (`full`, `basic`, `both`) | Applicable platform tier: `full`, `basic`, or `both` |
| `content_url` | `Text` | Nullable | URL to the downloadable asset (poster PDF, graphic file, video link) |
| `sender_email` | `String(255)` | Nullable | Sender email address for email-type assets |
| `subject` | `String(300)` | Nullable | Subject line for email-type assets |
| `html_content` | `Text` | Nullable | HTML body content for email-type assets |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether this asset is available for download in the org admin marketing library |
| `linked_benefit_uuids` | `JSONB` | NOT NULL, default=`[]` | Array of `master_benefits.uuid` strings associating this asset with one or more master benefits. Replaces the deleted `marketing_benefit_links` join table. **Requires a GIN Index** for fast reverse-lookup (find all assets for a given benefit). See **`linked_benefit_uuids` JSONB Structure** below |
| `category_uuids` | `JSONB` | NOT NULL, default=`[]` | Array of `marketing_categories.uuid` strings categorising this asset. Replaces the deleted `marketing_category_materials` join table. **Requires a GIN Index** for fast reverse-lookup (filter all assets by category). See **`category_uuids` JSONB Structure** below |

> **SQLAlchemy `__table_args__` Note:** The following ENUM columns require `CheckConstraint` entries:
> - `material_type`: `CheckConstraint("material_type IN ('poster', 'email', 'graphic', 'video')", name='ck_marketing_assets_material_type')`
> - `launch_phase`: `CheckConstraint("launch_phase IN ('pre', 'post', 'both')", name='ck_marketing_assets_launch_phase')`
> - `platform_type`: `CheckConstraint("platform_type IN ('full', 'basic', 'both')", name='ck_marketing_assets_platform_type')`

#### `linked_benefit_uuids` JSONB Structure

```json
["<master_benefit_uuid_1>", "<master_benefit_uuid_2>"]
```

| Key | Type | Description |
|---|---|---|
| *(array element)* | `String (UUID)` | UUID string referencing a `master_benefits.uuid` value. Enables filtering the marketing library by benefit. *Replaces the deleted `marketing_benefit_links` join table.* |

#### `category_uuids` JSONB Structure

```json
["<marketing_category_uuid_1>", "<marketing_category_uuid_2>"]
```

| Key | Type | Description |
|---|---|---|
| *(array element)* | `String (UUID)` | UUID string referencing a `marketing_categories.uuid` value. Enables filtering the marketing library by category. *Replaces the deleted `marketing_category_materials` join table.* |

> **`after_delete` Hook on `MarketingCategory`:** When a `MarketingCategory` row is deleted, a service hook must remove the deleted UUID from `marketing_assets.category_uuids` across all affected rows, using a PostgreSQL array-remove expression.

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_marketing_assets_linked_benefits` | `linked_benefit_uuids` | **GIN Index** — required for fast reverse-lookup: find all assets linked to a given `master_benefit_uuid` |
| `idx_marketing_assets_categories` | `category_uuids` | **GIN Index** — required for fast reverse-lookup: filter all assets belonging to a given category |

---

### `adverts`

**Business Purpose:** Runtime user-facing adverts shown to employees in the Bravo portal (distinct from marketing assets — different lifecycle, different query patterns). Managed by Bravo staff. Org targeting is managed via the `target_org_uuids` JSONB array (replacing the deleted `advert_org_targets` join table). Per-user suppression (employee dismissals) is tracked separately in `advert_user_suppression`.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid` (NULL = platform-wide), `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `name` | `String(200)` | NOT NULL | Internal advert name used in the Bravo staff advert management UI |
| `body_text` | `Text` | Nullable | Advert body copy displayed to the employee in the portal |
| `image_url` | `Text` | Nullable | URL of the advert image displayed in the employee portal |
| `cta_link` | `Text` | Nullable | Call-to-action URL opened when the employee clicks the advert |
| `target_type` | `String(20)` | NOT NULL, default=`all`, CHECK IN (`all`, `specific_orgs`, `specific_benefit`) | Targeting mode: `all` = shown to every eligible employee; `specific_orgs` = only shown to employees in the listed orgs in `target_org_uuids`; `specific_benefit` = shown only to employees whose org has the targeted benefit |
| `target_benefit_uuid` | `String` | FK → `master_benefits.uuid`, Nullable | The master benefit used for targeting when `target_type = specific_benefit` (`ON DELETE SET NULL`) |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether the advert is currently live and eligible to be served to employees |
| `target_org_uuids` | `JSONB` | NOT NULL, default=`[]` | Array of `organisations.uuid` strings restricting advert delivery to specific orgs. Used when `target_type = specific_orgs`. An empty array means the `target_type` takes precedence (i.e. `all` or `specific_benefit`). Replaces the deleted `advert_org_targets` join table. **Requires a GIN Index** for fast eligibility lookup. See **`target_org_uuids` JSONB Structure** below |

> **SQLAlchemy `__table_args__` Note:** `target_type` requires:
> `CheckConstraint("target_type IN ('all', 'specific_orgs', 'specific_benefit')", name='ck_adverts_target_type')`

#### `target_org_uuids` JSONB Structure

```json
["<org_uuid_1>", "<org_uuid_2>"]
```

| Key | Type | Description |
|---|---|---|
| *(array element)* | `String (UUID)` | UUID string referencing an `organisations.uuid` value. When `target_type = specific_orgs`, the advert is served only to employees whose `org_uuid` appears in this array. An empty array in combination with `target_type = all` means the advert is served platform-wide. *Replaces the deleted `advert_org_targets` join table.* |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_adverts_active` | `is_active` WHERE `is_active = True AND deleted_at IS NULL` | Partial Index — used by the advert eligibility query on every page load |
| `idx_adverts_target_org_uuids` | `target_org_uuids` | **GIN Index** — required for fast reverse-lookup: find all adverts targeting a given `org_uuid` |

---

### `advert_user_suppression`

**Business Purpose:** Per-user advert suppression — tracks which employees have dismissed a specific advert so it is not shown again. Designed to handle **high-concurrency dismissals** safely: the composite unique constraint on `(advert_uuid, membership_uuid)` means concurrent dismissal attempts by the same user (e.g. double-tap) are safely idempotent via INSERT … ON CONFLICT DO NOTHING, without row-locking the main `adverts` table.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `advert_uuid` | `String` | FK → `adverts.uuid`, NOT NULL | The advert the employee has dismissed (`ON DELETE CASCADE`) |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | The employee who dismissed the advert |
| `suppress_advert` | `Boolean` | NOT NULL, default=True | Suppression flag — True means this advert must not be served to this employee; retained as a boolean (rather than just a row presence check) to allow future re-activation flows if needed |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `uq_advert_user_suppression` | `advert_uuid`, `membership_uuid` | Unique — one suppression record per employee per advert; enables safe idempotent upserts |
| `idx_advert_suppression_membership` | `membership_uuid` | Index — used to look up all suppressed adverts for a given employee |

---

### `fact_sheets`

**Business Purpose:** Platform-level health and wellbeing content tiles created exclusively by Bravo staff. Displayed at the bottom of the employee benefit view as curated reading. No `org_uuid` — these are platform-wide records visible to all orgs. Org-level visibility is controlled by `organisations.settings.features.show_fact_sheets` — an organisation either sees **all** fact sheets or **none** (no selective per-team assignment per PRD Module 9).

**Base Class:** `db.Model` + `UserAuditMixin` (platform-level lookup — no org tenancy)

> **PRD Module 9:** Fact sheets are Bravo-only content (client orgs cannot create them). The visibility toggle is all-or-nothing at the org level — there is no per-team or per-benefit selective assignment.

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK, auto-increment | Internal surrogate primary key |
| `uuid` | `String` | Unique | Public identifier used in API responses and deep-links |
| `title` | `String(200)` | NOT NULL | Fact sheet display title shown on the content tile |
| `body_html` | `Text` | NOT NULL, default=`""` | Rich HTML body content rendered on the fact sheet detail page |
| `image_uuid` | `String` | FK → `image.uuid`, Nullable | Hero/thumbnail image UUID from the `image` table (`ON DELETE SET NULL`) |
| `attachment_url` | `String(500)` | Nullable | URL to a downloadable PDF or supplementary attachment |
| `is_active` | `Boolean` | NOT NULL, default=True | Whether the fact sheet is live and visible to employees; False hides it from all orgs |
| `created_at` | `DateTime (tz)` | NOT NULL, default=now | Record creation timestamp |
| `updated_at` | `DateTime (tz)` | NOT NULL, default=now, onupdate=now | Last update timestamp |
| `deleted_at` | `DateTime (tz)` | Nullable | Soft-delete timestamp; NULL = active |
| `created_by` | `String` | FK → `users.uuid`, Nullable | Bravo staff user who created this fact sheet (via `UserAuditMixin`) |
| `updated_by` | `String` | FK → `users.uuid`, Nullable | Bravo staff user who last updated this fact sheet (via `UserAuditMixin`) |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_fact_sheets_active` | `is_active` WHERE `deleted_at IS NULL` | Partial Index — used on every employee portal load to fetch the active fact sheet list |

---

### Domain 5 — Platform Config & Scheduling

---

### `scheduled_jobs`

**Business Purpose:** Background job queue for all time-based platform operations: benefit go-live status promotions, employee onboarding invitations, R&R birthday and long-service sweeps, monthly roundup emails, fortnightly admin emails, and invite expiry checks. Each row represents a single scheduled execution. Workers poll the table for `status = 'pending'` rows whose `scheduled_for` has elapsed, execute the job, and update the row to `done` or `failed`. The `reference_uuid` + `reference_table` pair provides a polymorphic pointer to the subject entity, allowing workers to explicitly resolve the target row without hardcoding lookup logic based on `job_type`.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `job_type` | `String(40)` | NOT NULL, CHECK IN (`benefit_go_live`, `employee_go_live`, `rnr_birthday_sweep`, `rnr_longservice_sweep`, `rnr_monthly_roundup`, `rnr_fortnightly_admin_email`, `invite_expiry_check`) | Type of scheduled job — determines which worker handler processes this row |
| `reference_uuid` | `String` | Nullable | UUID of the related subject entity (e.g. the `org_benefit_uuid` for a `benefit_go_live` job, the `membership_uuid` for an `employee_go_live` job). Paired with `reference_table` to form a polymorphic reference |
| `reference_table` | `String(100)` | Nullable | Table name of the subject entity referenced by `reference_uuid` (e.g. `org_benefits`, `user_org_memberships`). Together with `reference_uuid`, this creates a polymorphic relationship that allows background workers to explicitly know which table to query — without hardcoding table-lookup logic based on `job_type` alone. This decouples job routing from business logic |
| `scheduled_for` | `DateTime (tz)` | NOT NULL | Timestamp (UK timezone) when this job should be executed; the scheduler sweeps for rows where `scheduled_for <= now() AND status = 'pending'` |
| `executed_at` | `DateTime (tz)` | Nullable | Timestamp when the worker actually began executing the job; NULL until picked up |
| `status` | `String(10)` | NOT NULL, default=`pending`, CHECK IN (`pending`, `running`, `done`, `failed`) | Job execution lifecycle: `pending` = waiting to be picked up; `running` = currently being processed by a worker; `done` = completed successfully; `failed` = execution error (see `error_message`) |
| `error_message` | `Text` | Nullable | Full error message and traceback captured when `status = 'failed'`; used for debugging via the Bravo staff job monitor |

> **SQLAlchemy `__table_args__` Note:** The following ENUM columns require `CheckConstraint` entries:
> - `job_type`: `CheckConstraint("job_type IN ('benefit_go_live','employee_go_live','rnr_birthday_sweep','rnr_longservice_sweep','rnr_monthly_roundup','rnr_fortnightly_admin_email','invite_expiry_check')", name='ck_scheduled_jobs_job_type')`
> - `status`: `CheckConstraint("status IN ('pending', 'running', 'done', 'failed')", name='ck_scheduled_jobs_status')`

> **Polymorphic Reference Note (`reference_uuid` + `reference_table`):** These two columns together form an explicit polymorphic pointer. Example: a `benefit_go_live` row carries `reference_uuid = <org_benefits.uuid>` and `reference_table = 'org_benefits'`. The worker reads `reference_table` to determine which model to query, then resolves the row by `reference_uuid`. This avoids brittle `if job_type == 'benefit_go_live': query org_benefits` branches scattered across worker code — the routing information is self-contained in the job row.

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_scheduled_jobs_run` | `scheduled_for` WHERE `status = 'pending'` | Partial Index — used by the scheduler sweep polling query |
| `idx_scheduled_jobs_org` | `org_uuid` | Index — used when listing scheduled jobs for a specific org in the admin UI |

---

### Domain 6 — Analytics

---

### `analytics_events`

**Business Purpose:** Raw event log for all significant user interactions across the platform. Aggregated views and materialised queries serve the self-serve analytics dashboard. **Range-partitioned by `occurred_at` (monthly)** for query performance at scale — each monthly partition is a separate physical table, keeping index scans bounded and VACUUM efficient. Self-serve analytics window is capped at 1–2 years at the application layer to limit partition scan depth.

> ⚡ **Partitioning:** This table uses **PostgreSQL declarative range partitioning** on `occurred_at`. New monthly partitions must be pre-created (via `pg_partman` or a scheduled migration). The composite PK (`id`, `occurred_at`) is **required** by PostgreSQL declarative partitioning — the partition key must be included in the primary key.

**Base Class:** `db.Model` directly — **NOT `Base`**. This avoids audit recursion (writing an audit log event for every analytics event would create an infinite loop via `AuditableEvent`) and eliminates unnecessary overhead (`org_uuid` auto-stamping, soft-delete columns, `UserAuditMixin` FK columns) for a high-volume, append-only log that is never updated or soft-deleted.

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| `id` | `BigInteger` | PK **(composite with `occurred_at`)** — required by PostgreSQL declarative partitioning | Surrogate row key — not globally unique across partitions without the `occurred_at` component |
| `org_uuid` | `String` | FK → `organisations.uuid`, NOT NULL | Tenant scoping — every event is pinned to an org for multi-tenant analytics isolation |
| `membership_uuid` | `String` | FK → `user_org_memberships.uuid`, Nullable | The employee membership performing the action; NULL for system-generated events (e.g. scheduled job completions) |
| `team_uuid` | `String` | FK → `org_teams.uuid`, Nullable | Benefit-access team context at the time of the event, if applicable (e.g. for `benefit_viewed` events where team determines the available benefit set) |
| `event_type` | `String(50)` | NOT NULL, CHECK IN (`employee_registered`, `employee_login`, `benefit_viewed`, `document_downloaded`, `notice_viewed`, `rnr_nomination_created`, `support_ticket_raised`) | Type of tracked interaction — determines which analytics dimension the event rolls up to |
| `entity_uuid` | `String` | Nullable | UUID of the subject entity involved in the event (e.g. the `org_benefit_uuid` for a `benefit_viewed` event, the `document_uuid` for `document_downloaded`). Stored as a UUID string; polymorphic — no FK constraint as the table does not extend `Base` |
| `occurred_at` | `DateTime (tz)` | NOT NULL, default=now | Event timestamp — the **partition key**; all range partition boundaries are defined on this column. Must be included in every INSERT and range-scan query |

> **SQLAlchemy `__table_args__` Note:** `event_type` requires:
> ```python
> CheckConstraint(
>     "event_type IN ('employee_registered','employee_login','benefit_viewed',"
>     "'document_downloaded','notice_viewed','rnr_nomination_created',"
>     "'support_ticket_raised')",
>     name='ck_analytics_events_event_type'
> )
> ```
> The composite PK must be declared as:
> ```python
> PrimaryKeyConstraint('id', 'occurred_at', name='pk_analytics_events')
> ```

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `pk_analytics_events` | `id`, `occurred_at` | Composite Primary Key — required by PostgreSQL declarative partitioning; partition key must be part of the PK |
| `idx_analytics_org_date` | `org_uuid`, `occurred_at DESC` | Index — primary access pattern: all events for an org within a time window |
| `idx_analytics_type_date` | `event_type`, `occurred_at DESC` | Index — secondary access pattern: platform-wide event type rollup |
| *(partitioning)* | `occurred_at` (RANGE, monthly) | Partition Key — one physical child table per calendar month |

#### Partitioning Setup

> ⚡ **Partition Management:** Use `pg_partman` to automate monthly partition creation and retention. Example initial setup:
>
> ```sql
> -- Create the parent partitioned table
> CREATE TABLE analytics_events (
>     id          BIGSERIAL,
>     org_uuid    TEXT NOT NULL,
>     membership_uuid TEXT,
>     team_uuid   TEXT,
>     event_type  TEXT NOT NULL,
>     entity_uuid TEXT,
>     occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
>     PRIMARY KEY (id, occurred_at)
> ) PARTITION BY RANGE (occurred_at);
>
> -- Example: create the May 2026 partition
> CREATE TABLE analytics_events_2026_05
>     PARTITION OF analytics_events
>     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
> ```
>
> Configure `pg_partman` to pre-create 3 months ahead and retain 24 months of history (drop older partitions to manage storage).

---

### `internal_conversations`

**Business Purpose:** Manages the metadata and lifecycle state for company-to-employee broadcast communications. Strictly separate from `support_tickets` — internal conversations are org-admin-initiated broadcasts (e.g. policy announcements, HR updates), whereas support tickets are employee-initiated enquiries to Bravo staff. A conversation starts as a `draft`, is composed, then transitions to `sent` when the org admin dispatches it. Individual message payloads live in `internal_messages`. Requires `organisations.settings.features.is_internal_messages_enabled = true` to be accessible.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `sender_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | Membership UUID of the org admin who created and owns this conversation |
| `subject` | `String(300)` | NOT NULL | Subject line of the broadcast displayed to recipients in their message inbox |
| `status` | `String(10)` | NOT NULL, default=`draft`, CHECK IN (`draft`, `sent`) | Lifecycle status: `draft` = being composed, not yet dispatched; `sent` = dispatched to recipients, body is immutable |

> **SQLAlchemy `__table_args__` Note:** `status` requires:
> `CheckConstraint("status IN ('draft', 'sent')", name='ck_internal_conversations_status')`

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `sender` | `UserOrgMembership` | Many-to-One | The org admin who created the conversation | `ON DELETE RESTRICT` — membership cannot be deleted while conversations exist |
| `messages` | `InternalMessage` | One-to-Many | All message payloads within this conversation thread | `ON DELETE CASCADE` — messages removed when the conversation is deleted |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_internal_conversations_org` | `org_uuid` | Index |
| `idx_internal_conversations_sender` | `sender_uuid` | Index |
| `idx_internal_conversations_status` | `status` WHERE `status = 'draft'` | Partial Index — used to surface draft conversations in the admin compose UI |

---

### `internal_messages`

**Business Purpose:** Stores the individual message payloads within an internal conversation — both the original broadcast body sent by the org admin and any subsequent employee replies within the same thread. Each row is a single message from a single sender. Deleted automatically when the parent `internal_conversations` row is deleted (`ON DELETE CASCADE`). Requires `organisations.settings.features.is_internal_messages_enabled = true`.

**Base Class:** `Base`

#### Columns

| Name | Type | Constraints | Description |
|---|---|---|---|
| *(Base fields)* | — | — | `id`, `uuid`, `org_uuid`, `created_at`, `updated_at`, `deleted_at`, `created_by`, `updated_by`, `deleted_by` |
| `conversation_uuid` | `String` | FK → `internal_conversations.uuid`, NOT NULL | The parent conversation this message belongs to (`ON DELETE CASCADE` — messages are removed when the conversation is deleted) |
| `sender_uuid` | `String` | FK → `user_org_memberships.uuid`, NOT NULL | Membership UUID of the user who sent this specific message — may be the original org admin sender or an employee replying in the thread |
| `body` | `Text` | NOT NULL | Full message body content displayed in the conversation thread view |
| `sent_at` | `DateTime (tz)` | NOT NULL, default=now | Timestamp when the message was sent; used for thread chronological ordering |

#### Relationships

| Name | Target | Type | Description | Cascade Behavior |
|---|---|---|---|---|
| `conversation` | `InternalConversation` | Many-to-One | The parent conversation this message belongs to | `ON DELETE CASCADE` — message is removed with the conversation |
| `sender` | `UserOrgMembership` | Many-to-One | The membership who sent this message | `ON DELETE RESTRICT` — membership cannot be deleted while messages exist |

#### Indexes & Constraints

| Name | Columns | Type |
|---|---|---|
| `idx_internal_messages_conversation` | `conversation_uuid` | Index — used to retrieve all messages for a conversation in chronological order |
| `idx_internal_messages_sender` | `sender_uuid` | Index — used to look up all messages sent by a given membership |

---

## Deleted Tables — Migration Notes

The following tables from the previous schema version have been **removed** and their data migrated into JSONB columns on surviving tables. This section serves as a migration reference for developers writing Alembic migrations.

| Deleted Table | Data Migrated To | Migration Notes |
|---|---|---|
| `two_fa_policy` | `organisations.force_2fa_for_all` (top-level Boolean) | Copy `two_fa_policy.force_2fa_for_all` → `organisations.force_2fa_for_all` per org. The `bravo_admin_always_2fa` flag is now platform-enforced at the application layer — no storage needed. |
| `sso_config` | `organisations.settings['sso']` (JSONB sub-object) | Copy `provider`, `client_id`, `client_secret`, `is_enabled`, `metadata_url` into `settings.sso` per org. Add `tenant_id` (new field, default null). |
| `notification_preferences` | `user_org_memberships.notification_preferences` (JSONB column) | Copy all 6 boolean columns into a JSONB object keyed by column name, one row per membership. |
| `employee_favourite_benefits` | `user_org_memberships.favourite_benefit_uuids` (JSONB Array) | Aggregate `org_benefit_uuid` values per `membership_uuid` into an ordered array (order by `created_at` ASC). |
| `benefit_resources` | `master_benefits.resources` (JSONB Array) | Aggregate all resources per `benefit_uuid` ordered by `display_order` ASC into the array. |
| `benefit_tokens` | `master_benefits.required_tokens` (JSONB Array) | Aggregate all token definitions per `benefit_uuid` into the array. |
| `org_benefit_token_answers` | `org_benefits.token_answers` (JSONB Object) | For each `org_benefit_uuid`, build a `{token_key: answer_value}` map by joining `org_benefit_token_answers` → `benefit_tokens` on `token_uuid`. |
| `custom_benefit_team_access` | `custom_tiles.visible_to_team_uuids` (JSONB Array) | Aggregate `team_uuid` values per `custom_benefit_uuid` into the array. Empty array = visible to all (preserve existing semantics). |
| `rnr_perks` | `rnr_config.predefined_perks` (JSONB Array) | Aggregate all active and inactive perks per `org_uuid` into the array, mapping `perk_name` → `name`, `perk_emoji` → `emoji`, `is_active` → `is_active`. Also backfill `rnr_rewards.predefined_perk_name` and `rnr_rewards.predefined_perk_emoji` by joining `rnr_rewards.perk_uuid → rnr_perks.uuid` for all existing rows, then drop `rnr_rewards.perk_uuid` FK column. |
| `document_team_visibility` | `documents.visible_team_uuids` (JSONB Array) | For each `document_uuid`, aggregate all `team_uuid` values into a JSONB array on `documents.visible_team_uuids`. Documents with no visibility rows should receive an empty array `[]` (meaning visible to all teams — preserves existing semantics). Drop the `document_team_visibility` table after migration. |
| `notice_team_visibility` | `notices.visible_team_uuids` (JSONB Array) | For each `notice_uuid`, aggregate all `team_uuid` values into a JSONB array on `notices.visible_team_uuids`. Notices with no visibility rows receive `[]` (visible to all). Drop the `notice_team_visibility` table after migration. |
| `marketing_benefit_links` | `marketing_assets.linked_benefit_uuids` (JSONB Array) | For each `material_uuid` (now `marketing_assets.uuid`), aggregate all `benefit_uuid` values into the `linked_benefit_uuids` JSONB array. After migration, create GIN index on the column, then drop `marketing_benefit_links`. |
| `marketing_category_materials` | `marketing_assets.category_uuids` (JSONB Array) | For each `material_uuid` (now `marketing_assets.uuid`), aggregate all `category_uuid` values into the `category_uuids` JSONB array. After migration, create GIN index on the column, then drop `marketing_category_materials`. |
| `advert_org_targets` | `adverts.target_org_uuids` (JSONB Array) | For each `advert_uuid`, aggregate all `org_uuid` values into the `target_org_uuids` JSONB array. After migration, create GIN index on the column, then drop `advert_org_targets`. |

---

## Entity Relationship Summary

```
organisations (root tenant)
├── user_org_memberships  ──── users
│   ├── user_org_teams    ──── org_teams
│   ├── notification_preferences  [JSONB on membership]
│   └── favourite_benefit_uuids   [JSONB on membership]
│
├── org_benefits          ──── master_benefits
│   ├── org_benefit_team_levels ──── org_teams + benefit_levels
│   ├── benefit_requests  ──── user_org_memberships
│   └── employee_benefit_status ──── user_org_memberships
│
├── custom_tiles
│   └── visible_to_team_uuids [JSONB array of org_teams.uuid]
│
├── TRS Stack
│   ├── trs_config
│   ├── trs_components     ──── trs_component_types (platform)
│   ├── trs_employee_data  ──── user_org_memberships + trs_components
│   └── trs_statements     ──── user_org_memberships
│
├── R&R Stack
│   ├── rnr_config
│   │   ├── predefined_perks  [JSONB array — replaces rnr_perks table]
│   │   └── rules_data        [JSONB supplement of flat rule columns]
│   ├── rnr_teams
│   │   └── rnr_user_teams    ──── user_org_memberships
│   ├── rnr_values
│   ├── rnr_nominations       ──── user_org_memberships + rnr_values
│   │   └── rnr_rewards (1:1) ──── user_org_memberships
│   ├── rnr_budget_individual ──── user_org_memberships  [append-only ledger]
│   └── rnr_budget_team       ──── rnr_teams             [append-only ledger]
│
└── Content, Support & Marketing Stack
    ├── documents
    │   └── visible_team_uuids [JSONB array of org_teams.uuid — replaces document_team_visibility]
    ├── notices
    │   ├── visible_team_uuids [JSONB array of org_teams.uuid — replaces notice_team_visibility]
    │   ├── like_count         [denormalised counter for fast feed generation]
    │   └── notice_likes       ──── user_org_memberships  [concurrent-write-safe]
    ├── support_tickets        ──── user_org_memberships
    │   └── support_ticket_messages ──── users
    ├── notifications          ──── user_org_memberships  [polymorphic reference]
    ├── internal_conversations ──── user_org_memberships (sender)
    │   └── internal_messages  ──── user_org_memberships (sender, replies) [CASCADE]
    └── adverts (platform-wide)
        ├── target_org_uuids   [JSONB array — replaces advert_org_targets; GIN indexed]
        └── advert_user_suppression ──── user_org_memberships  [high-concurrency safe]

Platform-level lookups (no org_uuid):
  groups, sectors, lead_sources, trs_component_types, image (stock),
  templates, marketing_categories, marketing_assets, fact_sheets

Platform infrastructure (org-scoped or unscoped):
  scheduled_jobs  ──── organisations  [polymorphic ref: reference_uuid + reference_table]
  analytics_events ──── organisations + user_org_memberships + org_teams
                       [range-partitioned by occurred_at — NOT Base, no audit recursion]
```

---

## Migration Order

Execute Alembic migrations in the following dependency order to respect FK constraints:

```
1.  groups
2.  sectors
3.  lead_sources
4.  users                        (no FK deps at creation)
5.  organisations                (FK → groups, sectors, lead_sources, users)
6.  image                        (FK → organisations via org_uuid)
7.  org_teams                    (FK → organisations)
8.  trs_component_types          (no FK deps)
9.  master_benefits              (FK → image)
10. benefit_levels               (FK → master_benefits)
11. user_org_memberships         (FK → users, organisations)
12. user_org_teams               (FK → user_org_memberships, org_teams)
13. org_internal_notes           (FK → organisations)
14. org_benefits                 (FK → master_benefits, organisations)
15. org_benefit_team_levels      (FK → org_benefits, org_teams, benefit_levels)
16. custom_tiles                 (FK → organisations, groups, image)
17. benefit_requests             (FK → org_benefits, user_org_memberships)
18. employee_benefit_status      (FK → user_org_memberships, org_benefits)
19. trs_config                   (FK → organisations)
20. trs_components               (FK → organisations, trs_component_types)
21. trs_employee_data            (FK → user_org_memberships, trs_components)
22. trs_statements               (FK → user_org_memberships)
23. rnr_teams                    (FK → organisations)
24. rnr_config                   (FK → organisations, image)
25. rnr_values                   (FK → organisations, image)
26. rnr_user_teams               (FK → user_org_memberships, rnr_teams)
27. rnr_nominations              (FK → user_org_memberships, rnr_values, users)
28. rnr_rewards                  (FK → rnr_nominations, user_org_memberships)
29. rnr_budget_individual        (FK → organisations, user_org_memberships, users)
30. rnr_budget_team              (FK → organisations, rnr_teams, users)
31. documents                   (FK → organisations, groups)
32. notices                     (FK → organisations, groups, image)
33. notice_likes                (FK → notices, user_org_memberships)
34. support_tickets             (FK → user_org_memberships, org_benefits, users)
35. support_ticket_messages     (FK → support_tickets, users)
36. notifications               (FK → user_org_memberships, organisations)
37. templates                   (no FK deps — platform-level)
38. marketing_categories        (no FK deps — platform-level)
39. marketing_assets            (no FK deps — platform-level; add GIN indexes after data migration)
40. adverts                     (FK → master_benefits; add GIN index on target_org_uuids after data migration)
41. advert_user_suppression     (FK → adverts, user_org_memberships)
42. fact_sheets                 (FK → image)
43. internal_conversations      (FK → user_org_memberships)
44. internal_messages           (FK → internal_conversations, user_org_memberships)
45. scheduled_jobs              (FK → organisations)
46. analytics_events            (FK → organisations, user_org_memberships, org_teams — range-partitioned; configure pg_partman after creation)
```
