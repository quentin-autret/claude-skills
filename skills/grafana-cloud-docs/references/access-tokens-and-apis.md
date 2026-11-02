# Access tokens and APIs

Choosing between access policy tokens and service accounts, the Cloud API, the `/apis` instance API and stack secrets.

## Contents

- Which credential
- Access policies
- Token hygiene
- Label selectors and IP ranges
- Cloud API patterns
- Service accounts
- Instance `/apis` paths
- Secrets Management
- Old patterns

## Which credential

- Access policy tokens authorize telemetry (metrics, logs, traces, alerts) and the Cloud API at grafana.com. They do NOT authorize the instance HTTP API (dashboards, users, data sources, folders).
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/
- Instance HTTP API: service account tokens. Basic user:password is self-managed only.
  https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/authentication/
- Both send `Authorization: Bearer <TOKEN>`:
  ```http
  GET https://grafana.com/api/instances/<STACK_SLUG>/plugins HTTP/1.1
  Authorization: Bearer <CLOUD ACCESS POLICY TOKEN>
  ```
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/using-an-access-policy-token/
- Data sources in the UI: Basic auth, password = access policy token, user = the service's tenant ID (for Loki, the log tenant ID).
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/using-an-access-policy-token/
- Only the listed Cloud API endpoints are stable. Others can change without notice.
  https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/cloud-api/

## Access policies

- A policy = scopes (`service:action`, e.g. `metrics:read`, `logs:write`, `traces:write`, `stacks:read`, `stack-service-accounts:write`) + a realm + optional label selectors and IP ranges. It holds 1 or more tokens.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/
- Realm `org` can be applied to any stack of the organization. Realm `stack` limits the policy's tokens to that stack.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/create-access-policies/
- The in-stack page (Administration > Cloud access policies) shows only single-stack policies. Org-realm ones live in the Cloud Portal, where only the Admin role sees them.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/cloud-roles/
- Portal-created org policies land in a fixed US Central region. Other regions: Cloud API or Terraform.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/create-access-policies/

## Token hygiene

- One policy per task (read-metrics and write-metrics apart). Copy the token at creation: it is shown once.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/create-access-policies/
- A blank expiry (UI) or omitted `expiresAt` (API) never expires. Always set one. Org settings "Enforce token expiration" and "Max token expiration days" cap it.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/
- A created token cannot be modified in the UI. Deleting a token or a policy can take a few minutes to apply everywhere. Deleting a policy deletes its tokens.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/create-access-policies/

## Label selectors and IP ranges

- Selectors work only with read scopes for metrics and logs. Data matching ANY selector is returned (OR between, AND inside). With `{secret!="true", env="prod"}` and `{env="dev"}`, all of dev comes back, secrets included.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/label-access-policies/
- `conditions.allowedSubnets` (CIDR, IPv4 or IPv6) applies to every token of the policy. Connections from other addresses are denied.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/

## Cloud API patterns

```http
POST https://www.grafana.com/api/v1/accesspolicies?region=us
{ "name": "stack-readers", "displayName": "Stack Readers",
  "scopes": ["metrics:read","logs:read","traces:read","alerts:read"],
  "realms": [{"type":"stack","identifier":"123","labelPolicies":[{"selector":"{env != \"dev\"}"}]}],
  "conditions": {"allowedSubnets": ["192.168.0.0/24"]} }
```
Then `POST https://www.grafana.com/api/v1/tokens?region=...` with `accessPolicyId`, `name`, `expiresAt`.
https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/cloud-api/

- `region` is required (list at `https://grafana.com/api/stack-regions`). `name`: lowercase, digits, `-`, `_`, max 255. 600 requests per hour on policy endpoints. Page with `pageCursor` until `metadata.pagination.nextPage` is `null`.
  https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/cloud-api/
- Terraform: `grafana_cloud_access_policy` with a `realm` block, then `grafana_cloud_access_policy_token` with `access_policy_id`, `name` and `expires_at`.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/access-policies/using-an-access-policy-token/

## Service accounts

- Not tied to a user, so tokens survive the creator's deletion. Several tokens per account allow per-app rotation. One organization only, no instance-wide operations.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/
- Tokens have NO expiry by default. When unsure, expire within hours. Role `None` (10.2+) starts empty for RBAC grants.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/
- Team management permission does not add the account to the team. Check effective rights:
  ```sh
  curl -H "Authorization: Bearer glsa_..." '<grafana_url>/api/access-control/user/permissions' | jq
  ```
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/

## Instance `/apis` paths

`/apis/<GROUP>/<VERSION>/namespaces/<NAMESPACE>/<RESOURCE>[/<NAME>]`, Grafana 12+. Namespace `stacks-<STACK_ID>` on Cloud. Use `v1` (GA). `v1alpha1` and `v1beta1` are off by default. Paths use `metadata.name`, never `metadata.uid`. `resourceVersion` gives optimistic concurrency. Swagger at `/swagger`.
https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/apis/

## Secrets Management

- Stack secrets serve Grafana Cloud k6 and Synthetic Monitoring only. Values are never shown again, names never change. Deletion breaks every reference.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/manage-secrets/
- With no decrypter selected, no service can read the secret. Limits: name 253 chars, value 24 KiB, 10 labels. API `/apis/secret.grafana.app/v1beta1`.
  https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/manage-secrets/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| Instance API keys `/api/auth/keys` | Service accounts: `POST /api/serviceaccounts`, then `.../<id>/tokens` | https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/migrate-api-keys/ |
| Terraform `grafana_api_key` (removed in provider 3.0.0) | `grafana_service_account` + `grafana_service_account_token` | https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/migrate-api-keys/ |
| `grafana_cloud_stack_api_key` | `grafana_cloud_stack_service_account` + `_token` | https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/migrate-api-keys/ |
| Legacy `/api` routes (deprecated in Grafana 13, not yet disabled) | `/apis` | https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/apis/ |

Migrating ALL keys in the UI disables creating new API keys. `grafana_cloud_api_key` is not deprecated.
https://grafana.com/docs/grafana-cloud/platform/security-and-account-management/security-and-access/authentication-and-permissions/service-accounts/migrate-api-keys/
