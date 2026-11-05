# HANDOFF — casehub-ops

## Last Session

Rewrote Levels 05–09 to universal runtime (#229, #230, #231, #232, #233). All deployment-domain levels now use `casehubio/casehub-runtime:latest` with role-based config. Level 08 was silently using `claudony_app` node type — fixed to `casehub_application`. Level 09 collapsed 7 fictional nodes to 4 real service boundaries (platform, ledger, qhorus, chat-viewer). Blog entry written: "Config, Not Code."

## Immediate Next Step

Remaining epic #226 levels: #234 (Levels 10, 12, 14 — claudony domain) and #235 (Level 15 — governed interaction). These use different SPI node types (`claudony_app`, `claudony_pool`, `channel`) and need separate design decisions — not a mechanical rewrite.

## References

- Blog: `blog/2026-11-05-mdp01-config-not-code.md`
- Prior entries: `blog/2026-11-03-mdp01-one-image-to-rule-them-all.md`, `blog/2026-11-04-mdp01-the-test-is-the-tutorial.md`
- Strategy doc: `docs/guides/topology-levels-strategy.md`
