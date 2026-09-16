# Project environment

## Select and inspect

Read the owning `antioch.yaml`, Python dependencies/lock, Dockerfile, and
ignore rules. Check the project's interpreter and SDK, not only the global
CLI. Preserve deliberate pins and the selected deployment.

`antioch version --json` reports component versions. Production is the
default; `ANTIOCH_ENV=staging` selects staging. MCP executables resolve on
the agent's launch PATH: activating a shell environment later does not switch
an already-running adapter.

Use `antioch --help` for command groups and leaf help for flags.
`antioch project list --json` finds registered organization projects;
`antioch project show --json` combines local configuration with registry
information. These need authentication. Private source-image credentials use
`antioch project registry`; inspect its help before a requested login change.

## Create a project

For authorized authoring in an empty directory, choose the selected SDK
release and package source. This public uv example uses Python 3.12:

```bash
uv init --bare --python ">=3.12,<3.13"
uv python pin 3.12
uv add --compile-bytecode "antioch-sim[isaac-sim]==<sdk-version>"
uv run antioch init --engine isaac-sim-6.0.1 --json
```

Use `antioch-sim[isaac-lab]` and engine `isaac-lab-3.0` for Lab.
With one engine extra, init can infer the engine. Use the existing package
manager and private wheel/source when supplied; do not substitute public latest.

`antioch init --help` lists supported engine identifiers. Choose the image and
matching SDK types together; a different Isaac version requires a compatible
runtime image, not just a Python dependency change. [Research](../../antioch-research/SKILL.md)
can compare version-specific APIs, but its indexed versions are not an engine
availability list. Confirm a required version is supported before promising it.

Init writes the manifest, Dockerfile, ignore files, starter source, and example
suites. It does not install dependencies, register a remote project, or start
compute. Existing scaffold files are preserved. An owning manifest in this
directory or an ancestor causes refusal; there is no force/repair mode.
Repair existing files directly without deleting identity to rerun init.

Keep one intended project and remove unneeded generated examples from your
deliverable. Check that preserved Dockerfiles and ignore rules include the
requested source.

## Validate locally

These SDK calls need no authentication or simulator:

```python
from antioch.core.project import load_manifest
from common.models import ProjectManifest

manifest = load_manifest()
schema = ProjectManifest.model_json_schema()
```

Validate the manifest and import source with the project interpreter.
Inspect schema fields and installed API docstrings rather than guessing keys.
`antioch project show --json` also resolves registry/service information
and requires authentication; it is not an offline validator.

## Images and source

Init pins the installed SDK in `FROM antioch-engine/<engine>:<sdk-version>`.
Untagged engine references are refused. The simulator role follows this image,
not the service name. With multiple engine services, mark the runner using
`x-antioch: {runner: true}`.

A generated Dockerfile sets `ANTIOCH_PROJECT_DIR`, uses
`/workspace/project`, and ends with `COPY . .`. For custom dependencies,
insert installation before that final copy, for example:

```dockerfile
COPY pyproject.toml uv.lock ./
RUN uv export --frozen --no-dev --no-emit-project --output-file /tmp/requirements.txt \
    && uv pip install --system --no-cache --requirements /tmp/requirements.txt
```

Keep credentials out of build inputs. An interactive session receives initial
source and later sync edits. A detached run receives image contents: all
required source must be baked in, not merely present in a live notebook.
Registry images are mirrored to immutable digests. Recorded reruns use those
digests and saved parameters; they do not promise identical physics or timing.

## Install and update

```bash
antioch setup --dry-run
antioch project update --dry-run
```

Run the non-dry operation only when installation/update is requested:

- `antioch setup` installs the selected deployment's verified SDK/plugin
  pair, globally with uv or into an existing `--python PATH` environment.
  It configures detected Claude Code and Codex clients, not projects or shell
  profiles. Its post-install checks test existing sign-in and Research without
  logging in; read each component result even if the command fails.
- `antioch project update` updates project dependency and literal engine
  pins, locks/syncs with uv, and builds changed images. `--no-build` excludes
  builds. It preserves engine families and extras and does not replace a live
  session. Use a new session to run the new image.

Staging private artifacts need existing provider access. Do not replace a
missing release pair or credentials with a guessed alternative.

## Build and revision history

```bash
antioch project build
antioch project revision list --json
antioch project revision show REVISION
antioch project revision tag candidate REVISION
```

Build finalizes one immutable service graph. Revision tags are movable names
for existing revisions; tagging does not rebuild. Build only within the
authorized task, then use [sessions](sessions.md) for execution.
