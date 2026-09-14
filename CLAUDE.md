# CLAUDE.md

Guidance for Claude Code (claude.ai/code) and other AI coding agents working in this repository.

## What this repository is

A source export of X's recommendation algorithm — the services, models, and
frameworks that build the For You Timeline and Recommended Notifications.
Licensed AGPL-3.0 (`COPYING`). Start with `README.md` for the component table
and the system diagram in `docs/system-diagram.png`.

This is an **export of a subtree of X's internal monorepo**, not a standalone
project. That single fact drives most of the guidance below.

## Critical: this repository does not build, run, or test

Do not attempt `bazel build`, `bazel test`, `sbt`, `cargo build`, or "just run
the tests" — and do not propose them as verification steps in a plan. They will
fail for structural reasons, not fixable ones:

- There is **no top-level `WORKSPACE`, `BUILD`, or `.bazelrc`**. The ~968
  `BUILD`/`BUILD.bazel` files are per-directory targets with no workspace root
  to anchor them. `README.md` states this is intentional.
- `BUILD` files depend on internal monorepo paths that are **not vendored here**
  — `finagle/…`, `finatra/…`, `twitter-server/…`, `3rdparty/jvm/…`. None of
  those directories exist in this tree.
- `ci/ci.sh` is a stub: `#!/bin/sh` / `exit 0`. There is no real CI.
- Test sources exist (~32 Scala test files, mostly under
  `unified_user_actions/`), but there is no runner that can execute them.

**How to verify work instead:** read the surrounding code, keep changes
type-consistent with the Scala/Java signatures they touch, trace call sites with
`grep`, and check that new symbols actually exist where you reference them.
Reviewers verify by reading. Say plainly in your summary that a change is
unbuilt and unrun rather than implying it was tested.

## Repository map

| Path | What lives there |
|---|---|
| `home-mixer/` | Main Home Timeline service. Built on Product Mixer. Where For You is assembled. |
| `product-mixer/` | The pipeline framework everything else is written against (`core`, `component-library`, `shared-library`). |
| `src/scala/com/twitter/simclusters_v2/` | SimClusters — community detection, sparse user/tweet embeddings. |
| `src/scala/com/twitter/recos/` | GraphJet-based candidate sources (UTEG, etc.). |
| `src/scala/com/twitter/interaction_graph/` | RealGraph — user-to-user interaction prediction. |
| `src/scala/com/twitter/graph/` | Graph batch jobs, incl. tweepcred (PageRank reputation). |
| `src/java/com/twitter/search/` | Earlybird search index and its light ranker. |
| `src/python/`, `twml/` | Legacy TensorFlow v1 ML tooling (DeepBird, TWML). |
| `navi/` | Rust model-serving (four crates: `navi`, `segdense`, `dr_transform`, `thrift_bpr_adapter`). The only Rust in the repo. |
| `pushservice/` | Recommended Notifications service + its light/heavy rankers. |
| `visibilitylib/` | Visibility filtering library — hard filters, downranking, product treatments. |
| `trust_and_safety_models/` | NSFW / abusive / toxicity model code. |
| `tweetypie/` | Core tweet read/write service. |
| `timelines/` | Aggregation framework for batch/realtime features. |
| `unified_user_actions/` | Realtime user-action event stream. Has the most real test coverage. |
| `cr-mixer/`, `tweet-mixer/`, `follow-recommendations-service/` | Candidate coordination and follow recs. |
| `representation-manager/`, `representation-scorer/`, `graph-feature-service/`, `topic-social-proof/`, `user-signal-service/`, `simclusters-ann/`, `ann/`, `recos-injector/`, `timelineranker/` | Supporting feature/embedding/serving services. |

Rough composition: ~4,900 Scala, ~1,000 Java, ~180 Python, ~30 Rust files, plus
Thrift IDL (`*.thrift`) and protobufs.

## Architecture: the Product Mixer model

Nearly all serving code is a Product Mixer pipeline. Read
`product-mixer/README.md` before changing anything in `home-mixer/`. The request
flow is:

```
Product Pipeline  → picks the Mixer/Recommendation Pipeline for the request
  Recommendation Pipeline
    Candidate Pipelines  → fetch candidates (in-network, out-of-network, ads…)
    Scoring Pipelines    → hydrate features, score, rank
    Selectors            → interleave, dedupe, truncate
    Decorators/Marshallers → shape the response (URT)
```

Components live under `functional_component/` and are strictly typed by role.
Both `product-mixer/core/.../functional_component/` and
`home-mixer/.../functional_component/` use the same taxonomy:

`candidate_source`, `feature_hydrator`, `filter`, `gate`, `scorer`, `selector`,
`decorator`, `transformer` / `query_transformer`, `side_effect`, `marshaller`,
`premarshaller`, `access_policy`, `configapi`.

**Put new logic in the component type that matches its role**, and register it in
the relevant pipeline config — don't inline behavior into a pipeline config file.

Home Timeline products are `home-mixer/server/src/main/scala/com/twitter/home_mixer/product/`:
`for_you`, `following`, `scored_tweets`, `subscribed`, `list_tweets`,
`list_recommended_users`. Each mirrors the same internal layout
(`candidate_pipeline/`, `scorer/`, `filter/`, `selector/`, `param/`, …) — follow
the sibling product's structure when adding to one.

## Conventions

- **Scala 2**, Finagle/Finatra/Guice idioms. Futures are `com.twitter.util.Future`,
  not `scala.concurrent.Future`.
- **One import per line, fully qualified, alphabetized. No wildcard or brace-grouped
  imports.** This is consistent across the whole tree; match it exactly.
- Directory structure mirrors the package path
  (`…/src/main/scala/com/twitter/<pkg>/…`).
- **Every source directory has its own `BUILD`/`BUILD.bazel`.** If you add a file
  in a new directory, add the BUILD target too and wire its `dependencies` — even
  though nothing here can build it, the internal monorepo does, and reviewers check.
- Tunables are **params**, not literals: see
  `home-mixer/.../param/` (`HomeGlobalParams.scala`, `decider/`) and per-product
  `param/` dirs. New knobs go through `com.twitter.timelines.configapi` params,
  deciders, or feature switches.
- Identifiers (`ComponentIdentifier`, `PipelineIdentifier`, etc.) are explicit
  and must be unique — follow the naming already used by neighbors.
- No license headers on source files; don't add them.

## Finding things

```bash
grep -rn "ClassName" --include="*.scala" .          # locate a definition/usages
find . -path "*home_mixer*" -name "*Filter*.scala"  # components by role
find . -name "README.md" | head -40                 # ~79 md files; most dirs are documented
```

Per-directory `README.md` files are the best source of truth for each component
and are generally accurate. Prefer them over inference.

## What is *not* in this repository

Be accurate about scope; do not invent files or claim edits here change
production behavior that lives elsewhere. Absent from this export:

- Trained model weights, training data, and user data.
- The ML training code for the heavy ranker and TwHIN — those are in the separate
  [`the-algorithm-ml`](https://github.com/twitter/the-algorithm-ml) repository.
- Ads ranking and serving.
- Trust & Safety operational infrastructure: the actual URL/domain denylists,
  spam heuristics, abuse rules, enforcement tooling, and the policy data they
  read. `visibilitylib/` and `trust_and_safety_models/` contain *library and
  model code*, not the rule or blocklist data. Requests to "unblock domain X" or
  "change what gets filtered" cannot be satisfied by editing this repo — there is
  no such data here to edit.
- Internal build infrastructure (`3rdparty/`, `finagle/`, `finatra/`, Aurora
  deploy config referenced by `*.aurora` / `*.d6w` files).

## Working agreements

- **Scope discipline.** This is a large codebase with heavy cross-service
  coupling and no compiler to catch you. Make the smallest change that satisfies
  the request; don't opportunistically refactor.
- **Match the neighbors.** Import style, naming, file layout, and BUILD structure
  are extremely uniform. Copy the closest sibling file's shape.
- **State verification honestly.** Since nothing can be built or run here, never
  report a change as "tested" or "verified working". Say what you read and
  reasoned about.
- **Security issues** go to X's [HackerOne bug bounty](https://hackerone.com/x),
  not to a public issue or PR. Don't write exploit details into this repo.
- **Upstream contribution.** `README.md` invites issues and PRs, but upstream
  `twitter/the-algorithm` has been effectively inactive for a long time — open
  PRs against your own fork's branch and expect upstream merges to be unlikely.
