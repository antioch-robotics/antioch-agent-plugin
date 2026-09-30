# Identity and access

`antioch auth whoami --json` reports user, organization, deployment, API, and
credential source. Login is shared across virtual environments for the
selected deployment.

`ANTIOCH_TOKEN` overrides a saved browser login; login, logout, and
organization switching refuse while it is active. Do not print or replace it,
switch deployment, or change accounts to make a failed lookup pass.

Remote access and local authoring are separate: complete authorized project
setup and offline validation while access is blocked. A Research failure alone
does not establish that simulation access fails.

## User-requested changes

- `antioch auth login` prints a device code and activation URL for the human
  to approve. Repeating it replaces the saved deployment login.
- `antioch auth switch --org ORG` selects the organization that owns
  subsequent work.
- `antioch auth logout --json` removes the saved local login.

An error recommending login is a diagnostic, not permission to change
identity. Account settings are in the console.

## Private image registries

Registry credentials belong to the project, separately from the person's deployment login:

```bash
antioch auth registry list --json
antioch auth registry login HOST
antioch auth registry logout HOST
```

Login prompts for a username and hidden password or token. Automation can pass `--username` and read the secret with `--password-stdin`; never put it in a command argument, project file, or response. Listing returns host and username without secrets. These credentials let a project pull private source images during [image preparation](environment.md#images-and-source). Only store or remove them when that change is authorized.

Use [environment setup](environment.md) for SDK and plugin installation, [manifest identity](manifest.md#services-and-images) for an organization-bound checkout, and [troubleshooting](troubleshooting.md) for a rejected request. Return to [Antioch platform](../SKILL.md#capability-guides) for the full task index.
