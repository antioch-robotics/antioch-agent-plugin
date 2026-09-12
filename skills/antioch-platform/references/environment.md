# Select the SDK and runtime images

The local SDK supplies the `antioch` Python package, CLI, and editor stubs. It
does not install Isaac. Remote service images supply the simulator and other
runtime dependencies.

## Create a project

Choose one engine extra for the local project:

```bash
uv add --compile-bytecode "antioch-sim[isaac-sim]"
antioch init
```

Use `antioch-sim[isaac-lab]` for Isaac Lab. Isaac Lab also installs Isaac Sim
editor stubs because Lab builds on Sim.

`antioch init` chooses the engine from the installed extra and creates
`antioch.yaml`, `Dockerfile`, `.dockerignore`, example simulation code, and
example suites. It does not start a session or upload project content.

## Select the simulation image

The generated Dockerfile opens with `FROM antioch-engine/<engine>` — read the
generated file to see the exact tag.
`antioch init` writes the installed SDK release into this `FROM` line, so the
local SDK and the remote engine image start on the same release.
Do not write that line by hand and do not leave it
untagged: an engine reference with no tag has no default and the build refuses
it. A package-manager upgrade does not rewrite an existing Dockerfile. Explicit
`antioch project update` can update literal engine tags with the selected SDK;
ordinary build and init behavior is unchanged.

The simulator service is the one that uses an Antioch engine image or a
Dockerfile that starts from one; the role follows the image, not the service
name. Supporting services can use ordinary registry images.

## Publish custom dependencies

Append these instructions to the generated Dockerfile, below the `FROM` line
`antioch init` wrote:

```dockerfile
WORKDIR /workspace/project
COPY pyproject.toml uv.lock ./
RUN uv export --frozen --no-dev --no-emit-project --output-file /tmp/requirements.txt \
    && uv pip install --system --no-cache --requirements /tmp/requirements.txt

COPY . .
```

Extend the generated Dockerfile for project dependencies. Antioch builds it
from the declared context. Do not put credentials in `antioch.yaml` or the
Dockerfile.

## Use registry images

```yaml
services:
  autonomy:
    image: registry.example.com/robot/autonomy:release
```

Antioch resolves and mirrors the image to an Antioch-owned digest. A later
change to the source tag cannot change a submitted run.

## Understand repeatable identity

Each submitted scenario or suite saves the resolved digest for every service
along with the selected inputs. Together, they identify the exact software and
settings used for that run.

Project source lives at `/workspace/project` for `image:` and `build:`
services. Watch actions transfer edits into a live interactive session. A
background submission builds the current YAML independently. Dockerfile `COPY`
must place source at that same path; there is no run source bundle. A rerun
uses saved image digests and parameters under a new run ID. It does not preserve
unbuilt interactive edits. An unsupported old runtime is refused before
allocation, while its saved history remains readable.
It does not promise the same outcome or timing when scheduling, capacity,
simulator timing, or external assets differ.

## Update the project SDK

```bash
antioch project update --dry-run
antioch project update
antioch --version
```

Update the Dockerfile's engine tag deliberately. Run the narrow scenario or
suite that proves compatibility after the update.
