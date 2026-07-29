# Billing comprehension demo

A single-file static microsite for a telecoms product team. It has two jobs: explain the digital billing AI opportunity to executive stakeholders, and demonstrate the thing working inside a phone frame at the foot of the page. A customer messaging about their bill reaches an agent that, instead of reading a nine-line breakdown aloud, draws components onto the screen — a bill summary, then a usage and allowance card that proves an out-of-plan charge visually. Everything is mock data.

**Two phone frames sit side by side, showing two ways in.** They exist so a director can watch both and say which one we build.

| | Pattern A — in the app | Pattern B — in the chat |
| --- | --- | --- |
| Entry | Messaging offers a deep link | Messaging hands over in place |
| Where it happens | The native bills screen | The chat window itself |
| Modality | Voice | Text |
| Where components appear | A slide-out sheet over the app | Inline in the chat transcript |
| Journey | chat → app → voice → chat | chat → chat |

The comparison is the point. Pattern A moves the customer from chat to voice and back to chat again, which stakeholders have said is a heavy transition. Pattern B keeps them in text the whole way, at the cost of the app context the agent is working on top of.

Each frame runs independently: its own session, its own script, its own replay button, its own end marker. Starting one does not touch the other, and you can run both at once.

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
- With `mode: "live"`, anyone who finds the URL and presses **Open my bills** starts a real session against that deployment, which consumes quota. **There are now two frames, so a visitor who starts both opens two sessions and consumes two lots of quota.** If the page is going somewhere discoverable, set `mode` to `"simulated"` and flip it to `"live"` only for the machine driving a session.

## Running the demo

**Frame A.** Tap **Open my bills** in the messaging conversation. The agent turn plays, then a **Next** button appears in the caption strip above the AI bar — customer turns wait for you, so you control the pacing. **Replay A** under the phone resets that frame, including any live session.

**Frame B.** Tap **Continue in chat**. The messaging window is replaced where it stands by a full-screen agent surface — nothing navigates and no app opens. In scripted mode the customer's next line is loaded into the composer as a draft: tap the send arrow to play it, which is Frame B's equivalent of Frame A's **Next**. **Replay B** resets that frame.

The two frames are independent. Run them one after another to talk through each journey, or start both and let them play alongside each other.

## One agent serves both frames

**No second agent, no second deployment, no extra tools and no agent-side change of any kind.** Both frames run against the same `deploymentId` and register the same three tool resource names. Everything that differs between the two patterns is a decision this page makes.

- **Client functions are registered per element.** `registerClientSideFunction({ toolName, toolDisplayName }, handler)` is called on a `<ces-messenger>` element, not globally. The same resource name registered on two elements is two independent registrations. The agent calls `render_genui_component` with `{ component_name, component_json }`; *where* the result is drawn is entirely page-side. Frame A's handler calls `Sheet.show(node)`. Frame B's calls `insertMessage('BOT', { payload: { html } })`.
- **Modality is attribute configuration.** Frame A: `audio-input-mode="DEFAULT_OFF"`, `audio-output-mode="DEFAULT_ON"`, plus a `voice`. Frame B: `audio-input-mode="NONE"`, `audio-output-mode="DISABLED"`, no `voice`. Both keep `modality="chat"` — `modality="call"` is documented as incompatible with `NONE`.
- **`close_genui_surface` has no meaning in a transcript.** Frame B's handler is a no-op returning `{ status: "CLOSED" }`. It stays registered so the agent needs no change.
- **The catalogue is the same but for one field.** `getCatalog()` is page-side, so Frame B returns the identical component list with `surface: "web_chat_inline"` instead of `"app_voice_slideout"`. That is the only agent-visible difference between the frames.

### What Frame B configures

Frame B has one setting, directly under `CONFIG`:

```js
const CONFIG_B = {
  mode: "inherit"                 // "inherit" | "simulated" | "live"
};
```

`"inherit"` runs Frame B in whatever `CONFIG.mode` is. Set it to `"simulated"` to run Frame B from its script while Frame A stays live — that is the fallback described below, and it is also the cheaper way to demo.

### Two live sessions on one page

Two frames live at once means two sessions against the same deployment, which is **two lots of quota**. For a stakeholder session where only one journey needs to be live, set `CONFIG_B.mode = "simulated"` and Frame B plays from its script with no backend at all.

Tool calls stay correctly separated, because registration is per element. Widget *events* are the open question: Frame A listens for `ces-*` on `window`, so if Frame B's widget dispatched its events there too, Frame A would react to Frame B's turns.

Frame B is built so this cannot happen: it catches every `ces-*` event on its own host element and calls `stopPropagation()`, so an event belonging to Frame B never reaches `window`. Frame A's own events do not pass through that host and are untouched. That works as long as the widget dispatches from its element, which is the normal behaviour for a web component.

**If it dispatches straight onto `window` instead, the frames cannot be told apart.** The page detects this rather than leaving it to be discovered as odd behaviour: `fbAuditEventIsolation()` runs itself six seconds into every live session and writes its verdict to the debug log. Open `?debug`, start Frame B live, and look for the line beginning `[Frame B] isolation audit:`.

- *"…caught at its own host and stop there"* — isolated, both frames can run live.
- *"…the widget dispatches its events straight onto window"* — set `CONFIG_B.mode = "simulated"` and run only Frame A live.

You can also call `window.__demoB.auditIsolation()` in the console at any point.

### What has not been checked against the real widget

Everything above about the two frames was verified against a stand-in `<ces-messenger>` implementing the documented API, in headless Chromium. **It has not been run against the real deployment**, because the environment the change was built in has no outbound access to `gstatic.com`, so the widget script could never load. One run with `?debug` against the real deployment settles it — that is what the isolation audit is for.

### Getting a component into the transcript

Two things do not survive the trip into the widget's shadow root. Both are handled page-side; neither needed a change to the shared component builders.

- **Page CSS does not cross a shadow boundary.** A component injected with `insertMessage` would otherwise render completely unstyled. Frame B harvests the card rules from this page's own stylesheet at runtime and hands them to the widget as `custom-css`, so there is still one source of truth for the styling rather than a duplicated copy.
- **`<use href="#i-calendar">` resolves against the tree it sits in**, and the icon sprite is in the document, not the shadow root. Every `<use>` is replaced with the referenced symbol's own contents before the HTML is serialised.

One further difference: `buildUsageAllowanceCard` fills its bar from a `requestAnimationFrame` closure over the node it built, which cannot survive being serialised to HTML and re-parsed somewhere else. Inside Frame B the same fill is driven from CSS, so it works in both the scripted and the live path.

### The sanitiser, and why it does not matter here

`insertMessage` HTML is sanitised by the widget, and a sibling project has seen `data-*` attributes stripped. **What the real widget strips has not been established here** — see above. Frame B is built so the answer does not change anything.

The phrase behind each action button is kept on the page, in `fbState.actionMap`, keyed by the label the customer reads. Nothing in the injected HTML has to survive except the visible text. `data-send` is consulted only if it happens to have made it through, and the debug log records which of the two recovered the phrase (`recoveredFrom: "data-send"` or `"label lookup"`).

A tap is wired by two routes, since which survives cannot be known in advance:

1. `registerRichMessageHandler(templateId, (message, clickedElement) => …)`, the documented route.
2. A delegated click listener on the widget's shadow root, matched on the class names the shared builders already emit.

Both funnel through one handler that ignores a repeat of the same phrase within 700ms, so whichever survives, the tap fires exactly once. It then calls `insertMessage('USER', …)` to echo the turn and `sessionInput()` to send it — `sessionInput` does not echo the customer on its own, so without the first call a tap looks like it did nothing.

As in Frame A, the render handler returns immediately and never waits on a tap. A tap arrives later as an ordinary user turn.

## Connecting a live agent

Everything lives in the `CONFIG` block at the very top of the `<script>` in `index.html`, and applies to **both** frames:

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

- If `mode` is `"simulated"`, or `deploymentId` is empty, no `<ces-messenger>` element is created at all — no session, no token request, no billable connection. Confirm it in the elements inspector. The same is true of Frame B, which additionally honours `CONFIG_B.mode`.
- Any tool id left empty is simply not registered, in either frame, and the agent falls back to speech or plain text for that capability.
- Each widget element is created lazily, on the press that starts that frame, never on page load. Frame A's opens a persistent bidirectional audio stream as soon as it connects, which is exactly what is wanted there — it is the voice transport — and is why the customer has to opt in first. Frame B's is text only and opens no audio stream.
- Values are trimmed before use, in both frames. A resource name pasted with a leading space is otherwise a name that never matches, and nothing tells you why.
- The dual project-number/project-id registration below applies to both frames.

### Project number or project id

A tool is matched by its exact resource name, and a project can be addressed either by its number (`projects/667634947008/…`) or by its id (`projects/project-bf6872b8-…/…`). The console and the runtime do not always use the same one. If the `deploymentId` and the tool ids disagree, the page registers each tool under **both** spellings and logs the mismatch to the debug panel. Registering a name the agent never calls costs nothing; missing the one it does call looks exactly like the agent ignoring the tool.

The `<app>` segment must match across the deployment and all three tools — a tool belongs to an app, and a tool id carrying a different app id than the deployment will never be called no matter which project spelling is used.

If you would rather have one canonical form, make the project segment in `tools.*` match the one in `deploymentId` and the second registration stops happening.

### Publish before you test

A deployment serves a **published** agent version, not your working draft. Save your changes, verify them in the simulator, publish a new version, then point the deployment at that version. Only then will this page see them. Testing against an unpublished draft is the usual reason a tool call never arrives.

When a tool call does not arrive, open `?debug` and look for `ces-tool-call-received`. That event is the ground truth that a tool fired, and it carries the resource name the agent actually used — the console display name is not reliable for this. With two frames, check the prefix on the line: `[Frame B]` entries belong to Frame B, unprefixed ones to Frame A.

### The three client-side functions

Each frame registers whichever of these are configured, on its own element, on `ces-messenger-loaded`:

- `get_genui_catalog` — returns the catalogue descriptor, including the real JSON schema for every component, so the agent never has to hold the schemas in its instruction. Identical in both frames but for the `surface` field.
- `render_genui_component` — takes `{ component_name, component_json }`, validates the payload against that component's schema, and either renders it and returns `{ status: "RENDERED", component }` immediately, or returns `{ status: "INVALID", errors: [...] }` with specific, actionable messages so the agent can correct and retry once. The render call returns straight away and never waits on a customer tap; a tap comes back later as an ordinary user turn. Frame A renders into the slide-out; Frame B into the chat transcript.
- `close_genui_surface` — dismisses the slide-out in Frame A. In Frame B it is a no-op returning `{ status: "CLOSED" }`, because components in a transcript are part of the conversation and are not dismissed.

Validation, the component schemas and the component builders are shared between the frames — the same payload produces the same card in both. What is not shared is anything that touches a frame's own state or surface.

## URL flags

| Flag | Effect |
| --- | --- |
| `?debug` | Opens the debug panel on load. It logs widget events, tool calls with full arguments, validation results, render outcomes and state transitions for **both** frames, newest first. Frame B's entries are prefixed `[Frame B]`; unprefixed entries are Frame A's. Also on `window.__demoLog`. |
| `?phase2` | Makes the phase 2 tactical entry points (**Explain my bill**, **Query this charge**) live rather than inert. They start the agent with `entry_point: "in_app_tactical"` and, where relevant, the charge id. Frame A only — the phase 2 markers live on the bills screen, which Frame B never opens. |

Combine them: `index.html?debug&phase2`.

## Console handles

| Handle | Frame |
| --- | --- |
| `window.__demo` | Frame A — `renderComponent`, `closeSurface`, `getCatalog`, `sheet`, `state`, `replay`, `fixture`, `config` |
| `window.__demoB` | Frame B — `renderComponent`, `closeSurface`, `getCatalog`, `componentCss`, `componentHtml`, `auditIsolation`, `state`, `replay`, `config` |
| `window.__demoLog` | Both, newest first |

`window.__demoB.componentHtml('usage_allowance_card', node)` returns exactly the HTML string Frame B hands to `insertMessage`, which is the quickest way to see what the sanitiser is being given.

## Illustrative data

All figures, the account number, the device and the usage are invented for this demo. Nothing here is O2 pricing, and no real customer data is used. Demo "today" is hard-coded as Tuesday 14 July 2026 so the demo never drifts.
