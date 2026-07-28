# Billing comprehension demo — voice plus generative UI

A single-file static microsite for a telecoms product team. It has two jobs: explain the digital billing AI opportunity to executive stakeholders, and demonstrate the thing working inside a phone frame at the foot of the page. A customer messaging about their bill is deep-linked into the app's bills screen, a voice agent picks up what they already said, and instead of reading a nine-line breakdown aloud it draws components onto the screen — a bill summary, then a usage and allowance card that proves an out-of-plan charge visually. Everything is mock data.

Two files, no build step: `index.html` (all HTML, CSS and JavaScript) and this README. No frameworks, no bundler, no npm. The only external resource is the Google CES messenger web component, and it is fetched lazily — nothing is requested at all until a live agent session is started.

## Deploy to GitHub Pages

This repository holds an unrelated project as well, so the demo lives in `docs/` and Pages is pointed at that folder. **Settings → Pages → Build and deployment → Deploy from a branch**, then pick the branch and set the folder to **`/docs`**.

That publishes `docs/` and nothing else — the terraform, the React frontend and the backend agent definitions are never served. The site root is the folder itself, so the demo is at `https://<owner>.github.io/<repo>/`, not `/docs/`.

Pages only ever offers two folders, `/ (root)` and `/docs`; there is no way to nominate an arbitrary one. If you are dropping these two files into a repository of their own, put them at the root and choose `/ (root)` instead.

`.nojekyll` sits alongside them so Pages serves the files as they are rather than running them through Jekyll.

**The scripted demo works with nothing connected.** Set `CONFIG.mode` to `"simulated"` and the whole conversation plays from a script with no backend, no token request and no `<ces-messenger>` element on the page. Use this mode in stakeholder sessions — it is the reliable one, and it is a one-word change at the top of the file.

`CONFIG.mode` is currently `"live"` against a configured deployment. Nothing is created or requested on page load either way; a session only opens when the customer presses **Open my bills**. If the widget cannot be reached, the page falls back — a toast offers **Run scripted demo instead**, which switches to the scripted path without a reload.

### Before you publish this on a public URL

A Pages site is public even when the repository is private. Two consequences worth deciding on deliberately:

- The `CONFIG` block ships to the browser, so the deployment id and the tool resource names are readable by anyone with the URL. They are not credentials — the managed token broker handles authentication — but they do identify the deployment.
- With `mode: "live"`, anyone who finds the URL and presses **Open my bills** starts a real session against that deployment, which consumes quota. If the page is going somewhere discoverable, set `mode` to `"simulated"` and flip it to `"live"` only for the machine driving a session.

## Running the demo

Tap **Open my bills** in the messaging conversation. The agent turn plays, then a **Next** button appears in the caption strip above the AI bar — customer turns wait for you, so you control the pacing. **Replay demo** under the phone resets everything, including any live session.

## Connecting a live agent

Everything lives in the `CONFIG` block at the very top of the `<script>` in `index.html`:

```js
const CONFIG = {
  mode: "live",                   // "simulated" | "live"
  deploymentId: "projects/<p>/locations/eu/apps/<app>/deployments/<dep>",
  tokenBrokerUrl: "MANAGED",
  voice: "en-GB-Chirp3-HD-<name>",   // empty = widget default
  languageCode: "en-GB",
  tools: {
    get_genui_catalog:      "projects/<p>/locations/eu/apps/<app>/tools/<id>",
    render_genui_component: "projects/<p>/locations/eu/apps/<app>/tools/<id>",
    close_genui_surface:    "projects/<p>/locations/eu/apps/<app>/tools/<id>"
  }
};
```

Where each value comes from:

| Value | Where to find it |
| --- | --- |
| `deploymentId` | The channel page for the app in the console — the full `projects/…/deployments/…` resource name of the web widget deployment. |
| `tools.*` | Each tool's **Integration** tab in the console, which shows that tool's full resource name. The resource name goes in `toolName`, not the display name. |
| `voice` | Optional. Leave empty to use the widget default. |

Rules the page enforces for you:

- If `mode` is `"simulated"`, or `deploymentId` is empty, no `<ces-messenger>` element is created at all — no session, no token request, no billable connection. Confirm it in the elements inspector.
- Any tool id left empty is simply not registered, and the agent falls back to speech for that capability.
- The widget element is created lazily, on the deep-link press, never on page load. It opens a persistent bidirectional audio stream as soon as it connects, which is exactly what is wanted here — it is the voice transport — and is why the customer has to opt in first.
- Values are trimmed before use. A resource name pasted with a leading space is otherwise a name that never matches, and nothing tells you why.

### Project number or project id

A tool is matched by its exact resource name, and a project can be addressed either by its number (`projects/667634947008/…`) or by its id (`projects/project-bf6872b8-…/…`). The console and the runtime do not always use the same one. If the `deploymentId` and the tool ids disagree, the page registers each tool under **both** spellings and logs the mismatch to the debug panel. Registering a name the agent never calls costs nothing; missing the one it does call looks exactly like the agent ignoring the tool.

The `<app>` segment must match across the deployment and all three tools — a tool belongs to an app, and a tool id carrying a different app id than the deployment will never be called no matter which project spelling is used.

If you would rather have one canonical form, make the project segment in `tools.*` match the one in `deploymentId` and the second registration stops happening.

### Publish before you test

A deployment serves a **published** agent version, not your working draft. Save your changes, verify them in the simulator, publish a new version, then point the deployment at that version. Only then will this page see them. Testing against an unpublished draft is the usual reason a tool call never arrives.

When a tool call does not arrive, open `?debug` and look for `ces-tool-call-received`. That event is the ground truth that a tool fired, and it carries the resource name the agent actually used — the console display name is not reliable for this.

### The three client-side functions

The page registers whichever of these are configured, on `ces-messenger-loaded`:

- `get_genui_catalog` — returns the catalogue descriptor, including the real JSON schema for every component, so the agent never has to hold the schemas in its instruction.
- `render_genui_component` — takes `{ component_name, component_json }`, validates the payload against that component's schema, and either renders it and returns `{ status: "RENDERED", component }` immediately, or returns `{ status: "INVALID", errors: [...] }` with specific, actionable messages so the agent can correct and retry once. The render call returns straight away and never waits on a customer tap; a tap comes back later as an ordinary user turn.
- `close_genui_surface` — dismisses the slide-out.

## URL flags

| Flag | Effect |
| --- | --- |
| `?debug` | Opens the debug panel on load. It logs widget events, tool calls with full arguments, validation results, render outcomes and state transitions, newest first. Also on `window.__demoLog`. |
| `?phase2` | Makes the phase 2 tactical entry points (**Explain my bill**, **Query this charge**) live rather than inert. They start the agent with `entry_point: "in_app_tactical"` and, where relevant, the charge id. |

Combine them: `index.html?debug&phase2`.

## Illustrative data

All figures, the account number, the device and the usage are invented for this demo. Nothing here is O2 pricing, and no real customer data is used. Demo "today" is hard-coded as Tuesday 14 July 2026 so the demo never drifts.
