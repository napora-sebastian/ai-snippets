# Copilot Repository Instructions

Use `AGENTS.md` as the shared repository bootstrap for GitHub Copilot, Codex, Cursor agents, and GitNexus.

Before changing code:

1. Read `AGENTS.md`.
2. Read the base Cursor rules referenced there.
3. Read any label-specific Cursor rules from the user request.
4. Use GitNexus for exploration, context, impact analysis, rename safety, and pre-commit change detection as described in `AGENTS.md`.
5. Include the rules read, GitNexus repo queried, impact result, files changed, tests run, and remaining risks in handoff notes.

Keep detailed rules in the Cursor rule files and GitNexus index instead of duplicating them here.

<!-- rtk-instructions v2 -->
# RTK — Token-Optimized CLI

**rtk** is a CLI proxy that filters and compresses command outputs, saving 60-90% tokens.

## Rule

Always prefix shell commands with `rtk`:

```bash
# Instead of:              Use:
git status                 rtk git status
git log -10                rtk git log -10
cargo test                 rtk cargo test
docker ps                  rtk docker ps
kubectl get pods           rtk kubectl pods
```

## Meta commands (use directly)

```bash
rtk gain              # Token savings dashboard
rtk gain --history    # Per-command savings history
rtk discover          # Find missed rtk opportunities
rtk proxy <cmd>       # Run raw (no filtering) but track usage
```
<!-- /rtk-instructions -->
