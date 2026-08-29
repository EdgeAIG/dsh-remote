# Fix: Subagents unusable after disabling the official DeepSeek adapter

Applies to the product patch shared by:

- `DHS-M/dsh-remote-web` (scratch installer)
- `DHS-M/dsh-official-modifier` (existing-checkout modifier)

## Problem

The product patch removes `deepseek-official` from the active composition,
but two things still pointed at that removed route:

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
  session's latest `request/header` config (the live route the parent
  actually used) before falling back to creation-time options.

## Patches

Each file is a `git format-patch` commit ready to apply in its product repo:

- `subagent-model-inheritance-dsh-remote-web.patch`
- `subagent-model-inheritance-dsh-official-modifier.patch`

Because the Arena/GitHub token has no write access to `DHS-M/*`, the commits
were created locally in sandbox clones but could not be pushed. Apply these
patches manually from machines with write access:

```bash
# in a dsh-remote-web checkout
git am subagent-model-inheritance-dsh-remote-web.patch

# in a dsh-official-modifier checkout
git am subagent-model-inheritance-dsh-official-modifier.patch
```

The signed-off-by footer is omitted; add your own identity if `git am`
requires it, or use `git apply`.
