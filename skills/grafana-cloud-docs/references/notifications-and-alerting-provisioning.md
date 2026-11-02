# Notifications and alerting provisioning

Contact points, notification policies, grouping, mute timings, silences, notification templates plus provisioning by Terraform or HTTP API.

## Contents

- Contact points and webhooks
- Notification policies
- Grouping and timing
- Mute timings and silences
- Notification templates
- Provisioning
- Terraform
- Old patterns

## Contact points and webhooks

- A contact point with no integrations sends no notifications.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/notifications/contact-points/
- Email in Cloud needs no SMTP. "Single email" sends one mail to all addresses.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/manage-contact-points/integrations/configure-email/
- Never put secrets in a webhook URL: it is stored in plain text and shown on read. Use Basic auth or the Authorization header (default scheme `Bearer`). OAuth2 overrides both.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/manage-contact-points/integrations/webhook-notifier/
- HMAC-SHA256 signing goes in `X-Grafana-Alerting-Signature` (hex). With a timestamp header it signs `timestamp + ":" + body`: verify freshness, hash the raw body.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/manage-contact-points/integrations/webhook-notifier/
- Max Alerts drops the extras (`0` = no limit), counted in `truncatedAlerts` (`TruncatedAlerts` in templates). Unresolved `endsAt` is `0001-01-01T00:00:00Z`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/manage-contact-points/integrations/webhook-notifier/

## Notification policies

- A tree. The default policy matches everything and cannot be deleted.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/notifications/notification-policies/
- Routing goes top-down. The deepest matching child wins. Siblings stop at the first match unless "Continue matching siblings" is on, so sibling order matters. Matchers are ANDed. Exclude a label with `team !~ .+`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/notifications/notification-policies/
- Children inherit contact point, grouping and timings unless overridden. The policy search box has no regex.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/create-notification-policy/
- Extra policy trees exist: a rule picks one with `notification_settings.policy`, else it uses the default tree.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/terraform-provisioning/

## Grouping and timing

- Default group by `alertname` + `grafana_folder`. Group by `...` disables grouping.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/notifications/group-alert-notifications/
- Defaults: group wait 30 s, group interval 5 min, repeat 4 h. An alert resolving within group wait sends nothing. Repeat rounds up to a multiple of group interval (9 min becomes 10), max 5 days.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/fundamentals/notifications/group-alert-notifications/

## Mute timings and silences

- Mute timings and active time intervals only mute notifications: rules still evaluate. Mute timings take precedence over active time intervals when they overlap. They attach to child policies only and are not inherited.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/mute-timings/
- Ranges: start inclusive, end exclusive, UTC by default, `monday:thursday`, `-1` = last day of month. First Monday of March, June, September and December: weekday `monday`, months `3, 6, 9, 12`, days `1:7`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/mute-timings/
- Silences cannot be permanently deleted manually: "Unsilence" ends them, expired ones vanish after 5 days. A changed label value makes the instance a new one that might no longer match. One rule: `__alert_rule_uid__=<uid>`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/create-silence/
- Prefilled link: `https://<host>/alerting/silence/new?matcher=severity%3Dcritical&comment=...`. In templates use `.Alert.SilenceURL`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/create-silence/

## Notification templates

- Dot is the notification data: `.Alerts.Firing`, `.CommonLabels`, `.GroupLabels`, `.Status`. Rule templates use `$labels` instead.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/template-notifications/reference/
- Duplicate names, defaults like `default.message` included, raise no error and either one may win. Use unique names.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/template-notifications/language/
- Dot changes inside `range` and `with`: pass it, `{{ template "print_alerts" .Alerts.Firing }}`. Trim with `{{-` `-}}`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/template-notifications/language/

## Provisioning

- Cloud has no file provisioning: use Terraform or the HTTP API. Provisioned resources cannot be edited in the UI by default.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/
- The HTTP API covers Grafana-managed alerts only. Data source-managed alerts, recording rules included, use the Mimir or Cortex tool.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/
- The policy tree is one resource: provisioning it overwrites all policies.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/export-alerting-resources/
- Export with `GET /api/v1/provisioning/alert-rules/export?format=yaml|json|hcl`. Export JSON cannot update via the API. Template groups have no export.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/export-alerting-resources/
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/http-api-provisioning/
- Recording rule by API: a `record` block instead of `condition`, no `noDataState` or `execErrState`, `for: 0s`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/http-api-provisioning/
- `X-Disable-Provenance: true` keeps API resources UI-editable. `DELETE /api/v1/provisioning/policies` resets and unlocks the tree. Contact point endpoints see v1 integrations only.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/http-api-provisioning/

## Terraform

- Service account token with the role or permission for the Alerting provisioning API. Resources: `grafana_rule_group`, `grafana_contact_point`, `grafana_message_template`, `grafana_notification_policy`, `grafana_mute_timing`, `grafana_apps_notifications_routingtree_v1beta1`.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/terraform-provisioning/
- Resources link through `uid` references (`folder_uid = grafana_folder.<name>.uid`). Expressions use `datasource_uid = "__expr__"`. An existing rule group can be exported in Terraform format and edited.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/terraform-provisioning/
- Template from a contact point: `message = "{{ template \"custom_email.message\" .}}"`. `disable_provenance = true` allows UI edits.
  https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/terraform-provisioning/

## Old patterns

| Old | Current | Source |
| --- | --- | --- |
| `/api/v1/provisioning/contact-points`, `policies`, `templates`, `mute-timings` | `/apis/notifications.alerting.grafana.app/v1beta1/namespaces/{namespace}/receivers`, `timeintervals`, `templategroups`, `routingtrees` | https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/http-api-provisioning/ |
| `/api/v1/provisioning/alert-rules` | `/apis/rules.alerting.grafana.app/v1beta1/namespaces/{namespace}/alertrules`, `recordingrules` | https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/api-legacy/alerting_provisioning/ |
| `/api` routes (Grafana 13) | `/apis` | https://grafana.com/docs/grafana-cloud/learn-and-build/developer-resources/api-reference/http-api/api-legacy/alerting_provisioning/ |
| Pre-provisioned data source-managed alerting | Grafana-managed (import tool) | https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/set-up/provision-alerting-resources/http-api-provisioning/ |
| Standalone Alertmanager UI | Grafana Alerting UI | https://grafana.com/docs/grafana-cloud/observe-and-act/alert-and-measure-reliability/alerting/configure-notifications/create-silence/ |
