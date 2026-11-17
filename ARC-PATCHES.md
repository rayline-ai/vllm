# ARC patches to vLLM

This branch is vLLM with a small patch stack used by Atlas Futures' ARC (Adaptive Routing Controller) plugin.
Branch `arc/v<version>` starts at upstream tag `v<version>`; each patch is one commit on top of it. This file is
a documentation commit after the patches: it is not part of any release patch.

## Notice (Apache License 2.0, section 4(b))

This is a modified version of vLLM. The files listed below were modified by Atlas Futures, Inc., 2026. vLLM is
licensed under the Apache License 2.0 (see `LICENSE`); the modifications are under the same license.

## Patch 0001: incremental pooling sessions (streaming input and in-place extend)

Commit `dca0ed16e564ab048405888952fd86f56ef9335a` on vLLM v0.33.0 (`4ba787ec9959e566d3b587a796dff07673324da5`).

**What it changes.** A pooling request can take streaming input, as generation requests already can: each update
carries new tokens and resumes the paused session with that update's `PoolingParams` (per-input readout offsets are
passed in `PoolingParams.extra_kwargs["arc_gather"]`). On the V2 GPU model runner a resumed text-only request is
extended in place: only the new token ids, their M-RoPE positions and the new block ids are written. An update that
brings a multimodal item takes the existing full re-add.

**Why.** ARC scores a growing conversation turn by turn with a pooling model. Stock vLLM either restarts the request
(recomputing the whole prefix) or, with streaming input, re-adds the whole request on every append, which costs
O(session length) per input. The patch keeps the session's KV and state and makes each append cost only its new
tokens.

**Guards.** Streaming pooling input requires `extra_kwargs["arc_gather"]` (a stock MEAN/ALL pooler would silently pool
the new input alone) and refuses prefix-cache reads. The in-place path falls back to stock behaviour for generation,
pipeline parallelism, speculative decoding, prompt embeddings, non-extending updates and new multimodal items.

**Files.**

| File | Change |
|---|---|
| `vllm/v1/engine/async_llm.py` | accepts an input stream for pooling requests; validates the `arc_gather` key and no prefix-cache reads; clones `PoolingParams` once; ends the session on stream end or generator close |
| `vllm/v1/core/sched/scheduler.py` | a pooling session keeps every token on resume (nothing was sampled) and takes the update's pooling params |
| `vllm/v1/request.py` | a streaming update carries its own `pooling_params` |
| `vllm/v1/worker/gpu/model_runner.py` | `_arc_extend_request`: extends a resumed pooling request in place, else falls back to the full re-add |

## Releases and reproducing the patch

A release `arc-v<version>-<n>` tags the last patch commit (never this doc commit) and carries the stack as one asset,
`arc-v<version>.patch`. For `arc-v0.33.0-1` (sha256 `91d39ed57058f9ef53ad1967af99424db7b75080a1c072ab6f29ada0d50c6103`):

```
git fetch origin tag v0.33.0 tag arc-v0.33.0-1
git format-patch --stdout v0.33.0..arc-v0.33.0-1 > arc-v0.33.0.patch
sha256sum arc-v0.33.0.patch
```

The output is byte-identical with git 2.47.3; the last line of a `format-patch` file is the git version, so another
git version differs only there. The same files are on branch `arc-patches` under `patches/`.

## A new vLLM release

1. Branch `arc/v<version>` from upstream tag `v<version>` (push the upstream tag to this repository too).
2. Rebase the patch commits onto it and fix conflicts; keep one commit per patch.
3. Tag the last patch commit `arc-v<version>-<n>` and publish a release of that name whose asset is
   `git format-patch --stdout v<version>..arc-v<version>-<n>`; add the same file to `arc-patches`.
4. Re-pin the plugin's `engine.lock` to the new tag, commit and asset sha256.

A weekly canary in the plugin repository checks that the current patch still applies to the latest vLLM release.

## Upstream status

Not yet proposed upstream. The plan is to offer it in two steps: first generic streaming input for pooling requests,
then the in-place extend on the V2 model runner.
