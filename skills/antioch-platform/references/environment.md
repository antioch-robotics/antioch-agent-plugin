# Project environment

## Select and inspect

Read the owning `antioch.yaml`, Python dependencies/lock, Dockerfile, and
ignore rules. Use the project's virtual environment for the SDK and CLI, and
preserve deliberate dependency and engine pins.

`antioch version --json` reports component versions, including the
deployment and API host the CLI talks to. MCP executables resolve on the
agent's launch PATH: activating a shell environment later does not switch an
already-running adapter.

Private source-image credentials use `antioch auth registry login HOST`.

## Create a project

For authorized authoring in an empty directory, choose the selected SDK
release and package source. This public uv example uses Python 3.12:

```bash
uv init --bare --python ">=3.12,<3.13"
uv python pin 3.12
uv add --compile-bytecode "antioch-sim[isaac-sim]==<sdk-version>"
uv run antioch init --engine isaac-sim-6.0.1 --json
```

Use `antioch-sim[isaac-lab]` and engine `isaac-lab-3.0` for Lab; with one
engine extra, init can infer the engine. Use the existing package manager and
private wheel/source when supplied; do not substitute public latest.

`antioch init --help` lists supported engine identifiers. A different Isaac
version requires a compatible runtime image, not just a Python dependency
change; [research](../../antioch-research/SKILL.md) indexes versions, but
that is not an engine availability list.

Init writes the manifest, Dockerfile, ignore files, starter source, and
example suites. It does not install dependencies, register a remote project,
or start compute. Existing scaffold files are preserved; an owning manifest in
this directory or an ancestor causes refusal, and there is no force/repair
mode, so repair existing files directly. Remove unneeded generated examples
from the deliverable.

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

## Images and source

Init pins the installed SDK in `FROM antioch-engine/<engine>:<sdk-version>`.
Dockerfile engine references require a release tag; an untagged service
`image` uses the submitting SDK release. Scenario capability follows verified
image metadata, not the service name.

A generated Dockerfile sets `ANTIOCH_PROJECT_DIR` and uses
`/workspace/project`; project files reach that directory through the
manifest's `sync` mapping, not the image. Install custom dependencies in the
Dockerfile:

```dockerfile
COPY pyproject.toml uv.lock ./
RUN uv export --frozen --no-dev --no-emit-project --output-file /tmp/requirements.txt \
    && uv pip install --system --no-cache --requirements /tmp/requirements.txt
```

The generated `.dockerignore` ignores everything, so admit each copied file
with a `!pyproject.toml` or `!uv.lock` line. Keep credentials out of build
inputs. Registry images are mirrored to immutable digests; recorded reruns
use those digests, saved parameters and the saved bundle.

## Install and update

```bash
antioch setup --dry-run
```

Run the non-dry operation only when plugin installation or update is
requested. `antioch setup` configures the deployment's plugin for detected
Claude Code and Codex clients, then tests existing sign-in and Research
without logging in; read each component result even if the command fails.

Antioch never installs or upgrades the SDK. For a requested upgrade, rerun the
user's own package-manager install with the new exact version, preserving
their engine extra, pins, and lockfile. Change the Dockerfile's engine tag
only when asked. A live session keeps its image; a new session uses the
update.

## Builds and revisions

`antioch session new` builds changed images and freezes one immutable service
graph before it starts the session; an unchanged build is a cache hit. See
[sessions](sessions.md) for execution.

A session's containers keep their pinned images after an SDK update. Build inputs determine whether a later submission rebuilds them; [source mappings](manifest.md#source-sync) determine which files can change without rebuilding. An unchanged build key reuses existing images, so editing only mapped source does not require a new image.

Continue with [sessions](sessions.md) to run a script, [scenario design](../../scenario-design/SKILL.md) to author a recorded evaluation, or [the Python client](python-client.md) to automate the same workflow. [Authentication](auth.md) owns sign-in and private registry access; [Antioch platform](../SKILL.md#capability-guides) is the task index.

External Dockerfile `FROM` tags resolve to immutable digests before Antioch computes the build key; use `@sha256:` when you want to choose the digest yourself. A moved tag can affect a newly prepared revision, never an existing session or saved rerun. Antioch supplies the Dockerfile frontend, so a leading `# syntax=` directive is refused. Build context uses the platform exclusions and `.dockerignore`, not `.gitignore`; source sync uses its separate [mapping rules](manifest.md#source-sync).
