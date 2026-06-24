# React frontends (SPA / Next)

These are **minimal run/deploy contracts** so the preview runtime can detect and serve
your app automatically. They do **not** constrain how you write components, structure
state, choose libraries, or test — that is all yours (floor, not ceiling).

Detection works by reading your dependencies and config — no manifest needed for the
common cases below.

## What runs out of the box

The framework you depend on determines the build output directory that gets served:

| Framework | How it's detected | Build script | Served from |
|---|---|---|---|
| **Vite** | `vite` in dependencies | `build` script → | `dist/` |
| **Create React App** | `react-scripts` in dependencies | (works even without one) | `build/` |
| **Next (static export)** | `next` dep + `output: 'export'` in `next.config.*` | (detected by config) | `out/` |
| **Next (server / SSR)** | `next` dep + `output: 'standalone'` or default | built server process | single port |

- **Vite / CRA / Next-export** build to a static directory; the runtime serves that
  directory directly. You do **not** write a server.
- **Next server (standalone or default)** runs as a long-lived process. It must bind to
  the `PORT` environment variable (see `networking.md`). The runtime builds and starts it
  for you; you don't hand-roll the start command for the standard layout.

Vite-style build-to-static projects are detected by their `build` script. CRA (from
`react-scripts`) and Next (from the `next` dependency plus its config) are recognized by
their dependency, so detection does not hinge on the `build` script name for them — though
Next still runs one to produce its output.

## Common failures

1. **SPA deep-link 404** — a client-routed SPA with no fallback breaks on direct visits
   to nested paths. Serve `index.html` for navigation requests (history-API fallback) so
   `/some/route` loads the app instead of returning 404.
2. **Secrets inlined into the client bundle** — build-time env values are baked into the
   shipped JS and visible to anyone. Only put non-secret, intentionally-public values in
   build-time variables; keep real secrets server-side (see `env-and-secrets.md`).
3. **Missing `build` script (Vite-style)** — a Vite-style build-to-static project with no
   standard `build` script is not detected and won't preview. Keep the conventional script
   name. (CRA and Next are recognized by their dependency instead.)
4. **Customized output directory** — if you change the build output away from the
   framework default (`dist` / `build` / `out`), detection looks in the wrong place.
   Either keep the default, or declare it explicitly in `preview.toml` (see
   `preview-toml.md`).

## Checklist

- [ ] Standard `build` script in `package.json` (Vite-style; CRA and Next are recognized by their dependency).
- [ ] Build output left at the framework default, **or** declared in `preview.toml`.
- [ ] SPA serves `index.html` as a fallback for client routes.
- [ ] No secrets in build-time / client-side variables.
- [ ] Next: pick a mode explicitly — `output: 'export'` (static) vs `'standalone'`
      (server) — and for the server mode, bind to `PORT`.
