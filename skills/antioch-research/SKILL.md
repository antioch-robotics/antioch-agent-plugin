---
name: antioch-research
description: The retrieval hub for the entire simulation substrate. Fires for ANY question, implementation, review, or debugging that touches Isaac Sim, Isaac Lab, Omniverse/Kit, OpenUSD, PhysX, Newton, Warp, cuRobo, skrl, rsl-rl, Cosmos, NuRec, Isaac ROS, or Rerun — the research_* MCP tools are the canonical lookup verb for those surfaces and beat memory, upstream websites, and every other skill at API detail. One research_search spans all 21 indexed corpora, including pinned source for Isaac Sim, Isaac Lab, rsl-rl, Warp, and cuRobo. Load this before writing, porting, reviewing, or debugging any code on those stacks; whenever an API signature, parameter semantics, default, error string, or source behavior is in question; and when grounding or verifying any claim about them. Teaches the search, expand, and open loop, detailed-query style, kind='source' code localization, and grep for exact symbols. Not for Antioch platform or CLI questions (antioch-platform) or vendors outside the index (web research).
---

# Antioch Research

Antioch Research is the agent's one verb for "go find out" on the vendor
simulation stack. Six MCP tools query Antioch's hosted retrieval index — a
hybrid (dense + sparse + reranked) index over versioned documentation and
pinned source where available, with every hit expandable to its neighbors or
its whole file/page.

## The hard rule

**Before writing, reviewing, or debugging anything that touches an indexed
substrate, call a retrieval tool.** Which tool depends on one question: do
you KNOW the symbol, or are you GUESSING it?

* **Know it** — you read it in a hit, a traceback, an error string, or the
  user's code. Use `research_grep`, then chain `research_open` on the
  matching artifact. Grep confirms exact strings well.
* **Guessing it** — you are inferring a plausible name. Use
  `research_search`. Grep is AND-of-tokens, so a wrong guess returns almost
  nothing, and thin grep output is indistinguishable from "this API does not
  exist". Grepping the invented `RtxRadar` returns noise; the real symbols
  are `Radar`, `RadarSensor`, and `OmniRadar`, and only semantic search
  finds them.

Use `research_search` first for conceptual or natural-language questions.
The index is ground truth at the pinned versions and a search costs
seconds; guessing an API and burning a turn discovering the guess was wrong
costs far more. Do not skip the lookup
because the API feels familiar — adapted NVIDIA content and model memory both
carry stale local-install-era surfaces. The skills in this plugin orient
(concepts, architecture, recipes, invariants); research grounds (signatures,
parameters, defaults, source behavior, error strings).

Use the tools liberally: several research calls per task is the norm, not
the exception. Search when starting a task, search again whenever a detail
is uncertain, expand or open when a hit is close, grep when a symbol or
error string is exact. A call costs seconds; an unresearched guess costs a
turn.

## One search, all corpora

**The default `research_search` spans ALL 21 indexed corpora simultaneously**,
and its ranking diversifies across corpora so the right vendor surfaces on
its own. Corpus filters (`corpus=`) are optional drill-ins for a second,
narrower pass — they are never required, and filtering the first pass is an
anti-pattern. `kind='source'` / `kind='docs'` are the only first-pass
filters worth setting, and only when the intent is clearly one or the other.

## Corpora indexed (21)

| Corpus | Version | Covers |
|---|---|---|
| `isaac-sim-docs` | 6.0.1 | Isaac Sim documentation |
| `isaac-sim-source` | v6.0.1 | Isaac Sim extensions, python packages, standalone examples |
| `isaac-lab-docs` | 3.0.0-beta2 | Isaac Lab documentation |
| `isaac-lab-source` | v3.0.0-beta2 | Isaac Lab source and scripts |
| `omniverse-docs` | dated crawl | Omniverse ecosystem: Omni Physics, Extensions, Materials + Rendering, USD, CAE |
| `kit-docs` | dated crawl | Kit Manual |
| `openusd-docs` | dated crawl | OpenUSD user guides + Doxygen API |
| `physx-docs` | 5.6.1 | PhysX solver reference: iterations, contacts, soft bodies, attachments |
| `newton-docs` | dated crawl | Newton differentiable physics backend |
| `warp-docs` | 1.16.0 | NVIDIA Warp docs — upstream `stable` tree; NEWER than the Isaac Lab 3.0 pin |
| `warp-source` | v1.13.0 | NVIDIA Warp source at Isaac Lab 3.0.0-beta2's exact pin (`warp-lang==1.13.0`) |
| `cosmos-docs` | dated crawl | Cosmos World Foundation Model + cookbook |
| `nurec-docs` | dated crawl | NuRec neural scene reconstruction |
| `isaac-ros-docs` | dated crawl | Isaac ROS: cuVSLAM, Apriltag, nav2, image pipeline |
| `rerun-docs` | dated crawl | Rerun telemetry SDK + Python API reference |
| `skrl-docs` | dated crawl | skrl RL library (Isaac Lab-supported) |
| `curobo-docs` | dated crawl | NVIDIA cuRobo motion planning |
| `curobo-source` | v0.8.0 | NVIDIA cuRobo Python source |
| `rsl-rl-source` | v5.0.1 | rsl-rl-lib source at Isaac Lab 3.0.0-beta2's exact pin — its default RL training library |
| `gymnasium-source` | v1.2.1 | Gymnasium source at Isaac Lab's exact pin — env API, vector envs, autoreset semantics (CHANGED in 1.0) |
| `rl-games-source` | python3.11 | rl_games at the isaac-sim fork Isaac Lab pins — the library behind its 37 rl_games tasks |

`deprecated/`, `legacy/`, and archived-version doc trees are excluded at
ingest, so a superseded page cannot outrank its current one. Two things
that filter cannot decide for you still need your eyes:

* **`experimental` is not a synonym for stale.** In Isaac Sim 6,
  `isaacsim.sensors.experimental.rtx` is the CURRENT RTX sensor API while
  `isaacsim.sensors.rtx` is the deprecated one — one path segment apart,
  different behavior. Read the path on every hit and prefer the one the
  pinned version ships.
* **A dated crawl is not a pinned version.** `research_versions` reports a
  crawl date, not an upstream release, for the ten corpora marked "dated
  crawl" above. For those, read the version out of the page title or URL of
  a hit instead. Call `research_versions` at the start of version-sensitive
  work; if the user is on a different Isaac or Kit version, say so and treat
  hits as approximate.

## Proactive triggers

Kick off `research_search` for any ask in this table — do not wait to be
asked twice. The "where it lives" column is what the diversified default
will surface, not a filter to set:

| User says / asks about | Where the answer lives |
|---|---|
| Camera, Imu, LiDAR, RTX sensor, render product | `isaac-sim-source` + `isaac-sim-docs` |
| Articulation, joint config, PD gains, drives, contacts | `isaac-sim-source` + `physx-docs` |
| Isaac Lab envs, managers, XYZW quaternions, RL config | `isaac-lab-source` + `isaac-lab-docs` |
| RL training loop, PPO config, runner, policy export | `rsl-rl-source` + `skrl-docs` |
| Warp kernels, `wp.array`, ProxyArray internals, kernel launch | `warp-source`, `warp-docs` |
| USD prim, schema, layer, composition, xform, materials | `openusd-docs` + `omniverse-docs` |
| PhysX solver, contact reports, soft body, cloth | `physx-docs` (+ `isaac-sim-source` for the wrapping) |
| Kit settings, extensions, commands, viewport, timeline | `kit-docs` + `omniverse-docs` |
| Replicator annotators, writers, render products | `omniverse-docs` + `isaac-sim-source` |
| Newton, differentiable physics, MPM | `newton-docs` |
| Motion planning, collision-free trajectories, cuRobo | `curobo-docs` + `curobo-source` |
| ROS 2 bridge, cuVSLAM, Apriltag, nav2 | `isaac-ros-docs` |
| Cosmos / WFM / synthetic data | `cosmos-docs` |
| NuRec / neural reconstruction | `nurec-docs` |
| `.rrd`, Rerun blueprint, telemetry channel | `rerun-docs` |

## The tool loop

1. **`research_search(query)`** — the default entry. Every hit is an
   evidence card carrying `artifact=<id> ordinal=<n>`; those handles are
   accepted verbatim by the other tools.
2. **`research_expand(artifact, ordinal)`** — the hit is relevant but
   incomplete: fetch the neighboring methods of the class, the adjacent
   sections of the page.
3. **`research_open(artifact)`** — the whole implementation or full page is
   needed, not just the matching chunk.
4. **`research_artifacts(query)`** — whole-file/page ranking for broad
   intents: "the entire Articulation class", "the full camera tutorial".
5. **`research_grep(pattern)`** — token matching over everything indexed:
   symbols, error strings, and config keys when semantic search is too fuzzy.
   Multi-word patterns are AND-of-tokens, so rows can match every token
   without holding the string. Only the negative case is labelled: a row
   tagged `[tokens match; exact string not in this chunk]` is a token-only
   hit, and an unlabelled row is verbatim. Then open a matching artifact
   with `research_open`.
6. **`research_versions()`** — what is indexed, at which pins.

Scores are relative to one query, never across queries: a 0.86 on a long
multi-clause question is not weaker evidence than a 0.95 on a short one.
`research_artifacts` scores are whole-artifact sums and routinely exceed
1.0, so never compare them against `research_search` scores.

A worked loop — localizing a sensor bug:

```text
research_search(
  query="CameraSensor.get_data returns None for the rgb annotator right
  after timeline play in Isaac Sim 6 — what does the experimental RTX
  camera sensor require between play and the first get_data call, and
  where does the render product warm-up happen in the sensor runtime?",
  kind="source",
)
# hit: isaac-sim-source :: .../camera_sensor.py :: CameraSensor.get_data
#      artifact=abc123 ordinal=7
research_expand(artifact="abc123", ordinal=7)   # neighboring methods
research_open(artifact="abc123")                # the whole runtime, if needed
```

**Pass `kind='source'` when localizing code for a bug or feature.** On a
107-task edit-recall benchmark mined from real Isaac Lab PRs,
task-aware `kind='source'` filtering measured **+5.6 points** of recall@10
over the unfiltered baseline. It is not unconditional: when a query is
already code-shaped, ranking returns source anyway and the filter only
removes the docs safety net — a measured probe kept ranks 1-5 identical and
lost a relevant docs hit at rank 6. Set it when docs pages are crowding out
code, not by reflex. Use `kind='docs'` for conceptual and tutorial
questions.

## How to write a good query

The retriever rewards detail, multi-clause prose, and explicit symbols. The
reranker needs signal to discriminate; a 4-word query gives it almost none.
Write queries like you would ask a senior engineer for help.

Good queries:

```text
research_search(
  query="How do I configure an Articulation in Isaac Sim 6 so that joint
  drives apply position targets with explicit stiffness, damping, and
  max-effort gains that survive a play/stop reset cycle? What is the
  canonical pattern for setting PD gains on a wheeled robot or arm?",
  kind="source",
)
```

```text
research_search(
  query="In OpenUSD, what is the precedence rule for mass and inertia when
  both UsdPhysicsMassAPI and a child geometry's mass-density are authored
  on the same rigid body, and how does the resolver decide which density
  to use?"
)
```

What makes a query good:

* **Multi-clause and detailed.** One dense sentence plus a follow-up clause
  beats two short queries.
* **Name the symbols you suspect.** `UsdPhysicsMassAPI`,
  `set_drive_stiffness`, `Replicator render product` — every named symbol
  gives the sparse channel an exact match.
* **State the use case plus the question.** "I'm tuning PD gains for a
  wheeled robot" + "what is the canonical pattern" beats "PD gains".
* **Broad first, then drill.** No `corpus=` on the first pass; use `kind=`
  from the start when the intent is clearly code-localization or clearly
  conceptual.

What makes a query bad:

* **Speculative clauses.** Detail narrows; it does not automatically
  improve. Every clause you add is a claim about what the answer contains.
  A correct clause earns a precise hit; a guessed one trades away breadth.
  Short queries are not automatically bad — "drive stiffness" alone returns
  the PhysX articulation-drive reference and the gain tuner. Add detail you
  are confident about, then drill.
* **Corpus-filter-first.** The diversified default finds the right corpus
  for you.
* **Asking for an answer instead of the relevant text.** "Tell me X" does
  not retrieve; "what is the documented behavior of X and where does it
  differ from Y" does.

## Citing results

Every hit carries `corpus@version`, a path, a symbol, and (for docs
corpora) a public `source_url`. Cite the source URL and symbol so the user
can verify — the value of the tool is the grounding, not the prose.

## When to fall back to web research

One non-web routing rule first: anything about Antioch itself — the CLI,
projects, machines, run history, or the asset catalog ("find me a warehouse
asset") — is the `antioch-platform` skill, not this index.

* The topic is a vendor or product not in the corpora table.
* The user runs a different version than the pins (Isaac Sim 5.x, another
  Kit) and the delta itself is the question.
* Papers, GitHub issues, release announcements, roadmap or licensing
  questions — the index holds docs and source, not the discourse around
  them.

When you fall back, say so: an answer from the web is not grounded in the
pinned index and must be labelled as such.

## Registration notes

Claude Code and Codex load the MCP from the plugin manifest automatically. For
a harness that does not load the manifest, register the SDK executable:

```
codex mcp add antioch-research -- antioch-research-mcp
```

The first research call prompts for approval; a non-interactive (stdin-null)
run cancels it unless the harness's approval mode allows MCP calls.

Indexed corpora are public vendor documentation and Apache-2.0 source at
pinned versions; reproducing retrieved files verbatim to the user, with
their citations, is the intended use.

## Failure modes

* **503 / "research is not configured"** — this Antioch deployment does not
  host the research service. Surface it; do not work around it.
* **401 / "not authenticated"** — the local credential store is missing or
  stale. Ask the user to run `antioch auth login`; the server re-reads the
  store on every call, so no restart is needed.
* **Evidence-floor abstention / "not in the index"** — Antioch Research returns no hits
  when the best rerank score is below the evidence floor. Tell the user the
  answer is outside the indexed corpora. Do not improvise from adjacent hits
  or present them as evidence; call `research_versions` when the boundary is
  unclear, then fall back to web research and label it as outside the index.
* **Empty results** — a plain `no results` means the pool was empty, often
  because an unknown corpus filter was used. Do not treat it as evidence.
* **Warp version drift** — `warp-docs` tracks the upstream `stable` docs
  (1.16.0), which is ahead of Isaac Lab 3.0's `warp-lang==1.13.0` pin.
  For pin-exact signatures and behavior, prefer `warp-source` (v1.13.0)
  with `kind="source"`; use `warp-docs` for concepts and explanations.
* **Hits from the wrong version** — the user runs Isaac Sim 5.x or another
  Kit: mention the pin mismatch and verify externally.

## Anti-patterns

* **Guessing an indexed API from memory.** The search is seconds; a wrong
  guess is a burned turn.
* **Short queries.** Spend the ten extra seconds writing a detailed query
  — it pays back in retrieval quality.
* **Corpus-filtering the first pass.** The default diversifies across all
  19 corpora; drill in only after a broad search told you where the answer
  lives.
* **`research_open` as a first move.** Search first; open only the
  artifact a card pointed at. Opening whole files speculatively floods the
  context.
* **Ignoring the artifact handles.** Re-searching to "see more of that
  file" wastes a round trip — `expand` and `open` take the handle you
  already have.
