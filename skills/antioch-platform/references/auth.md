# Identity: `antioch auth`

## `antioch auth login` — device flow

```
antioch auth login
```

- Prints a sign-in code and activation URL, then polls for approval. **In an agent context, surface the code and URL to the user and wait** — only the human can complete the browser step. `--json` emits the authenticated identity once it lands.
- The CLI stores the login securely under `~/.config/antioch` and refreshes it when needed. The login survives virtual-environment changes and new projects; running `auth login` again replaces it.

## Identity and organizations

- `antioch auth whoami --json` names the user and active organization. Run this
  first when identity or access is unclear.
- `antioch auth switch --org ORG` selects an organization without a prompt;
  add `--json` to emit the resulting identity. Without `--org`, the command
  remains an interactive selector. The selected organization owns every run
  and asset created afterwards.
- `antioch auth logout --json` removes the local login and machine access from this computer.

## Account settings

Account settings live in the webapp, not the CLI.
