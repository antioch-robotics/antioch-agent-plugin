# Identity and access

`antioch auth whoami --json` reports user, organization, deployment, API,
and credential source. Check this when access is unclear. Login is shared
across virtual environments for the selected deployment.

`ANTIOCH_TOKEN` overrides a saved browser login. Login, logout, and
organization switching refuse while it is active. Do not print or replace it,
switch deployment, or change accounts to make a failed lookup pass.

Remote access and local authoring are separate. Complete authorized project
setup and offline validation while access is blocked. A Research failure alone
does not establish that simulation access fails.

## User-requested changes

- `antioch auth login` prints a device code and activation URL for the
  human to approve. Repeating it replaces the saved deployment login.
- `antioch auth switch --org ORG` selects the organization that owns
  subsequent work.
- `antioch auth logout --json` removes the saved local login.

An error recommending login is a diagnostic, not permission to change identity.
Account settings are in the console.
