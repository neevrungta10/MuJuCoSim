<!-- LOVABLE:BEGIN -->
> [!IMPORTANT]
> This project is connected to [Lovable](https://lovable.dev). Avoid rewriting
> published git history — force pushing, or rebasing/amending/squashing commits
> that are already pushed — as it rewrites history on Lovable's side and the
> user will likely lose their project history.
>
> Commits you push to the connected branch sync back to Lovable and show up in
> the editor, so keep the branch in a working state.

## What Lovable manages

Rules outside this block are the project's own. Change the pieces below in
Lovable, not by hand. With the [Lovable MCP](https://mcp.lovable.dev)
connected, `send_message` asks Lovable to do it; otherwise ask the user to
use the editor.

- **Database.** Do not add or edit `supabase/migrations/` or `drizzle/`.
  Ask Lovable to change the schema.
- **Generated files.** Do not edit a file whose header says it is automatically
  generated.
- **AI gateway.** Keep model calls on Lovable's gateway, and keep
  `LOVABLE_API_KEY` out of browser code.
- **Connectors.** Lovable provides native connectors. `list_connectors` shows
  which are enabled; `add_connector` only returns the dashboard link. When a
  connector is enabled, calls go through the connector gateway and need
  `LOVABLE_API_KEY`. When Lovable does not provide one, users can bring their
  own key and store it in Secrets.
- **Secrets.** Lovable manages them. Do not put secret values in the repo.
<!-- LOVABLE:END -->

## Project rules

- The simulation lives in `src/sim/` (MJCF builder, controller, Three.js renderer, engine) and is plain TS with no React; UI in `src/components/sim/` only talks to `SimEngine`. Why: the controller and model run unchanged in Node tests and in the browser.
- `src/sim/engine.ts` (MuJoCo WASM + Three.js) must only be loaded via dynamic `import()` inside an effect. Why: SSR runs on a Worker without WebGL/WASM loading.
- `@mujoco/mujoco` stays in `optimizeDeps.exclude` and its `.wasm` is passed via `?url` + `locateFile`. Why: pre-bundling breaks the loader's `import.meta.url` lookup.
- Robot joints are driven by MuJoCo `position` actuators whose ctrl = q_des + (kv/kp)·qd_des, with gravcomp on all robot links. Why: implicit PD with velocity feed-forward tracks fast toss strokes stably.
- Grasps use weld equality constraints whose relpose is written at activation (stride `mj.mjNEQDATA`). Why: freezes the apple where it touches the hand instead of snapping.
- `src/sim/toss.test.ts` runs a full headless cycle; keep it passing after any model or controller change.
