# Reusable assets

The organization catalog holds versioned robots, props, environments,
datasets, and checkpoints. Search before recreating named content:

```bash
antioch asset list -q "mobile robot" --json
antioch asset show robots/example --json
antioch asset pull robots/example --version v1 --output ./assets/example
```

Every query word must occur in the name or description; this is text search,
not a directory filter. Read scope, description, and versions before choosing.
Wire records use `scope` (`tenant`, `shared`). CLI scope is `organization`
or `shared`; `organization` selects the organization's assets. Show includes content type,
size, digest, and preview.

## Load and measure

After simulator startup:

```python
import antioch

prim = antioch.load_asset("robots/example", prim_path="/World/Robot", version="v1")
path = antioch.fetch_asset("datasets/example", version="v2")
```

Load requires an active stage, an unused absolute prim path, and USD with a
valid default prim. The reference includes that prim's subtree, not sibling
materials; keep required materials inside it or reference them explicitly.
Fetch returns verified bytes as a local file path for
another loader; it does not require Kit or validate USD dependencies.
Pin versions for repeatable work.
The catalog has no dimensions: read loaded bounds and stage `metersPerUnit`
before converting sizes to meters.

## Isaac's native asset library

Isaac also supplies robots, props, and environments outside the Antioch catalog.
Use [research](../../antioch-research/SKILL.md) to find the asset's exact path
and prerequisites for the selected Isaac version. After
[simulator startup](../../isaac-sim-6/SKILL.md), use
`isaacsim.storage.native.get_assets_root_path()` to resolve the configured
library root and `path_join` to append the verified library-relative path.
The root may be a remote URL, not a directory on the authoring computer.

Example paths relative to that root in Isaac Sim 6.0.1:

- Warehouse: `Isaac/Environments/Simple_Warehouse/full_warehouse.usd`
- Robot: `Isaac/Robots/FrankaRobotics/FrankaPanda/franka.usd`

Pass these without a leading slash: `path_join(root, "Isaac/Robots/FrankaRobotics/FrankaPanda/franka.usd")`.
For other content, search the indexed [Isaac asset catalog](https://docs.isaacsim.omniverse.nvidia.com/6.0.1/assets/usd_assets_overview.html) and examples first.
To browse a listable runtime directory, use
`isaacsim.storage.native.find_filtered_files([directory], max_depth=2)`;
choose a narrow directory, such as the resolved `Isaac/Robots/FrankaRobotics`,
rather than recursively scanning the whole library. A URL can serve files
without supporting directory listing; an empty listing does not prove absence.

Reference the resolved USD path with native USD/Isaac APIs; an absolute path
must be reachable from the remote service, not merely on the client. `fetch_asset` and
`load_asset` accept Antioch catalog names or IDs, not arbitrary paths or URLs.
Check availability and dependent textures/layers before judging a loaded scene.
See [USD pipeline](../../isaac-sim-6/references/usd-pipeline.md) for packaging
and [spatial reasoning](../../isaac-sim-6/references/spatial-reasoning.md) for
bounds, units, and placement.

## Publish

Before publishing USD or calling it ready for reuse, reference it into a clean
scene on the target runtime and inspect geometry, materials, and dependencies.

```bash
antioch asset push ./robot.usdz --name robots/example --version v2
antioch asset verify robots/example --version v2
```

Versions are immutable. Repeating the same digest is idempotent; different
bytes under the same version are refused. Publish self-contained content:
flattening USD composition does not package textures. Use the Isaac
[USD pipeline](../../isaac-sim-6/references/usd-pipeline.md) for dependency packaging.

Verify checks the stored object digest and supported text-USD references,
not a local file or whether USD parses and renders. For an incomplete upload, inspect `antioch asset repair --help`
and target only that version. The SDK's `save_asset` publishes files or
the flattened stage; inspect its installed signature for version/preview inputs.
