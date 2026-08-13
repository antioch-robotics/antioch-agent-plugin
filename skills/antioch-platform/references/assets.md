# Shared files: `antioch assets`

An asset is a named, versioned file available across the organization — USDs, datasets, checkpoints. Names are path-like (`robots/commissioning-arm`) to group related files. Published versions are **immutable**.

## CLI

- `antioch assets list --json` — the team's assets **alongside the shared Antioch shelf**, each row tagged with its source. JSON list output is `{ "items": [...], "next_cursor": "..." }`; `-q` searches a name substring, `--cursor` continues a previous page and pins its query, and `--limit` is 1–200 (default 50).
- `antioch assets show robots/commissioning-arm --json` — one asset with every revision it has published. Addressed by **name or exact asset id** — name wins on collision.
- `antioch assets push robot.usda --version v1 --description "Validated commissioning geometry"` — publishes one immutable revision (`-n/--name` defaults to the file's own name; `-v/--version` is **required**; `--content-type` overrides media-type inference; `--preview preview.png` stores a preview beside the revision; `--json`). Re-publishing the same name+version+content digest is an idempotent retry that repairs an interrupted upload; the same name+version with *different* content is refused with both digests. Text `.usda` and text-form `.usd` files with external `@asset@` or `@@@asset-with-@-characters@@@` references are refused because push sends one file; flatten the stage or publish a self-contained `.usdz`. Text inspection is bounded at 64 MiB. Binary `.usd`, `.usdc`, and `.usdz` files pass through the client without a dependency claim because it has no binary USD parser; `save_asset(source=...)` adds a pxr check inside a simulator when available.
- `antioch assets pull robots/commissioning-arm --version v1 --output robot.usda` — downloads (`-v` defaults to the latest version, `-o` names the destination, `--preview` fetches the preview instead, `--force` overwrites an existing file) via signed URL straight from object storage. Prints the destination path; `--json` emits one transfer manifest for scripts.
- `antioch assets verify robots/commissioning-arm --version v1` — downloads one version temporarily, verifies its SHA-256 digest, and scans text USD files for unpublished external dependencies. It exits nonzero when either check fails.
- `antioch assets repair robots/commissioning-arm --json` — removes published versions whose stored object is missing (a phantom left by an interrupted upload); `-v` limits it to one version, and the default sweeps every published version. The JSON result names the versions removed and kept. Reach for it when `assets pull` or `verify` reports a missing object.
- An organization asset **shadows** an Antioch-provided shared asset of the same
  name. The organization's version wins in listings and name resolution, while
  the shared asset stays addressable by exact ID. Names reserved on Antioch's
  shared shelf cannot be reused for an organization asset.

## From Python

```python
import antioch

prim = antioch.load_asset("robots/commissioning-arm", prim_path="/World/CommissioningArm", version="v1")
asset = antioch.save_asset("robots/commissioning-arm", version="v2", source="robot.usda", description="Validated geometry")
```

- `antioch.load_asset(name, *, prim_path, version=None) -> Usd.Prim` — downloads the revision into the verified local cache and references it as **native USD** into the active stage at `prim_path`. The path must be absolute and unused (`ValidationError` otherwise); with no live stage it raises `StateError`; auth, cache, or USD failures raise `AssetError`. Antioch checks access before using cached bytes.
- `antioch.fetch_asset(name, version=None) -> Path` — fetches a binary or text asset into the run-scoped verified cache for a framework loader; omitted versions resolve latest and log a pin hint; see the API docstring for the full contract.
- `antioch.save_asset(name, *, version, source=None, description=None, content_type=None, preview=None) -> AssetRecord` — with `source` it publishes that file (same immutability rules as `assets push`); with `source=None` it **flattens the active stage** and packages it as a `.usdz` — the way a conversion or import scenario publishes its result back to the shelf (the `isaac-sim-6` skill's assets references cover those workflows).
- `antioch.lib.asset` is gone — all three functions live at the package root (`antioch.fetch_asset`, `antioch.load_asset`, and `antioch.save_asset`, plus `antioch.AssetError`).
