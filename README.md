# ship-ready · universal baseline

You cloned the `universal/20260624` branch. It ships only conventions, no app code:

- **`AGENTS.md`** — the run/deploy contract every app here follows (agents read this first).
- **`conventions/`** — per-topic guides the contract routes you to (stacks, datastores, env, networking, deployment, and a `preview.toml` escape hatch).

Build your app on top. The contract keeps it easy to **run locally** and **ship as a single container** out of the box, without constraining how you design or test your code.

When your app is ready, replace this README with your project's own — including the green-field local run steps (migrations + seed) and any dummy accounts the seed creates, as required by `AGENTS.md`.
