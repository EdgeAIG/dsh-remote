# Fix: Subagents unusable after disabling the official DeepSeek adapter + OpenCode Zen bridge

Applies to the product patch shared by:

- `DHS-M/dsh-remote-web` (scratch installer)
- `DHS-M/dsh-official-modifier` (existing-checkout modifier)

## Problem

The product patch removes `deepseek-official` from the active composition,
but two things still resolved that removed route:

1. `packages/bundle/base/cordis.patch.yml` configured `agent-default-model`
   with `provider: deepseek-official`.
2. `packages/subagent/subagent/src/child-agent.ts` resolved a child agent's
   provider/model/maxTokens from the parent's **creation-time** `parent.options`,
   which came from that default. So subagent children attempted
   `deepseek-official` even when the parent session was using an available
   provider, and failed with:

   `no adapter registered for provider "deepseek-official"`

## Fix

- Repoint the default agent model from `deepseek-official` to the pi-ai
  multi-provider `deepseek` route (same model family).
- Resolve in-process subagent provider/model/maxTokens from the parent
  session's latest `request/header` config (the live route the parent actually
  used) before falling back to creation-time options.
- Update two `apiproxy` test fixtures for the new `readDocument`/`writeDocument`
  `SettingsApi` methods (the shipped patch did not type-check without this).
- Add a focused regression test (`child-agent-route.spec.ts`).

## OpenCode Zen bridge (opt-in, dsh-remote-web only)

`0002-Add-opt-in-DSH-OpenCode-Zen-bridge.patch` and
`0004-Restrict-OpenCode-Zen-bridge-to-the-model-adapter-on.patch` add the
`DHS-M/dsh-opencode-zen` integration to the scratch installer behind
`OPENCODE_ZEN=1`. The bridge installs **only** the standalone
`@deepseek-ai/dsh-llm-opencode` model adapter (web-search plugins are
intentionally excluded), registers it in the bundle manifests and host TS
references, points the default model at `opencode-zen` / `big-pickle`, and
normalizes the adapter: it removes `export default apply` (which otherwise
breaks the DSH Cordis loader) and rewrites the SSE tool-call name accumulation
(`state.name += raw.function.name`) to an overwrite, which otherwise records
`web_searchweb_search` and rejects every model tool call.

## Apply

Because the Arena/GitHub token has no write access to `DHS-M/*`, these commits
were created in local sandbox clones but not pushed to `DHS-M`. They are
committed on the `arena/01a04c30-dsh-remote` branch of `EdgeAIG/dsh-remote`
under `fixes/`.

```bash
# dsh-remote-web checkout
cd /path/to/dsh-remote-web
git am /path/to/this/dir/dsh-remote-web/*.patch

# dsh-official-modifier checkout
cd /path/to/dsh-official-modifier
git am /path/to/this/dir/dsh-official-modifier/*.patch
```

If `git am` requires an identity, add your own `--signoff` or apply with
`git apply` per file.

## Verification performed here

- `pnpm install` on the product-patched upstream (`b150a551…`) succeeds after
  adding the bridge packages.
- `pnpm run build:lib:host`, `build:lib:client`, and `build:web` all succeed
  (with `NODE_OPTIONS=--max-old-space-size=6144` because the host build exceeds
  Node's default 2 GB heap).
- Focused subagent route-inheritance test passes (3/3).
- `dsh web --host 0.0.0.0 --open-authority --no-open` boots with the OpenCode
  bridge and serves on port 3080.

The sandbox cannot reach `opencode.ai` (egress blocked), so no live Zen traffic
was run here; the adapter route (`opencode-zen` / `big-pickle`) boots and
registers, and the tool-call/schema and subagent route-inheritance behavior was
verified against a temporary local stand-in that has since been deleted. The
real endpoint must be tested on a machine with outbound access; the product
integration does not ship or require any mock.
