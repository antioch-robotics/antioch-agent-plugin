# Reusable assets

The organization catalog holds versioned robots, props, environments,
datasets, and checkpoints. Search before recreating named content:

```bash
antioch asset list -q "mobile robot" --json
antioch asset show robots/example --json
antioch asset pull robots/example --version v1 --output ./assets/example
```

Every query word must occur in the name or description; this is text search,
not a directory filter. Wire records use `scope` (`tenant`, `shared`), which
the CLI's SCOPE column shows as Organization and Shared. Show includes
content type, size, digest, and preview.

## Fetch or load

`fetch_asset` returns verified bytes as a local `Path`, without a simulator:

```python
import antioch

path = antioch.fetch_asset("datasets/example", version="v2")
```

For USD, start the simulator and reference it into the scene:

```python
import antioch

prim = antioch.load_asset("robots/example", prim_path="/World/Robot", version="v1")
```

Load requires an active stage, an unused absolute prim path, and USD with a
valid default prim; the reference includes that prim's subtree, not sibling
materials. Fetch does not validate USD dependencies. Pin versions
for repeatable work. The catalog has no dimensions: read loaded bounds and
stage `metersPerUnit` before converting sizes to meters.

## Isaac's native asset library

Isaac also supplies robots, props, and environments outside the Antioch
catalog. Use [research](../../antioch-research/SKILL.md) to find the asset's
exact path for the selected Isaac version. After
[simulator startup](../../isaac-sim-6/SKILL.md),
`isaacsim.storage.native.get_assets_root_path()` resolves the library root,
which may be a remote URL, and `path_join` appends the library-relative path
without a leading slash:

- Warehouse: `Isaac/Environments/Simple_Warehouse/full_warehouse.usd`
- Robot: `Isaac/Robots/FrankaRobotics/FrankaPanda/franka.usd`

For other content, search the indexed
[Isaac asset catalog](https://docs.isaacsim.omniverse.nvidia.com/6.0.1/assets/usd_assets_overview.html)
and examples first. To browse a listable runtime directory, use
`isaacsim.storage.native.find_filtered_files([directory], max_depth=2)` on a
narrow directory such as the resolved `Isaac/Robots/FrankaRobotics`; a URL
can serve files without supporting listing, so an empty listing does not
prove absence.

Reference the resolved USD path with native USD/Isaac APIs; an absolute path
must be reachable from the remote service, not merely on the client.
`fetch_asset` and `load_asset` accept Antioch catalog names or IDs, not
arbitrary paths or URLs. See [USD pipeline](../../isaac-sim-6/references/usd-pipeline.md)
for packaging and [spatial reasoning](../../isaac-sim-6/references/spatial-reasoning.md)
for bounds, units, and placement.

## Publish

Before publishing USD, reference it into a clean scene on the target runtime
and inspect geometry, materials, and dependencies.

```bash
antioch asset push ./robot.usdz --name robots/example --version v2
antioch asset verify robots/example --version v2
```

Versions are immutable: repeating the same digest is idempotent, and
different bytes under the same version are refused. Publish self-contained
content; flattening USD composition does not package textures (see the
[USD pipeline](../../isaac-sim-6/references/usd-pipeline.md)).

Verify checks the stored object digest and supported text-USD references,
not a local file or whether USD parses and renders. For an incomplete upload,
inspect `antioch asset repair --help` and target that version. In Python,
`save_asset(name, version=..., source=...)` publishes a file without Kit;
omitting `source` packages the active USD stage. Use the catalog for reusable
inputs, `run.add_artifact` for one run's evidence. Publishing shared content
must be within the user's request.

`asset push --preview` attaches an image to the published version; `asset pull --preview` downloads that image instead of the content. A name with slashes groups related assets in the console; version labels remain immutable regardless of the displayed folder. `asset repair` removes only versions whose stored object is missing; it does not repair malformed USD or replace different bytes under an existing version. Inspect the exact version before requesting that mutation.

For automation, use [the Python client](python-client.md#inspect-history-and-reusable-assets). For validation before publication, use [Isaac validation](../../isaac-sim-6/references/isaac-sim-validator.md). For per-run outputs, use [scenario artifacts](../../scenario-design/SKILL.md#measured-verdicts). Return to [Antioch platform](../SKILL.md#capability-guides) for project and session operations.
