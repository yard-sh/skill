# License keys and device activations

Keys minted at purchase that shipped software validates. Docs: https://docs.yard.sh/v1/platform/projects/license-keys.md (settings, when keys are generated, testing), https://docs.yard.sh/v1/api/licenses/validate-license.md (request, response, every `error_code`), https://docs.yard.sh/v1/guides/license-keys/license-key-validation.md (an activation screen end to end). Plan-gated: `.team_permissions.license_keys`; the server answers `upgrade_required`.

## Settings

- `license_key_enabled`, `activations_enabled`, `max_activations` (1-10000 per key): in `yard init --spec` or `yard projects edit --spec` (projects.md). Enabling keys on a project that already has purchases mints the missing keys and emails every holder; the command does not ask first.
- A purchase mints one key for `single`, `seat_count` keys for `fixed_pack` and one per seat for `per_seat`. Users see their keys in the Library.
- Each sandbox carries its own copy of these settings (copied at creation, then independent), set on its Sandbox Settings page, with the Update Sandbox endpoint or through MCP.

## Validating

`POST https://api.yard.sh/v1/licenses/validate` with an API key holding `licenses:validate` (plus `licenses:activate` when the body carries `device_id`); body `{ project_id, license_key, device_id?, device_name?, sandbox? }`. `project_id` is `yard projects show <slug> --json | jq -r .id`; it keeps a key bought for another project from validating (`license_not_for_project`). Letter case and surrounding spaces in the key are ignored.

- Only a `200` answers for the key: `valid: true`, or `valid: false` with an `error_code` to branch on (never match the message). `401`, `402` and `403` are about the team's API key or plan and `429` and `5xx` are transient, so shipped software keeps its last answer rather than locking the user out. `license_validation_disabled` means the team's plan no longer includes keys, not that the user's key is bad.
- Activations: only a validate that sends `device_id` counts against `max_activations`. A new device is refused once every slot is used, a known device keeps validating, and a validate without `device_id` checks the key alone, so send it on every check. `POST /v1/licenses/deactivate` frees a slot; users also manage devices from their Library.
- Sandbox keys are opt-in: validate and deactivate find a sandbox's keys only when the body names it (`"sandbox": "<name>"`). Without it, and for an unknown sandbox name, a sandbox key answers exactly like an unknown key (`license_not_found`), so a simulated purchase never unlocks a shipped build. A test build sends the sandbox's name.
- API keys are Pro-only (api.md). Installed software fetches updates with the license key itself, not an API key (updates.md).

## Testing end to end

```sh
yard sandbox create staging                       # inherits the project's license settings
yard keys create --spec - --json <<<'{"name":"local-validate","scopes":["licenses:validate","licenses:activate"]}'   # capture .key
PROJECT_ID=$(yard projects show <slug> --json | jq -r .id)
# Buy the project inside the sandbox from its checkout page (simulated, no card), then:
curl -X POST https://api.yard.sh/v1/licenses/validate \
  -H "Authorization: Bearer $YARD_API_KEY" -H 'Content-Type: application/json' \
  -d '{"project_id":"'"$PROJECT_ID"'","sandbox":"staging","license_key":"XXXX-XXXX-XXXX-XXXX","device_id":"laptop-42"}'
yard sandbox delete staging --yes                 # keys and activations go with it
```
