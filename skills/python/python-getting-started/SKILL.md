---
name: "python-getting-started"
description: "Payrix Python SDK identity and lookup layer (Python only) — install, import root, base URL/environments, the auth pattern, the SDK map that ships at the SDK root (`sdk-map.md` + `map/operations/`) and how to traverse it, and the module table naming the one file owning each fact the map leaves to the source. Load this before answering any Payrix Python SDK contract question or writing any SDK code."
---

# Getting started with the Payrix Python SDK

> **Who this skill is for.** This is the **lookup layer** for anyone writing Payrix Python SDK code — it is yours to follow directly and fully. Ground every contract fact here (in the SDK map, and in the source modules it names) rather than in recall, and carry those facts onto a contract sheet before you implement. Load `python-integrate-payrix` for the workflow that wraps this skill.

This is the **SDK-specific** entry point. For general patterns that apply to any APIMatic-generated Python SDK (client construction, auth, calling endpoints, models, error handling, resilience, testing), see the companion API-agnostic skills: `python-client-initialization`, `python-authentication`, `python-calling-endpoints`, `python-models`, `python-error-handling`, `python-configuration-resilience` and `python-testing`.

**This page and those companion skills are complementary — load both.** This page is authoritative for the SDK's *identity and surface* (what to install, what to import, the controllers, which module owns which fact); the companion skills are the *usage layer* on top — the best-practice way to call each piece and the gotchas a signature cannot show. Reading a module in the installed package does not remove the need to load the skill for that step, so at each step below, load the companion *and* confirm names against the installed package.

## SDK identity

Verified against `payrix/` and `pyproject.toml` of the generated package at version `1.7.0`. **Re-verify after a version bump** — this page is a snapshot, not a live read.

| Fact | Value |
| --- | --- |
| API | Payrix |
| Distribution name (what you install) | `payrix-gp-apimatic-sdk` |
| Import root (what you import) | `payrix` — not the string you install |
| Source repository | https://github.com/sdks-io/payrix-gp-apimatic-python-sdk |
| Source branch | `main` |
| Version | `1.7.0` |
| Sync client class | `PayrixClient` (alias `Client`) |
| Async client class | `AsyncPayrixClient` (alias `AsyncClient`) |
| Client construction | **keyword-only**: `environment` · `base_url` · `timeout` (default `30.0`) · `retry_options` · `api_key` · `session_key` · `txn_session_key` · `username` · `password`, and the transport override — `custom_http_client` on the sync client, `custom_async_http_client` on the async one (the names differ; see step 1) |
| Auth | **API key** in the `APIKEY` header — set `api_key` · **API key** in the `SESSIONKEY` header — set `session_key` · **API key** in the `TXNSESSIONKEY` header — set `txn_session_key` · **API key** in the `USERNAME` header — set `username` · **API key** in the `PASSWORD` header — set `password` |
| Environments | 3 environments selected by `environment` (default `"production"`), overridable with `base_url` |
| Base-URL config | `ServerConfig` (`payrix/server/server_config.py`), frozen, `extra="forbid"` |
| Python floor | **`>=3.10`** (classifiers list `3.10–3.14`) |
| Runtime dependencies | `httpx (>=0.28.1,<1.0.0)` · `pydantic[email] (>=2.11.0,<3.0.0)` · `typing-extensions (>=4.13.0,<5.0.0)` |
| Typing | ships `py.typed`; the package is checked under `mypy --strict` with `warn_unreachable`. Callers get full inference — **a type error against this SDK is a real contract violation, not noise** |
| Line length / lint | `ruff`, 120 cols (only relevant when editing the SDK itself) |
| Surface | 361 operations across 105 controllers · 781 models · 410 unions · 750 enums · 361 per-operation error unions |

The table above is **orientation, not a copy-paste recipe** — it gives you the names and facts (install, import roots, the auth *pattern*, the base-URL knob), while the actual integration code comes from the companion skills. Load each one as you reach its step (see **Integration workflow** below) and confirm its types against the installed package.

## Install

Into the same environment your project runs in, install the distribution from its package index:

```bash
pip install payrix-gp-apimatic-sdk
python -c "import payrix, pathlib; print(pathlib.Path(payrix.__file__).parent)"
```

**Run the second line too.** A `pip install` that reports success is not proof the SDK is importable where your project runs. If the probe raises `ModuleNotFoundError`, the install went into a different interpreter: re-run it through the one your project uses (`python -m pip install …`) and probe again. If it prints a path whose contents are not this SDK, what you installed is not this distribution.

An index name can belong to an unrelated project — install it from its own repository instead, at <https://github.com/sdks-io/payrix-gp-apimatic-python-sdk>:

```bash
pip install "payrix-gp-apimatic-sdk @ git+https://github.com/sdks-io/payrix-gp-apimatic-python-sdk.git@main"
```

The distribution carries its own `pyproject.toml`, so `pip` builds and installs it like any released package, copied into site-packages. Do not vendor its source into your project, add its directory to `sys.path`, or install it editable (`-e`) from a throwaway clone — an editable install points at that directory, so deleting the clone breaks every import.

Once the second line prints this SDK's directory, write the imports from the table above: the distribution name you install and the package name you import are not the same string. Requires Python 3.10 or newer.

## Imports — the package splits its surface across four modules

Python does not re-export child modules transitively, so `from payrix import models` alone does **not** make enums, error unions, or runtime types reachable. Import each kind of type from the module that owns it.

`payrix/__init__.py` exports exactly 6 names beside the `payrix.models` subpackage it re-exports:

```python
from payrix import (
    AsyncClient,
    AsyncPayrixClient,
    Client,
    Environment,
    PayrixClient,
    ServerConfig,
)
```

Everything else comes from its own subpackage, and the split matters because each kind of type a caller reaches for lives in a different module:

| What you need | Where it lives |
| --- | --- |
| Domain models, their `…Dict` companions | `payrix.models` |
| Enums (and their open `…OrStr`/`…OrInt` aliases) | `payrix.models.enums` |
| `ApiError` · `RawError` · `ApiResult` · `RequestOptions` · `HttpClient` · `SdkBaseModel` · `UNSET` · `Optional` | `payrix.core` |
| Per-operation error *unions* | `payrix.errors` (`DeleteAccountsIdErrorBody`, …) |

`payrix.core` re-exports its whole public surface (a curated `__all__`), so import from `…core` rather than from the private modules beneath it (`…core.results`, `…core.exceptions`, `…core.auth.schemes`).

## Environments

The constructor takes an `environment` keyword — a **string literal alias**, not an enum — followed by `base_url`. The declared environments and the base URL each resolves to:

| `environment=` | Base URL | Hosting |
| --- | --- | --- |
| `"production"` *(default)* | `https://test-api.payrix.com` | — |
| `"environment2"` | `https://test-api.payrixcanada.com` | — |
| `"environment3"` | `{serverUrl}` | — |

Consequences to state on every contract sheet that touches configuration:

- Omitting `environment` gives you **`"production"`**, silently — it is whatever the description listed first, not necessarily production.
- `base_url` overrides the environment's URL entirely when both are passed.
- `ServerConfig` is a frozen pydantic model with `extra="forbid"`: a misspelled keyword raises `ValidationError` at construction rather than being ignored.
- `timeout` is validated too — the base client raises `ValueError` for any non-positive value.

## Auth pattern (five schemes)

An API key sent as the `APIKEY` header, exposed as the client's `api_key=` keyword taking a plain string.

```python
from payrix import Client

client = Client(api_key="<api_key>")
```

An API key sent as the `SESSIONKEY` header, exposed as the client's `session_key=` keyword taking a plain string.

```python
from payrix import Client

client = Client(session_key="<session_key>")
```

An API key sent as the `TXNSESSIONKEY` header, exposed as the client's `txn_session_key=` keyword taking a plain string.

```python
from payrix import Client

client = Client(txn_session_key="<txn_session_key>")
```

An API key sent as the `USERNAME` header, exposed as the client's `username=` keyword taking a plain string.

```python
from payrix import Client

client = Client(username="<username>")
```

An API key sent as the `PASSWORD` header, exposed as the client's `password=` keyword taking a plain string.

```python
from payrix import Client

client = Client(password="<password>")
```

**Every credentials keyword is optional at the type level and that is a trap worth flagging on every sheet.** Omit it and the client is built with `no_auth`: every request goes out unauthenticated. Nothing fails at construction. Most APIs then answer `401`; an API that serves anonymous traffic at all answers `200` and hides the omission entirely — verify the keyword is set rather than waiting for a `401` to tell you.

See `python-authentication` for the full picture, including what a *failed token fetch* raises — it is not what a caller expects, and it is the single most common surprise in this SDK.

## Controllers

Controllers and their operation counts (`client.<attr>`):

| Attribute | Class | Ops | Area |
| --- | --- | --- | --- |
| `client.api_keys` | `ApiKeys` / `AsyncApiKeys` | 5 | `delete_apikeys_id` · `get_apikeys` · `get_apikeys_id` · `post_apikeys` · `put_apikeys_id` |
| `client.accounts` | `Accounts` / `AsyncAccounts` | 5 | `delete_accounts_id` · `get_accounts` · `get_accounts_id` · `post_accounts` · `put_accounts_id` |
| `client.accounts_verifications` | `AccountsVerifications` / `AsyncAccountsVerifications` | 2 | `get_account_verifications` · `get_account_verifications_id` |
| `client.adjustments` | `Adjustments` / `AsyncAdjustments` | 2 | `get_adjustments` · `get_adjustments_id` |
| `client.aggregation_result_groups` | `AggregationResultGroups` / `AsyncAggregationResultGroups` | 3 | `delete_aggregation_result_groups_id` · `get_aggregation_result_groups` · `get_aggregation_result_groups_id` |
| `client.aggregation_results` | `AggregationResults` / `AsyncAggregationResults` | 3 | `delete_aggregation_results_id` · `get_aggregation_results` · `get_aggregation_results_id` |
| `client.aggregations` | `Aggregations` / `AsyncAggregations` | 5 | `delete_aggregations_id` · `get_aggregations` · `get_aggregations_id` · `post_aggregations` · `put_aggregations_id` |
| `client.alert_actions` | `AlertActions` / `AsyncAlertActions` | 5 | `delete_alert_actions_id` · `get_alert_actions` · `get_alert_actions_id` · `post_alert_actions` · `put_alert_actions_id` |
| `client.alert_triggers` | `AlertTriggers` / `AsyncAlertTriggers` | 5 | `delete_alert_triggers_id` · `get_alert_triggers` · `get_alert_triggers_id` · `post_alert_triggers` · `put_alert_triggers_id` |
| `client.alerts` | `Alerts` / `AsyncAlerts` | 5 | `delete_alerts_id` · `get_alerts` · `get_alerts_id` · `post_alerts` · `put_alerts_id` |
| `client.apple_domains` | `AppleDomains` / `AsyncAppleDomains` | 2 | `get_apple_domains` · `get_apple_domains_id` |
| `client.apple_domains_mass_enablement` | `AppleDomainsMassEnablement` / `AsyncAppleDomainsMassEnablement` | 4 | `post_apple_pay_for_csv` · `post_apple_pay_for_merchant_list` · `post_apple_pay_for_merchant_list_disable` · `post_update_apple_pay_domain` |
| `client.authentication_tokens` | `AuthenticationTokens` / `AsyncAuthenticationTokens` | 5 | `delete_auth_tokens_id` · `get_auth_tokens` · `get_auth_tokens_id` · `post_auth_tokens` · `put_auth_tokens_id` |
| `client.batches_settlements` | `BatchesSettlements` / `AsyncBatchesSettlements` | 4 | `get_batches` · `get_batches_id` · `post_batches` · `put_batches_id` |
| `client.bins` | `Bins` / `AsyncBins` | 2 | `get_bins` · `get_bins_id` |
| `client.change_requests` | `ChangeRequests` / `AsyncChangeRequests` | 3 | `delete_change_requests_id` · `get_change_requests` · `get_change_requests_id` |
| `client.chargeback_documents` | `ChargebackDocuments` / `AsyncChargebackDocuments` | 5 | `delete_chargeback_documents_id` · `get_chargeback_documents` · `get_chargeback_documents_id` · `post_chargeback_documents` · `put_chargeback_documents_id` |
| `client.chargeback_message_results` | `ChargebackMessageResults` / `AsyncChargebackMessageResults` | 2 | `get_chargeback_message_results` · `get_chargeback_message_results_id` |
| `client.chargeback_messages` | `ChargebackMessages` / `AsyncChargebackMessages` | 3 | `get_chargeback_messages` · `get_chargeback_messages_id` · `post_chargeback_messages` |
| `client.chargeback_statuses` | `ChargebackStatuses` / `AsyncChargebackStatuses` | 2 | `get_chargeback_statuses` · `get_chargeback_statuses_id` |
| `client.chargebacks` | `Chargebacks` / `AsyncChargebacks` | 2 | `get_chargebacks` · `get_chargebacks_id` |
| `client.confirmation_codes` | `ConfirmationCodes` / `AsyncConfirmationCodes` | 4 | `delete_confirm_codes_id` · `get_confirm_codes` · `get_confirm_codes_id` · `post_confirm_codes` |
| `client.contacts` | `Contacts` / `AsyncContacts` | 5 | `delete_contacts_id` · `get_contacts` · `get_contacts_id` · `post_contacts` · `put_contacts_id` |
| `client.credentials` | `Credentials` / `AsyncCredentials` | 5 | `delete_credentials_id` · `get_credentials` · `get_credentials_id` · `post_credentials` · `put_credentials_id` |
| `client.customers` | `Customers` / `AsyncCustomers` | 5 | `delete_customers_id` · `get_customers` · `get_customers_id` · `post_customers` · `put_customers_id` |
| `client.decision_actions` | `DecisionActions` / `AsyncDecisionActions` | 5 | `delete_decision_actions_id` · `get_decision_actions` · `get_decision_actions_id` · `post_decision_actions` · `put_decision_actions_id` |
| `client.decision_rules` | `DecisionRules` / `AsyncDecisionRules` | 5 | `delete_decision_rules_id` · `get_decision_rules` · `get_decision_rules_id` · `post_decision_rules` · `put_decision_rules_id` |
| `client.decisions` | `Decisions` / `AsyncDecisions` | 5 | `delete_decisions_id` · `get_decisions` · `get_decisions_id` · `post_decisions` · `put_decisions_id` |
| `client.disbursement_entries` | `DisbursementEntries` / `AsyncDisbursementEntries` | 2 | `get_disbursement_entries` · `get_disbursement_entries_id` |
| `client.disbursement_results` | `DisbursementResults` / `AsyncDisbursementResults` | 2 | `get_disbursement_results` · `get_disbursement_results_id` |
| `client.disbursements` | `Disbursements` / `AsyncDisbursements` | 2 | `get_disbursements` · `get_disbursements_id` |
| `client.embedded_finance` | `EmbeddedFinance` / `AsyncEmbeddedFinance` | 2 | `get_embedded_finance` · `get_embedded_finance_id` |
| `client.entities` | `Entities` / `AsyncEntities` | 6 | `delete_entities_id` · `get_entities` · `get_entities_id` · `get_entities_id_next_payout` · `post_entities` · `put_entities_id` |
| `client.entities_custom_fields` | `EntitiesCustomFields` / `AsyncEntitiesCustomFields` | 5 | `delete_entity_custom_fields_id` · `get_entity_custom_fields` · `get_entity_custom_fields_id` · `post_entity_custom_fields` · `put_entity_custom_fields_id` |
| `client.entity_debts` | `EntityDebts` / `AsyncEntityDebts` | 2 | `get_entity_debts` · `get_entity_debts_id` |
| `client.entity_returns` | `EntityReturns` / `AsyncEntityReturns` | 2 | `get_entity_returns` · `get_entity_returns_id` |
| `client.entity_terms` | `EntityTerms` / `AsyncEntityTerms` | 3 | `get_entity_terms` · `get_entity_terms_id` · `post_entity_terms` |
| `client.entity_data` | `EntityData` / `AsyncEntityData` | 3 | `get_entity_datas` · `get_entity_datas_id` · `post_entity_datas` |
| `client.entries` | `Entries` / `AsyncEntries` | 2 | `get_entries` · `get_entries_id` |
| `client.entry_origin` | `EntryOrigin` / `AsyncEntryOrigin` | 2 | `get_entry_origins` · `get_entry_origins_id` |
| `client.fraud_sight_enablements` | `FraudSightEnablements` / `AsyncFraudSightEnablements` | 2 | `get_fraud_sight_enablements` · `get_fraud_sight_enablements_id` |
| `client.fraud_sight_results` | `FraudSightResults` / `AsyncFraudSightResults` | 2 | `get_fraud_sight_results` · `get_fraud_sight_results_id` |
| `client.fund_origins` | `FundOrigins` / `AsyncFundOrigins` | 2 | `get_fund_origins` · `get_fund_origins_id` |
| `client.funds` | `Funds` / `AsyncFunds` | 2 | `get_funds` · `get_funds_id` |
| `client.host_themes` | `HostThemes` / `AsyncHostThemes` | 2 | `get_host_themes` · `get_host_themes_id` |
| `client.ip_address_lists` | `IpAddressLists` / `AsyncIpAddressLists` | 5 | `delete_iplists_id` · `get_iplists` · `get_iplists_id` · `post_iplists` · `put_iplists_id` |
| `client.invoice_items` | `InvoiceItems` / `AsyncInvoiceItems` | 5 | `delete_invoice_items_id` · `get_invoice_items` · `get_invoice_items_id` · `post_invoice_items` · `put_invoice_items_id` |
| `client.invoice_line_items` | `InvoiceLineItems` / `AsyncInvoiceLineItems` | 5 | `delete_invoice_line_items_id` · `get_invoice_line_items` · `get_invoice_line_items_id` · `post_invoice_line_items` · `put_invoice_line_items_id` |
| `client.invoice_parameters` | `InvoiceParameters` / `AsyncInvoiceParameters` | 5 | `delete_invoice_parameters_id` · `get_invoice_parameters` · `get_invoice_parameters_id` · `post_invoice_parameters` · `put_invoice_parameters_id` |
| `client.invoice_results` | `InvoiceResults` / `AsyncInvoiceResults` | 3 | `get_invoice_results` · `get_invoice_results_id` · `post_invoice_results` |
| `client.invoices` | `Invoices` / `AsyncInvoices` | 5 | `delete_invoices_id` · `get_invoices` · `get_invoices_id` · `post_invoices` · `put_invoices_id` |
| `client.logins` | `Logins` / `AsyncLogins` | 5 | `delete_logins_id` · `get_logins` · `get_logins_id` · `post_logins` · `put_logins_id` |
| `client.logins_helpers` | `LoginsHelpers` / `AsyncLoginsHelpers` | 3 | `get_logins_helpers` · `get_logins_helpers_id` · `put_logins_helpers_id` |
| `client.mappings` | `Mappings` / `AsyncMappings` | 5 | `delete_mappings_id` · `get_mappings` · `get_mappings_id` · `post_mappings` · `put_mappings_id` |
| `client.members` | `Members` / `AsyncMembers` | 5 | `delete_members_id` · `get_members` · `get_members_id` · `post_members` · `put_members_id` |
| `client.merchant_platform_statuses` | `MerchantPlatformStatuses` / `AsyncMerchantPlatformStatuses` | 2 | `get_merchant_platform_statuses` · `get_merchant_platform_statuses_id` |
| `client.merchant_results` | `MerchantResults` / `AsyncMerchantResults` | 2 | `get_merchant_results` · `get_merchant_results_id` |
| `client.merchants` | `Merchants` / `AsyncMerchants` | 3 | `get_merchants` · `get_merchants_id` · `put_merchants_id` |
| `client.message_threads` | `MessageThreads` / `AsyncMessageThreads` | 3 | `get_message_threads` · `get_message_threads_id` · `put_message_threads_id` |
| `client.messages` | `Messages` / `AsyncMessages` | 4 | `get_messages` · `get_messages_id` · `post_messages` · `put_messages_id` |
| `client.note_documents` | `NoteDocuments` / `AsyncNoteDocuments` | 5 | `delete_note_documents_id` · `get_note_documents` · `get_note_documents_id` · `post_note_documents` · `put_note_documents_id` |
| `client.notes` | `Notes` / `AsyncNotes` | 5 | `delete_notes_id` · `get_notes` · `get_notes_id` · `post_notes` · `put_notes_id` |
| `client.omni_tokens` | `OmniTokens` / `AsyncOmniTokens` | 2 | `get_omni_tokens` · `get_omni_tokens_id` |
| `client.opt_outs` | `OptOuts` / `AsyncOptOuts` | 1 | `post_optouts` |
| `client.org_entities` | `OrgEntities` / `AsyncOrgEntities` | 5 | `delete_org_entities_id` · `get_org_entities` · `get_org_entities_id` · `post_org_entities` · `put_org_entities_id` |
| `client.org_flow_rules` | `OrgFlowRules` / `AsyncOrgFlowRules` | 5 | `delete_org_flow_rules_id` · `get_org_flow_rules` · `get_org_flow_rules_id` · `post_org_flow_rules` · `put_org_flow_rules_id` |
| `client.orgs` | `Orgs` / `AsyncOrgs` | 5 | `delete_orgs_id` · `get_orgs` · `get_orgs_id` · `post_orgs` · `put_orgs_id` |
| `client.orgs_vas_revenue_boosts` | `OrgsVasRevenueBoosts` / `AsyncOrgsVasRevenueBoosts` | 2 | `get_orgs_vas_revenue_boosts` · `get_orgs_vas_revenue_boosts_id` |
| `client.payment_update_groups` | `PaymentUpdateGroups` / `AsyncPaymentUpdateGroups` | 2 | `get_payment_update_groups` · `get_payment_update_groups_id` |
| `client.payment_updates` | `PaymentUpdates` / `AsyncPaymentUpdates` | 2 | `get_payment_updates` · `get_payment_updates_id` |
| `client.payouts` | `Payouts` / `AsyncPayouts` | 5 | `delete_payouts_id` · `get_payouts` · `get_payouts_id` · `post_payouts` · `put_payouts_id` |
| `client.pending_entry_api` | `PendingEntryApi` / `AsyncPendingEntryApi` | 2 | `get_pending_entries` · `get_pending_entries_id` |
| `client.pinless_debit_conversions` | `PinlessDebitConversions` / `AsyncPinlessDebitConversions` | 2 | `get_pinless_debit_conversions` · `get_pinless_debit_conversions_id` |
| `client.plans` | `Plans` / `AsyncPlans` | 5 | `delete_plans_id` · `get_plans` · `get_plans_id` · `post_plans` · `put_plans_id` |
| `client.profit_share_results` | `ProfitShareResults` / `AsyncProfitShareResults` | 2 | `get_profit_share_results` · `get_profit_share_results_id` |
| `client.rev_share_schedules` | `RevShareSchedules` / `AsyncRevShareSchedules` | 5 | `delete_rev_share_schedules_id` · `get_rev_share_schedules` · `get_rev_share_schedules_id` · `post_rev_share_schedules` · `put_rev_share_schedules_id` |
| `client.rev_share_statements` | `RevShareStatements` / `AsyncRevShareStatements` | 2 | `get_rev_share_statements` · `get_rev_share_statements_id` |
| `client.revenue_boosts` | `RevenueBoosts` / `AsyncRevenueBoosts` | 2 | `get_revenue_boosts` · `get_revenue_boosts_id` |
| `client.safer_payments` | `SaferPayments` / `AsyncSaferPayments` | 2 | `get_safer_payments` · `get_safer_payments_id` |
| `client.secrets` | `Secrets` / `AsyncSecrets` | 5 | `delete_secrets_id` · `get_secrets` · `get_secrets_id` · `post_secrets` · `put_secrets_id` |
| `client.sessions` | `Sessions` / `AsyncSessions` | 4 | `delete_sessions_id` · `get_sessions` · `get_sessions_id` · `post_sessions` |
| `client.statement_entry` | `StatementEntry` / `AsyncStatementEntry` | 2 | `get_statement_entries` · `get_statement_entries_id` |
| `client.statements` | `Statements` / `AsyncStatements` | 2 | `get_statements` · `get_statements_id` |
| `client.subscription_tokens` | `SubscriptionTokens` / `AsyncSubscriptionTokens` | 5 | `delete_subscription_tokens_id` · `get_subscription_tokens` · `get_subscription_tokens_id` · `post_subscription_tokens` · `put_subscription_tokens_id` |
| `client.subscriptions` | `Subscriptions` / `AsyncSubscriptions` | 5 | `delete_subscriptions_id` · `get_subscriptions` · `get_subscriptions_id` · `post_subscriptions` · `put_subscriptions_id` |
| `client.tax_form_requests` | `TaxFormRequests` / `AsyncTaxFormRequests` | 3 | `get_tax_form_requests` · `get_tax_form_requests_id` · `post_tax_form_requests` |
| `client.team_logins` | `TeamLogins` / `AsyncTeamLogins` | 5 | `delete_team_logins_id` · `get_team_logins` · `get_team_logins_id` · `post_team_logins` · `put_team_logins_id` |
| `client.teams` | `Teams` / `AsyncTeams` | 5 | `delete_teams_id` · `get_teams` · `get_teams_id` · `post_teams` · `put_teams_id` |
| `client.terminal_transaction_reference` | `TerminalTransactionReference` / `AsyncTerminalTransactionReference` | 1 | `post_terminal_txn_refs` |
| `client.terminal_transaction_results` | `TerminalTransactionResults` / `AsyncTerminalTransactionResults` | 2 | `get_terminal_txn_results` · `get_terminal_txn_results_id` |
| `client.terminal_transactions` | `TerminalTransactions` / `AsyncTerminalTransactions` | 5 | `delete_terminal_txns_by_id` · `get_terminal_txns` · `get_terminal_txns_by_id` · `post_terminal_txns` · `update_terminal_txns_by_id` |
| `client.terminal_transactions_datas` | `TerminalTransactionsDatas` / `AsyncTerminalTransactionsDatas` | 3 | `get_terminal_txns_datas` · `get_terminal_txns_datas_id` · `post_terminal_txns_datas` |
| `client.terminal_transactions_metadatas` | `TerminalTransactionsMetadatas` / `AsyncTerminalTransactionsMetadatas` | 3 | `get_terminal_txns_metadatas` · `get_terminal_txns_metadatas_id` · `post_terminal_txns_metadatas` |
| `client.terminals` | `Terminals` / `AsyncTerminals` | 5 | `delete_terminals_id` · `get_terminals` · `get_terminals_id` · `post_terminals` · `put_terminals_id` |
| `client.token_results` | `TokenResults` / `AsyncTokenResults` | 2 | `get_token_results` · `get_token_results_id` |
| `client.tokens` | `Tokens` / `AsyncTokens` | 5 | `delete_tokens_id` · `get_tokens` · `get_tokens_id` · `post_tokens` · `put_tokens_id` |
| `client.transaction_datas` | `TransactionDatas` / `AsyncTransactionDatas` | 3 | `get_txn_datas` · `get_txn_datas_id` · `post_txn_datas` |
| `client.transaction_holds` | `TransactionHolds` / `AsyncTransactionHolds` | 4 | `get_holds` · `get_holds_id` · `post_holds` · `put_holds_id` |
| `client.transaction_items` | `TransactionItems` / `AsyncTransactionItems` | 5 | `delete_items_id` · `get_items` · `get_items_id` · `post_items` · `put_items_id` |
| `client.transaction_metadatas` | `TransactionMetadatas` / `AsyncTransactionMetadatas` | 2 | `get_txn_metadatas` · `get_txn_metadatas_id` |
| `client.transactions_txns` | `TransactionsTxns` / `AsyncTransactionsTxns` | 4 | `get_txns` · `get_txns_id` · `post_txns` · `put_txns_id` |
| `client.transactions_results` | `TransactionsResults` / `AsyncTransactionsResults` | 2 | `get_txn_results` · `get_txn_results_id` |
| `client.txn_sessions` | `TxnSessions` / `AsyncTxnSessions` | 4 | `delete_txn_sessions_id` · `get_txn_sessions` · `get_txn_sessions_id` · `post_txn_sessions` |
| `client.vas_efe_offers` | `VasEfeOffers` / `AsyncVasEfeOffers` | 2 | `get_vas_efe_offers` · `get_vas_efe_offers_id` |
| `client.value_added_services` | `ValueAddedServices` / `AsyncValueAddedServices` | 1 | `get_value_added_services_enablement_status_target_id` |

Every controller has an `Async…` peer whose operations are identical in name and parameters and differ solely by being awaited. Do not emit a separate row for an async operation; state the rule once on the sheet.

The SDK map's controller table carries the same counts and links to a page per controller (`map/operations/<controller>.md`) — go there for the operations themselves.

## SDK map — look up first, open the module second

The SDK ships a generated map at its **root** — the directory holding `pyproject.toml`, the `payrix/` source directory, and these two entries:

- **`sdk-map.md`** — the index: client construction with the full constructor-keyword table, the error-handling model (`ApiError` / `ApiResult` / `RawError`, Case A vs Case B), where models, enums and error aliases live, servers and auth, and the link table into the operations pages.
- **`map/operations/<controller>.md`** — one page per controller, one `###` block per operation: the HTTP verb and route, the sync parsed signature, each parameter's role and wire name, both return types, the error alias with the status each arm maps from, and a **Type sources** table naming the module that declares every *generated* type the operation mentions.

**Installing the distribution does not give you the map.** `pyproject.toml` ships the `payrix/` package only, so `sdk-map.md` and `map/operations/` are absent from the installed package — they live in the SDK's own source tree, at the root of its repository, <https://github.com/sdks-io/payrix-gp-apimatic-python-sdk>, in lockstep with the code the map describes — clone it to a temporary directory, outside the project repo:

```bash
git clone --depth 1 --branch main https://github.com/sdks-io/payrix-gp-apimatic-python-sdk
```

Confirm the map comes from the release you installed before you read anything out of it: the clone above is pinned to `main`, the branch this SDK is released from — the repository's default branch may carry a different version. A map read from any other release describes code you do not have.

Every `Source` path on the map is relative to that SDK root, so `payrix/models/account.py` opens as written from there — and the same path resolves inside the installed package, which is where you read a module's body once the map has named it.

**The map is the locator; the source modules are the shapes.** Read the map first — signatures, routes, parameter roles, return types, error unions, and which module declares a type are all answered there without opening a single `.py` file. Then open the one module the map names for what it deliberately does not carry: a model's members, an enum's values, a field's wire alias. The map says so itself — *"Shapes live only in the source … Never grep for a type."*

**The map carries shapes; what an operation *means* lives elsewhere.** When *what* to pass depends on meaning — which values a field accepts beyond its type, a rule that couples two fields, what a defaulted parameter actually selects — the map will not settle it. Open the operation's docstring in `payrix/apis/<controller>.py` (or `client.py` where operations sit on the client) and `api-reference.md` at the SDK root for that operation *before* writing the sheet row, and record what you found there. A value you already "know" for a field the map types as a plain `str` is a lookup, not a recall — the memory ban applies to it.

`sdk-map.md` carries the invariants every operation block assumes, so load it before any `map/operations/` page; the pages are written to be read beside it.

## Contract facts — the map first, then the module

**Six of these are map lookups — don't open the module for them:** an operation's signature, parameters and return type; the client's constructor keywords; the `timeout` default; the members of `ApiError` / `Success` / `Failure` / `RawError`; an operation's error union and the status each arm maps from; base-URL and auth wiring. The table below covers everything else, and the full body behind a map row.

Read the one module that owns the fact **inside the installed package**. Locate it first:

```bash
python -c "import payrix, pathlib; print(pathlib.Path(payrix.__file__).parent)"
```

Failing that, it is under the project's environment (`.venv/Lib/site-packages/payrix` on Windows, `.venv/lib/python3.*/site-packages/payrix` elsewhere). **If the package is not installed, install it first** — see *Install* above; until then mark the fact `UNVERIFIED` and say what would settle it rather than answering from memory. Paths below are relative to that package root:

| Question | Module |
| --- | --- |
| An operation's real signature, parameters and return type | `apis/<controller>.py` |
| Client construction, keywords, controller wiring | `client.py`, `async_client.py`, `base_client.py` |
| Timeout default and validation | `base_client.py` (`DEFAULT_TIMEOUT = 30.0`) |
| The request/response pipeline, 401 handling, 2xx-vs-error split | `core/raw_client.py` |
| Exception shape (`ApiError.error`, `.status_code`, `.headers`) | `core/exceptions.py` |
| `Success`/`Failure`/`RawError` | `core/results.py` |
| Per-call overrides | `core/request_options.py` |
| `UNSET`, `Optional`, `OptionalNullable` | `core/optionality.py` |
| Model base config, `to_dict`/`to_json` | `core/models.py` |
| A model's members, required vs `UNSET`, wire aliases | `models/<model_name>.py` |
| An enum's members and wire values | `models/enums/` |
| Open-enum coercion | `core/converters/open_enum.py` |
| Date/time wire formats — `Date`, `RFC3339DateTime`, `RFC1123DateTime`, `UnixSecondsDateTime` (`Annotated` aliases over `datetime.date` / `datetime.datetime`; not in the map's Type sources) | `core/converters/date_time.py` |
| Transport protocols (the test seam) | `core/transport.py` |
| httpx adapter, proxy/TLS knobs | `core/httpx_transport.py` |
| Token fetch, credential placement | `core/auth/`, `core/auth/models.py` |
| Base-URL resolution | `server/server_config.py`, `server/server.py` |
| An operation's error mapper (status → schema) | `errors/<operation>_error.py` |

**Read scoped.** These modules carry long design docstrings; `grep -n` for the symbol and read the surrounding lines rather than whole files. Never quote a docstring's design rationale onto a contract sheet — the sheet carries facts an implementer must obey, not the reasoning behind them.

Keep lookups cheap — the rules that keep a session's context small:

- Collect the contracts for **every** in-scope operation in **one** pass — signature, required members with wire aliases, the error union, enum values — into a short **contract sheet** in your plan or working notes, then implement from the sheet. Don't re-open a module per member, and never re-look-up a fact the sheet already carries.
- Recurse into a model's members only where the task actually sets them — a full transitive expansion is hundreds of rows and nobody needs it.
- Trust the interpreter over this page: if a name here ever fails to type-check or import, re-read the module the table above names and report the drift; never patch around it from memory.

## Integration workflow — load the companion skill at each step

Before you write the code for each step, load the named companion skill — even if you have already read the relevant module. Each step calls out the trap the signature hides (in *parens*). A typical integration reaches them in this order:

1. **Client construction & lifetime** — load **python-client-initialization** before you write `Client(...)` or `AsyncClient(...)`. (*The signature won't tell you:* the constructor is keyword-only, so nothing can be passed positionally; the client owns an `httpx` connection pool and you **must** `close()` (sync) or `await aclose()` (async) or use it as a context manager; it must be long-lived and module- or app-scoped, never rebuilt per request; the sync and async clients do not mix; and the transport-override keyword differs by client — `custom_http_client` vs `custom_async_http_client`.)
2. **Authentication** — load **python-authentication** before you set credentials. The five schemes are `api_key=`, `session_key=`, `txn_session_key=`, `username=` and `password=`. (*The signature won't tell you:* every credentials keyword is *optional* — omit it and every request goes out unauthenticated, with no failure at construction and not necessarily a `401` to tell you; Load secrets from the environment or a secret store, never hardcode.)
3. **Calling an endpoint** — load **python-calling-endpoints** before the first `client.<controller>.<operation>(...)` call. (*The signature won't tell you:* every operation splits positional path params (and sometimes the body) from a keyword-only tail after `*`; every keyword-only parameter has a **real** default, so there is no "must pass `None` explicitly" hazard; and the two response modes — raising vs `ApiResult` — behave differently on failure.)
4. **Models** — load **python-models** the moment a request/response member is not a plain string or number. (*The signature won't tell you:* `Optional[T]` here is `T | UnsetType`, **not** `typing.Optional` — `None` is not a legal value for it; models are frozen pydantic instances with `…Dict` TypedDict companions; enums are **open** (`…OrStr`/`…OrInt`), so an unknown wire value passes through as a plain `str`/`int` rather than raising; wire aliases differ from Python member names; unknown response fields are **preserved**, not dropped; and serialize via `to_dict`/`to_json`.)
5. **Error handling** — load **python-error-handling** before you write any `try/except`. (*The signature won't tell you:* there is a single `ApiError` type whose `.error` is a **per-operation union**, and a decode failure raises `ValidationError`/`ValueError`, not `ApiError`, and bypasses both response modes; `httpx` transport exceptions reach your boundary unwrapped.)
6. **Configuration & resilience** — load **python-configuration-resilience** when you set the base URL, timeouts, retries, proxies, TLS, or logging. (*The signature won't tell you:* **retrying is on by default** — `retry_options` takes a count, a `RetryOptions` or its dict, `max_retries` set to `0` turns it off, and one call overrides it through `request_options`; a `401` is never retried; `timeout` defaults to `30.0` and is a single float that maps onto `httpx`'s timeout semantics, bounding one attempt rather than the whole call; and there is no logging hook — you wrap the transport seam.)
7. **Testing** — load **python-testing** before you stub the SDK. (*The signature won't tell you:* the seam is the **transport protocol** (`HttpClient`/`AsyncHttpClient` in `core/transport.py`) passed as `custom_http_client`, or `respx` at the `httpx` layer — not the client class; a fake transport implements `send` and `close`, the two the protocol declares, and `send` answers with the head alone; assert on the request the SDK actually built, and cover all four failure kinds, decode failures included.)

## What a contract sheet must carry for this SDK

Beyond the usual signatures and model members, a Python sheet is incomplete without these, because each one is a decision the implementer cannot make correctly from the signature alone:

1. **Sync or async** — which client class, and the reminder that the two do not mix. Plus the `close()`/`aclose()` obligation and where the client is held.
2. **The keyword-only boundary** for each operation: what is positional (path params, sometimes the body) and what sits after `*`. Every keyword-only parameter has a real default, so there is no "must pass `None` explicitly" hazard — say so, so nobody writes defensive `None`s.
3. **Every operation returns a payload** — no operation in this SDK returns `None`, so the `with_raw_response` peer is needed only where the status code itself matters.
4. **Required vs `UNSET`** for every model member the task sets, and the fact that `Optional[T]` here is `T | UnsetType` — **not** `typing.Optional`, so `None` is not a legal value for it.
5. **The `ApiError.error` union** for each operation in scope — there is **one** typed error body in this SDK, so the union is never uniform. Every union is `<Typed> | RawError`:
   1. `ErrorFourHundred1` — 361 operations (`api_keys.delete_apikeys_id` · `api_keys.get_apikeys` · `api_keys.get_apikeys_id` · `api_keys.post_apikeys` · `api_keys.put_apikeys_id` · `accounts.delete_accounts_id` · `accounts.get_accounts` · `accounts.get_accounts_id` · `accounts.post_accounts` · `accounts.put_accounts_id` · `accounts_verifications.get_account_verifications` · `accounts_verifications.get_account_verifications_id` · `adjustments.get_adjustments` · `adjustments.get_adjustments_id` · `aggregation_result_groups.delete_aggregation_result_groups_id` · `aggregation_result_groups.get_aggregation_result_groups` · `aggregation_result_groups.get_aggregation_result_groups_id` · `aggregation_results.delete_aggregation_results_id` · `aggregation_results.get_aggregation_results` · `aggregation_results.get_aggregation_results_id` · `aggregations.delete_aggregations_id` · `aggregations.get_aggregations` · `aggregations.get_aggregations_id` · `aggregations.post_aggregations` · `aggregations.put_aggregations_id` · `alert_actions.delete_alert_actions_id` · `alert_actions.get_alert_actions` · `alert_actions.get_alert_actions_id` · `alert_actions.post_alert_actions` · `alert_actions.put_alert_actions_id` · `alert_triggers.delete_alert_triggers_id` · `alert_triggers.get_alert_triggers` · `alert_triggers.get_alert_triggers_id` · `alert_triggers.post_alert_triggers` · `alert_triggers.put_alert_triggers_id` · `alerts.delete_alerts_id` · `alerts.get_alerts` · `alerts.get_alerts_id` · `alerts.post_alerts` · `alerts.put_alerts_id` · `apple_domains.get_apple_domains` · `apple_domains.get_apple_domains_id` · `apple_domains_mass_enablement.post_apple_pay_for_csv` · `apple_domains_mass_enablement.post_apple_pay_for_merchant_list` · `apple_domains_mass_enablement.post_apple_pay_for_merchant_list_disable` · `apple_domains_mass_enablement.post_update_apple_pay_domain` · `authentication_tokens.delete_auth_tokens_id` · `authentication_tokens.get_auth_tokens` · `authentication_tokens.get_auth_tokens_id` · `authentication_tokens.post_auth_tokens` · `authentication_tokens.put_auth_tokens_id` · `batches_settlements.get_batches` · `batches_settlements.get_batches_id` · `batches_settlements.post_batches` · `batches_settlements.put_batches_id` · `bins.get_bins` · `bins.get_bins_id` · `change_requests.delete_change_requests_id` · `change_requests.get_change_requests` · `change_requests.get_change_requests_id` · `chargeback_documents.delete_chargeback_documents_id` · `chargeback_documents.get_chargeback_documents` · `chargeback_documents.get_chargeback_documents_id` · `chargeback_documents.post_chargeback_documents` · `chargeback_documents.put_chargeback_documents_id` · `chargeback_message_results.get_chargeback_message_results` · `chargeback_message_results.get_chargeback_message_results_id` · `chargeback_messages.get_chargeback_messages` · `chargeback_messages.get_chargeback_messages_id` · `chargeback_messages.post_chargeback_messages` · `chargeback_statuses.get_chargeback_statuses` · `chargeback_statuses.get_chargeback_statuses_id` · `chargebacks.get_chargebacks` · `chargebacks.get_chargebacks_id` · `confirmation_codes.delete_confirm_codes_id` · `confirmation_codes.get_confirm_codes` · `confirmation_codes.get_confirm_codes_id` · `confirmation_codes.post_confirm_codes` · `contacts.delete_contacts_id` · `contacts.get_contacts` · `contacts.get_contacts_id` · `contacts.post_contacts` · `contacts.put_contacts_id` · `credentials.delete_credentials_id` · `credentials.get_credentials` · `credentials.get_credentials_id` · `credentials.post_credentials` · `credentials.put_credentials_id` · `customers.delete_customers_id` · `customers.get_customers` · `customers.get_customers_id` · `customers.post_customers` · `customers.put_customers_id` · `decision_actions.delete_decision_actions_id` · `decision_actions.get_decision_actions` · `decision_actions.get_decision_actions_id` · `decision_actions.post_decision_actions` · `decision_actions.put_decision_actions_id` · `decision_rules.delete_decision_rules_id` · `decision_rules.get_decision_rules` · `decision_rules.get_decision_rules_id` · `decision_rules.post_decision_rules` · `decision_rules.put_decision_rules_id` · `decisions.delete_decisions_id` · `decisions.get_decisions` · `decisions.get_decisions_id` · `decisions.post_decisions` · `decisions.put_decisions_id` · `disbursement_entries.get_disbursement_entries` · `disbursement_entries.get_disbursement_entries_id` · `disbursement_results.get_disbursement_results` · `disbursement_results.get_disbursement_results_id` · `disbursements.get_disbursements` · `disbursements.get_disbursements_id` · `embedded_finance.get_embedded_finance` · `embedded_finance.get_embedded_finance_id` · `entities.delete_entities_id` · `entities.get_entities` · `entities.get_entities_id` · `entities.get_entities_id_next_payout` · `entities.post_entities` · `entities.put_entities_id` · `entities_custom_fields.delete_entity_custom_fields_id` · `entities_custom_fields.get_entity_custom_fields` · `entities_custom_fields.get_entity_custom_fields_id` · `entities_custom_fields.post_entity_custom_fields` · `entities_custom_fields.put_entity_custom_fields_id` · `entity_debts.get_entity_debts` · `entity_debts.get_entity_debts_id` · `entity_returns.get_entity_returns` · `entity_returns.get_entity_returns_id` · `entity_terms.get_entity_terms` · `entity_terms.get_entity_terms_id` · `entity_terms.post_entity_terms` · `entity_data.get_entity_datas` · `entity_data.get_entity_datas_id` · `entity_data.post_entity_datas` · `entries.get_entries` · `entries.get_entries_id` · `entry_origin.get_entry_origins` · `entry_origin.get_entry_origins_id` · `fraud_sight_enablements.get_fraud_sight_enablements` · `fraud_sight_enablements.get_fraud_sight_enablements_id` · `fraud_sight_results.get_fraud_sight_results` · `fraud_sight_results.get_fraud_sight_results_id` · `fund_origins.get_fund_origins` · `fund_origins.get_fund_origins_id` · `funds.get_funds` · `funds.get_funds_id` · `host_themes.get_host_themes` · `host_themes.get_host_themes_id` · `ip_address_lists.delete_iplists_id` · `ip_address_lists.get_iplists` · `ip_address_lists.get_iplists_id` · `ip_address_lists.post_iplists` · `ip_address_lists.put_iplists_id` · `invoice_items.delete_invoice_items_id` · `invoice_items.get_invoice_items` · `invoice_items.get_invoice_items_id` · `invoice_items.post_invoice_items` · `invoice_items.put_invoice_items_id` · `invoice_line_items.delete_invoice_line_items_id` · `invoice_line_items.get_invoice_line_items` · `invoice_line_items.get_invoice_line_items_id` · `invoice_line_items.post_invoice_line_items` · `invoice_line_items.put_invoice_line_items_id` · `invoice_parameters.delete_invoice_parameters_id` · `invoice_parameters.get_invoice_parameters` · `invoice_parameters.get_invoice_parameters_id` · `invoice_parameters.post_invoice_parameters` · `invoice_parameters.put_invoice_parameters_id` · `invoice_results.get_invoice_results` · `invoice_results.get_invoice_results_id` · `invoice_results.post_invoice_results` · `invoices.delete_invoices_id` · `invoices.get_invoices` · `invoices.get_invoices_id` · `invoices.post_invoices` · `invoices.put_invoices_id` · `logins.delete_logins_id` · `logins.get_logins` · `logins.get_logins_id` · `logins.post_logins` · `logins.put_logins_id` · `logins_helpers.get_logins_helpers` · `logins_helpers.get_logins_helpers_id` · `logins_helpers.put_logins_helpers_id` · `mappings.delete_mappings_id` · `mappings.get_mappings` · `mappings.get_mappings_id` · `mappings.post_mappings` · `mappings.put_mappings_id` · `members.delete_members_id` · `members.get_members` · `members.get_members_id` · `members.post_members` · `members.put_members_id` · `merchant_platform_statuses.get_merchant_platform_statuses` · `merchant_platform_statuses.get_merchant_platform_statuses_id` · `merchant_results.get_merchant_results` · `merchant_results.get_merchant_results_id` · `merchants.get_merchants` · `merchants.get_merchants_id` · `merchants.put_merchants_id` · `message_threads.get_message_threads` · `message_threads.get_message_threads_id` · `message_threads.put_message_threads_id` · `messages.get_messages` · `messages.get_messages_id` · `messages.post_messages` · `messages.put_messages_id` · `note_documents.delete_note_documents_id` · `note_documents.get_note_documents` · `note_documents.get_note_documents_id` · `note_documents.post_note_documents` · `note_documents.put_note_documents_id` · `notes.delete_notes_id` · `notes.get_notes` · `notes.get_notes_id` · `notes.post_notes` · `notes.put_notes_id` · `omni_tokens.get_omni_tokens` · `omni_tokens.get_omni_tokens_id` · `opt_outs.post_optouts` · `org_entities.delete_org_entities_id` · `org_entities.get_org_entities` · `org_entities.get_org_entities_id` · `org_entities.post_org_entities` · `org_entities.put_org_entities_id` · `org_flow_rules.delete_org_flow_rules_id` · `org_flow_rules.get_org_flow_rules` · `org_flow_rules.get_org_flow_rules_id` · `org_flow_rules.post_org_flow_rules` · `org_flow_rules.put_org_flow_rules_id` · `orgs.delete_orgs_id` · `orgs.get_orgs` · `orgs.get_orgs_id` · `orgs.post_orgs` · `orgs.put_orgs_id` · `orgs_vas_revenue_boosts.get_orgs_vas_revenue_boosts` · `orgs_vas_revenue_boosts.get_orgs_vas_revenue_boosts_id` · `payment_update_groups.get_payment_update_groups` · `payment_update_groups.get_payment_update_groups_id` · `payment_updates.get_payment_updates` · `payment_updates.get_payment_updates_id` · `payouts.delete_payouts_id` · `payouts.get_payouts` · `payouts.get_payouts_id` · `payouts.post_payouts` · `payouts.put_payouts_id` · `pending_entry_api.get_pending_entries` · `pending_entry_api.get_pending_entries_id` · `pinless_debit_conversions.get_pinless_debit_conversions` · `pinless_debit_conversions.get_pinless_debit_conversions_id` · `plans.delete_plans_id` · `plans.get_plans` · `plans.get_plans_id` · `plans.post_plans` · `plans.put_plans_id` · `profit_share_results.get_profit_share_results` · `profit_share_results.get_profit_share_results_id` · `rev_share_schedules.delete_rev_share_schedules_id` · `rev_share_schedules.get_rev_share_schedules` · `rev_share_schedules.get_rev_share_schedules_id` · `rev_share_schedules.post_rev_share_schedules` · `rev_share_schedules.put_rev_share_schedules_id` · `rev_share_statements.get_rev_share_statements` · `rev_share_statements.get_rev_share_statements_id` · `revenue_boosts.get_revenue_boosts` · `revenue_boosts.get_revenue_boosts_id` · `safer_payments.get_safer_payments` · `safer_payments.get_safer_payments_id` · `secrets.delete_secrets_id` · `secrets.get_secrets` · `secrets.get_secrets_id` · `secrets.post_secrets` · `secrets.put_secrets_id` · `sessions.delete_sessions_id` · `sessions.get_sessions` · `sessions.get_sessions_id` · `sessions.post_sessions` · `statement_entry.get_statement_entries` · `statement_entry.get_statement_entries_id` · `statements.get_statements` · `statements.get_statements_id` · `subscription_tokens.delete_subscription_tokens_id` · `subscription_tokens.get_subscription_tokens` · `subscription_tokens.get_subscription_tokens_id` · `subscription_tokens.post_subscription_tokens` · `subscription_tokens.put_subscription_tokens_id` · `subscriptions.delete_subscriptions_id` · `subscriptions.get_subscriptions` · `subscriptions.get_subscriptions_id` · `subscriptions.post_subscriptions` · `subscriptions.put_subscriptions_id` · `tax_form_requests.get_tax_form_requests` · `tax_form_requests.get_tax_form_requests_id` · `tax_form_requests.post_tax_form_requests` · `team_logins.delete_team_logins_id` · `team_logins.get_team_logins` · `team_logins.get_team_logins_id` · `team_logins.post_team_logins` · `team_logins.put_team_logins_id` · `teams.delete_teams_id` · `teams.get_teams` · `teams.get_teams_id` · `teams.post_teams` · `teams.put_teams_id` · `terminal_transaction_reference.post_terminal_txn_refs` · `terminal_transaction_results.get_terminal_txn_results` · `terminal_transaction_results.get_terminal_txn_results_id` · `terminal_transactions.delete_terminal_txns_by_id` · `terminal_transactions.get_terminal_txns` · `terminal_transactions.get_terminal_txns_by_id` · `terminal_transactions.post_terminal_txns` · `terminal_transactions.update_terminal_txns_by_id` · `terminal_transactions_datas.get_terminal_txns_datas` · `terminal_transactions_datas.get_terminal_txns_datas_id` · `terminal_transactions_datas.post_terminal_txns_datas` · `terminal_transactions_metadatas.get_terminal_txns_metadatas` · `terminal_transactions_metadatas.get_terminal_txns_metadatas_id` · `terminal_transactions_metadatas.post_terminal_txns_metadatas` · `terminals.delete_terminals_id` · `terminals.get_terminals` · `terminals.get_terminals_id` · `terminals.post_terminals` · `terminals.put_terminals_id` · `token_results.get_token_results` · `token_results.get_token_results_id` · `tokens.delete_tokens_id` · `tokens.get_tokens` · `tokens.get_tokens_id` · `tokens.post_tokens` · `tokens.put_tokens_id` · `transaction_datas.get_txn_datas` · `transaction_datas.get_txn_datas_id` · `transaction_datas.post_txn_datas` · `transaction_holds.get_holds` · `transaction_holds.get_holds_id` · `transaction_holds.post_holds` · `transaction_holds.put_holds_id` · `transaction_items.delete_items_id` · `transaction_items.get_items` · `transaction_items.get_items_id` · `transaction_items.post_items` · `transaction_items.put_items_id` · `transaction_metadatas.get_txn_metadatas` · `transaction_metadatas.get_txn_metadatas_id` · `transactions_txns.get_txns` · `transactions_txns.get_txns_id` · `transactions_txns.post_txns` · `transactions_txns.put_txns_id` · `transactions_results.get_txn_results` · `transactions_results.get_txn_results_id` · `txn_sessions.delete_txn_sessions_id` · `txn_sessions.get_txn_sessions` · `txn_sessions.get_txn_sessions_id` · `txn_sessions.post_txn_sessions` · `vas_efe_offers.get_vas_efe_offers` · `vas_efe_offers.get_vas_efe_offers_id` · `value_added_services.get_value_added_services_enablement_status_target_id`); distinguishing members *none required*
   2. So `isinstance(e.error, ErrorFourHundred1)` matches only 361 of 361 operations.
6. **That a decode failure raises `ValidationError`/`ValueError`, not `ApiError`, in both response modes** — `core/raw_client.py` states this in `execute`'s own docstring. **And that the 2xx path declares no required member on any return type, so a truncated body decodes without complaint and the hole surfaces later.** Any sheet row for a call whose result is used must name the members the implementer has to assert on.
7. **That retrying is on by default** — `retry_options` on the client tunes it, `max_retries` set to `0` turns it off, and `request_options` overrides it per call — so the sheet says whether this integration keeps, tunes or disables it, and for which methods, since a write is repeated only when the policy names its method.
8. **Which environment the `environment` keyword selects**, because omitting it is silently `"production"` of the 3 declared.
9. A **REQUIRED READING** block naming the `python-*` companions that govern the steps, with `MUST load` pointers.

