# Billing comprehension demo

A single-file static microsite for a telecoms product team. It has two jobs: explain the digital billing AI opportunity to executive stakeholders, and demonstrate the thing working inside a phone frame at the foot of the page. A customer messaging about their bill reaches an agent that, instead of reading a nine-line breakdown aloud, draws components onto the screen — a bill summary, then a usage and allowance card that proves an out-of-plan charge visually. Everything is mock data.

**Two phone frames sit side by side, showing two ways in.** 

| | Pattern A — in the app | Pattern B — in the chat |
| --- | --- | --- |
| Entry | Messaging offers a deep link | Messaging hands over in place |
| Where it happens | The native bills screen | The chat window itself |
| Modality | Voice | Text |
| Where components appear | A slide-out sheet over the app | Inline in the chat transcript |
| Journey | app → AWS chat → GECX voice → AWS chat | app → AWS chat → GECX chat → AWS chat |

The comparison is the point. Pattern A moves the customer from chat to voice and back to chat again, which is a heavy transition. Pattern B keeps them in text the whole way.

Each frame runs independently: its own session, its own script, its own replay button, its own end marker. Starting one does not touch the other, and you can run both at once.

Two files, no build step: `index.html` (all HTML, CSS and JavaScript) and this README. No frameworks, no bundler, no npm. The only external resource is the Google CES messenger web component, and it is fetched lazily — nothing is requested at all until a live agent session is started.

## Deployed to GitHub Pages

**The scripted demo works with nothing connected.** Set `CONFIG.mode` to `"simulated"` and the whole conversation plays from a script with no backend, no token request and no `<ces-messenger>` element on the page.

`CONFIG.mode` is currently `"live"` against a configured deployment. Nothing is created or requested on page load either way; a session only opens when the customer presses **Open my bills**. If the widget cannot be reached, the page falls back — a toast offers **Run scripted demo instead**, which switches to the scripted path without a reload.

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

Frame B has two settings, directly under `CONFIG`:

```js
const CONFIG_B = {
  mode: "inherit",                // "inherit" | "simulated" | "live"
  messengerSrc: ""                // empty = the same gstatic build Frame A uses
};
```

`"inherit"` runs Frame B in whatever `CONFIG.mode` is. Set it to `"simulated"` to run Frame B from its script while Frame A stays live — that is the quota fallback described below, and it is also the cheaper way to demo.

`messengerSrc` is the widget build Frame B loads inside its iframe. Leave it empty for the gstatic URL; point it at a self-hosted copy to pin a version.

### Two live sessions on one page — why Frame B runs in an iframe

Two `<ces-messenger>` instances in one document interfere. This is no longer a hypothesis: it was observed live (a Frame A reply was spoken by Frame A **and** written into Frame B's transcript), then pinned down in the widget's source (`GoogleCloudPlatform/ces-messenger`, `src/BidiWidget.ce.vue`) and a captured HAR of the broken session. Three mechanisms, none fixable from the page:

- **Sessions are silently shared.** The widget picks its session id client-side, saves it — with the whole transcript — to `sessionStorage` under the fixed key `'bidi-widget-session'`, and loads that state on mount. Whichever widget mounts second adopts the first one's session, so the server streams one conversation to both frames. That is exactly the both-frames symptom.
- **Every event goes to `window`.** All `ces-*` events are dispatched with `window.dispatchEvent(...)`, so Frame A's listeners hear Frame B's turns, and nothing on the event tells the frames apart. (An earlier revision tried to intercept events at Frame B's host element; the source shows they never pass through it.)
- **The widget positions itself against the viewport.** Its container is `position: fixed; top: -60px; right: 40px`, which inside a phone mock renders as a clipped sliver in the corner.

Frame B therefore runs its widget inside a **same-origin iframe** filling the phone screen, which answers all three at once:

- the iframe has its own `window`, so Frame B's events arrive there — already attributed — and Frame A's listeners are physically out of reach;
- an in-memory `sessionStorage`/`localStorage` shim is installed in the iframe before the widget script runs, so Frame B's widget sees an empty store, generates a fresh session id, and its saves never touch the real storage Frame A's widget uses;
- the iframe viewport is the phone screen (390px wide), which lands in the widget's own `max-width: 768px` styles — the same full-screen mobile layout it uses on a real phone. Nothing fights the widget's positioning.

Frame A still runs its widget in the top document, exactly as before, and owns the real `sessionStorage`.

`fbAuditEventIsolation()` runs itself six seconds into every Frame B live session and writes its verdict to the debug log: it checks that events are arriving on the iframe's window and that Frame A's saved session id and Frame B's differ. Open `?debug` and look for `[Frame B] isolation audit:`, or call `window.__demoB.auditIsolation()` in the console. If it ever reports a shared session, `CONFIG_B.mode = "simulated"` remains the one-word fallback.

### Quota, and the 429 seen in testing

Two frames live at once is two sessions against the same deployment — **two lots of quota**. In live testing the CES API returned `429 RESOURCE_EXHAUSTED` on `runSession` after a handful of turns: that is a project quota on the API, not a fault in the page or the agent, and the demo project's quota needs raising before a two-frames-live stakeholder run.

The widget surfaces a failed send only as an apology bubble in its own transcript — it emits no event — so Frame B watches its own iframe's network (a pass-through wrap of `fetch`, observation only) and reports failures honestly: the header status shows *Quota exceeded — wait a moment and retry*, the debug log carries the status code, and after three consecutive failures the status points at **Replay B**, which runs the scripted version. For a session where only one journey needs to be live, set `CONFIG_B.mode = "simulated"`.

### Getting a component into the transcript

Two things do not survive the trip into the widget's shadow root. Both are handled page-side; neither needed a change to the shared component builders.

- **Page CSS does not cross a shadow boundary.** A component injected with `insertMessage` would otherwise render completely unstyled. Frame B harvests the card rules from this page's own stylesheet at runtime and hands them to the widget as `custom-css`, so there is still one source of truth for the styling rather than a duplicated copy.
- **`<use href="#i-calendar">` resolves against the tree it sits in**, and the icon sprite is in the document, not the shadow root. Every `<use>` is replaced with the referenced symbol's own contents before the HTML is serialised.

One further difference: `buildUsageAllowanceCard` fills its bar from a `requestAnimationFrame` closure over the node it built, which cannot survive being serialised to HTML and re-parsed somewhere else. Inside Frame B the same fill is driven from CSS, so it works in both the scripted and the live path.

### The sanitiser — now established from source

`insertMessage` runs `payload.html` through DOMPurify with a custom attribute allowlist: `href, title, class, style, target, rel, src, controls, width, height, autoplay, muted`. Two consequences:

- `data-*` is stripped, confirming what the sibling project saw;
- so is everything SVG (`viewBox`, `d`, `stroke`, …) — an inline icon comes out as empty tags.

It also honours an escape hatch: a payload marked `safe: true` skips sanitisation entirely. Frame B uses it, and the justification matters: the HTML it injects never contains raw agent output. It is serialised from a DOM the shared builders assembled node by node, where every agent-supplied string entered through `textContent` after validating against the component's schema. Sanitising it again would only strip the icons and the `data-send` attributes off something already safe by construction.

A tap on an action inside an injected card is wired by two routes:

1. `registerRichMessageHandler('genui_component', (message, clickedElement) => …)` — the widget's own route. Frame B injects each card with `templateId: 'genui_component'` in the payload, and the widget's click handler routes any click inside it to the registered handler. One contract detail read from the source: the widget dereferences the handler's return value, so the handler always returns an object.
2. A delegated click listener on the widget's shadow root, matched on the class names the builders already emit.

On current builds both fire for the same tap, so they funnel through one handler that drops the repeat (same phrase within 700ms). Two routes stay wired so a future build changing either one degrades to a working tap, not a dead button. The phrase is read from `data-send` (which survives under `safe: true`), falling back to a page-side label → phrase map — so even a future build that ignores `safe` and strips everything still sends the right turn; it just loses the icons. The debug log records which route and which source fired (`via`, `recoveredFrom`).

The handler then calls `insertMessage('USER', { text })` to echo the turn — as plain text, no markup, no sanitiser involved — and `sessionInput()` to send it; `sessionInput` does not echo the customer on its own, so without the first call a tap looks like it did nothing.

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
- Each widget is created lazily, on the press that starts that frame, never on page load. Frame A's opens a persistent bidirectional audio stream as soon as it connects, which is exactly what is wanted there — it is the voice transport — and is why the customer has to opt in first. Frame B's is text only, opens no audio stream, and is created inside its own iframe (see above).
- Values are trimmed before use, in both frames. A resource name pasted with a leading space is otherwise a name that never matches, and nothing tells you why.
- The dual project-number/project-id registration below applies to both frames.

### Project number or project id

A tool is matched by its exact resource name, and a project can be addressed either by its number (`projects/667634947008/…`) or by its id (`projects/project-bf6872b8-…/…`). The console and the runtime do not always use the same one. If the `deploymentId` and the tool ids disagree, the page registers each tool under **both** spellings and logs the mismatch to the debug panel. Registering a name the agent never calls costs nothing; missing the one it does call looks exactly like the agent ignoring the tool.

The `<app>` segment must match across the deployment and all three tools — a tool belongs to an app, and a tool id carrying a different app id than the deployment will never be called no matter which project spelling is used.

If you would rather have one canonical form, make the project segment in `tools.*` match the one in `deploymentId` and the second registration stops happening.

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

`window.__demoB.componentHtml('usage_allowance_card', node)` returns exactly the HTML string Frame B hands to `insertMessage`, which is the quickest way to see what the widget is being given. Frame B's live widget lives inside the iframe under `#fbCesmHost` — inspect it via `window.__demoB.state.iframe.contentWindow`.

## Provenance of the widget findings

The interference mechanisms, the sanitiser allowlist, the `safe`/`templateId` contract and the event dispatch target were read from the widget's public source (`GoogleCloudPlatform/ces-messenger` — `src/BidiWidget.ce.vue`, `src/bidi/*.js`) and cross-checked against a HAR capture of a real broken two-frame session, which also showed the full tool loop working over the text channel (`get_genui_catalog` called and answered with `surface: "web_chat_inline"`). The gstatic bundle is built from that repository; if a future build changes these internals, the isolation audit and the degraded tap route are the designed safety nets.

## Illustrative data

All figures, the account number, the device and the usage are invented for this demo. Nothing here is O2 pricing, and no real customer data is used. Demo "today" is hard-coded as Tuesday 14 July 2026 so the demo never drifts.
