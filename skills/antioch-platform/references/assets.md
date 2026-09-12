# Work with assets

Assets are versioned robots, props, environments, datasets, checkpoints, and
other reusable content. Organization members share the catalog. Search it before you
create equivalent content.

## Find an asset

Every `-q` word must appear, in any order and any case, in an asset's name or
description. Names are folder paths such as `robots/example`, so folder names
are searchable too. This is a text search, not a directory filter:

```bash
antioch asset list -q "mobile robot" --json
antioch asset list -q robots --json
antioch asset show robots/example --json
```

Asset records use `scope` (`tenant` or `shared`). The CLI renders `tenant` as
`organization` in both human and JSON output. Each listed entry carries `name`,
`description`, `scope`, `latest_version`, and `version_count`. `show` adds
`versions`:
each version's `content` carries `content_type`, `size_bytes`, and `sha256`;
its `preview` is present when published. When several assets could fit,
prefer the one whose description says what you need, and pin its version
when you load it. Use `antioch asset list --help` for paging.

The shelf stores no dimensions. To measure an asset, load it and read its
bounds inside the scenario body. The result is in stage units, not necessarily
meters; check the stage's `metersPerUnit` before converting it:

```python
from pxr import Usd, UsdGeom

prim = antioch.load_asset("robots/example", prim_path="/World/Robot", version="v1")
bounds = UsdGeom.BBoxCache(Usd.TimeCode.Default(), [UsdGeom.Tokens.default_]).ComputeWorldBound(prim)
size_stage_units = bounds.ComputeAlignedRange().GetSize()
```

## Pull and verify

```bash
antioch asset pull robots/example --version v1 --output ./assets/example
antioch asset verify robots/example --version v1
```

Transfers use signed links directly to object storage. Verification checks the
stored published object's SHA-256 digest and rejects unresolved external
references in supported text USD files. It does not read a local file.

## Publish a version

```bash
antioch asset push ./robot.usdz --name robots/example --version v2
```

Asset versions are immutable. Repeating a publish with the same digest is an
idempotent retry. The same name and version with different content is refused.
Publish self-contained USDZ content. Flattening USD composition does not
package textures or other external files. Load the Isaac Sim skill's
`references/usd.md` for dependency packaging and validation before publishing.
Use `antioch asset push --help` for version, description, media type, and
preview input.

If an interrupted publish left an asset version without content, inspect
`antioch asset repair --help` and repair only the named asset.

## Use assets in Python

```python
import antioch

prim = antioch.load_asset("robots/example", prim_path="/World/Robot", version="v1")
path = antioch.fetch_asset("datasets/example", version="v2")
```

`antioch.load_asset` verifies and references native USD content into an active
stage. `antioch.fetch_asset` returns a verified path for another framework
loader. Pin a version in repeatable work.

Publish a file or the flattened active stage with `antioch.save_asset`. Read
the installed API docstring before use because the required version and
preview behavior are part of the Python contract.
