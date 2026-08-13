---
name: rum-instrument
description: Instrument a web frontend so you can answer "what did this user actually do" — page views, clicks, JS errors and fetch spans, tied to a session id and to the backend traces they triggered. Trigger phrases include "add RUM", "set up real user monitoring", "instrument the frontend", "add browser monitoring", "track what users are doing", "why did this user see an error", "add session tracking", "instrument my React/Next/Vue app", or any variant naming OnePatch as the destination ("install OnePatch in my frontend", "add OnePatch RUM"). Installs `@onepatch/rum`, wires identity, decides trace propagation per backend after probing it, writes the tests, and documents the result in the repo's `TELEMETRY.md`. For backend/server instrumentation use `otel-instrument` instead.
---

# Instrument a frontend with `@onepatch/rum`

You are wiring browser telemetry into a web app so its behaviour becomes queryable: which pages, which clicks, which failed requests, which JS errors, in order, per session, per user.

**What this produces is an action list, not a video.** No DOM is captured, no session replay, no keystrokes. That is the design: an investigation reads the ordered list of what someone did, and skipping the recorder keeps the payload small and the privacy story ordinary.

This skill is published by **OnePatch** (`app.onepatch.dev`) and installs its browser package, which wraps the OpenTelemetry browser SDKs and speaks plain OTLP. For server-side instrumentation, use the `otel-instrument` skill instead — the two are complementary and share the same ingest endpoint.

Work the phases in order. Phases 5 and 7 are the ones that make this predictable rather than merely done; do not skip them.

## 1. Discover the frontend

Read the repo and identify the browser surface and how it boots:

- `next.config.*` → Next.js. Note the version — 15+ has `instrumentation-client.ts`.
- `vite.config.*` + `react` → Vite React. Entry is `src/main.tsx`.
- `react-scripts` in `package.json` → Create React App. Entry is `src/index.tsx`.
- `nuxt.config.*` → Nuxt. Use a client plugin.
- `svelte.config.*` → SvelteKit. Use `src/hooks.client.ts`.
- `vue` + `vite` → Vue. Entry is `src/main.ts`.
- `angular.json` → Angular. Entry is `src/main.ts`.
- No framework, plain `<script>` tags → use the prebuilt bundle (phase 3).
- `expo` / `react-native` → **stop.** This package needs browser APIs and will not work in React Native. Say so plainly and offer the `otel-instrument` skill for their backend instead.

A monorepo can hold several frontends. Instrument one at a time, and say which one you picked in a sentence before writing anything.

## 2. Get the ingest configuration

You need two values:

- **`ingestUrl`** — the tenant's OnePatch ingest URL, shaped `https://<slug>.logger.onepatch.dev`.
- **`ingestToken`** — an `op_…` token.

Three paths, in order:

**2a. The user pasted them.** OnePatch's onboarding has a *"Copy command for your coding agent"* button whose payload contains both inline. Extract with `https://[a-z0-9-]+\.logger\.onepatch\.dev` and `op_[A-Za-z0-9_-]+`.

**2b. The repo already has them.** If `otel-instrument` has already run against this repo there will be an `OTEL_EXPORTER_OTLP_ENDPOINT` in an env file, and its host is the same ingest URL. Reuse it. The token may be in the same file — reuse that too rather than asking.

**2c. Neither.** Ask the user to sign up at `app.onepatch.dev` and paste the copy-command payload. Do not guess a URL, and do not proceed with a placeholder: a wrong endpoint produces a silent nothing, which is the single most expensive failure mode here.

### About the token

`op_…` is a write-only, append-only, single-tenant ingest bearer, designed the way a Sentry DSN is designed: **it belongs in the frontend bundle.** Commit it, inline it, put it in a public env var (`NEXT_PUBLIC_*`, `VITE_*`). Do not route it through a server proxy and do not treat it as a secret — someone trying to hide it will end up building an unnecessary proxy and blaming the SDK when spans stop arriving.

No other credential goes in a browser. If the user offers an API key that is not `op_`-shaped, refuse it and ask for the ingest token.

## 3. Install and initialise

```sh
bun add @onepatch/rum   # or npm / pnpm / yarn
```

Three rules govern placement, and they are the whole content of this phase:

1. **Once.** `startRum` must run exactly once per page load. Twice is a warning and a wasted call.
2. **Client-side only.** It no-ops outside a browser, so importing it from server-rendered code is safe, but the *call* belongs on a client path.
3. **Early.** Before the app's own `fetch` calls, or the first few go untraced.

Three values decide whether the data is usable a month from now, so get them right rather than plausible:

- **`appName` names the app after the service it belongs to: `<service>-web`.** If the backend is `acme-api`, this is `acme-web`. It becomes `service.name`, which is the first column of the sort key and the thing the service map draws — an unrelated name puts the two halves of one trace in two unrelated places.
- **`environment` comes from wherever the backend reads its own.** Not a hand-typed `"production"` in a file that gets copied to staging. If the two halves disagree, every env-filtered query returns half a trace, and nobody notices because both halves individually look fine.
- **`appVersion` is the commit sha.** Every build system already has one: `NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA`, `VITE_COMMIT_SHA` fed from `$GITHUB_SHA`, or `git rev-parse --short HEAD` at build time. It becomes `service.version`, which is what turns "errors went up at 14:20" into "errors went up on this deploy". If the app has no build-time sha available, wire one in — that is part of this phase, not a follow-up.

### Next.js 15+

Create `instrumentation-client.ts` at the project root — Next runs it on the client before hydration, which is exactly right:

```ts
import { startRum } from "@onepatch/rum";

startRum({
  ingestUrl: process.env.NEXT_PUBLIC_ONEPATCH_INGEST_URL!,
  ingestToken: process.env.NEXT_PUBLIC_ONEPATCH_INGEST_TOKEN!,
  appName: "<service>-web",
  // On Vercel these two come free as NEXT_PUBLIC_VERCEL_ENV and
  // NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA. Elsewhere, define them in the build.
  environment: process.env.NEXT_PUBLIC_APP_ENV!,
  appVersion: process.env.NEXT_PUBLIC_COMMIT_SHA!,
  connectTracesTo: [/* phase 5 decides this — leave it out until then */],
});
```

### Next.js 13–14

No `instrumentation-client.ts`. Add a client component and render it in the root layout:

```tsx
// app/onepatch-rum.tsx
"use client";
import { startRum } from "@onepatch/rum";

let booted = false;
if (typeof window !== "undefined" && !booted) {
  booted = true;
  startRum({ /* … */ });
}

export function OnepatchRum() {
  return null;
}
```

The module-scope guard is deliberate: an effect in a `<StrictMode>` tree runs twice in development.

### Vite / CRA / Vue / Angular

Top of the entry module (`src/main.tsx`, `src/index.tsx`, `src/main.ts`), before the app mounts and before any other import that might fetch.

### SvelteKit

`src/hooks.client.ts`, at module scope.

### Nuxt

`plugins/onepatch-rum.client.ts` — the `.client` suffix keeps it off the server.

### No bundler

```html
<script src="https://unpkg.com/@onepatch/rum/dist/onepatch-rum.min.js"></script>
<script>
  OnePatchRum.startRum({
    ingestUrl: "…", ingestToken: "op_…",
    appName: "<service>-web", environment: "production", appVersion: "<commit-sha>",
  });
</script>
```

Place it in `<head>`, before the app's own scripts.

## 4. Wire identity — the highest-value five minutes

A session with no user on it answers almost nothing. Find where the app already knows who is signed in and call `identifyUser` there.

Grep for what is already doing this job — the answer is usually sitting in one of these:

- `analytics.identify(` / `posthog.identify(` / `mixpanel.identify(`
- `Sentry.setUser(` / `datadogRum.setUser(`
- a Datadog RUM `beforeSend` that reads `localStorage`
- the auth provider's callback: `onAuthStateChange`, `useUser`, `session` from a context

Put the call on the same path:

```ts
import { identifyUser } from "@onepatch/rum";

identifyUser({
  email: user.email,
  id: user.id,
  orgId: user.orgId,
  // anything else worth filtering sessions by
  plan: user.plan,
});
```

`id`, `email`, `name`, `orgId`, `orgName` become the conventional `user.*` / `org.*` attributes. Everything else passes through under the key you wrote.

Call it again when the user changes. On sign-out, call `identifyUser({ id: null, email: null })` — an explicit `null` clears the attribute, whereas omitting the key leaves the previous value stamped on subsequent spans, which is how one person's session ends up labelled as another's.

**Do not stamp anything you would not put in a log line.** No tokens, no full addresses, no free-text the user typed.

## 5. Decide `connectTracesTo` — probe before you list

This is the phase that can break the customer's application if you get it wrong, so it has its own rules.

Joining a browser span to its backend span means attaching a `traceparent` header. Same-origin requests get that automatically, with nothing configured and nothing at risk — **if the app's API is same-origin, skip this phase entirely and leave `connectTracesTo` out.**

Cross-origin is where the hazard is. Adding the header makes the request preflighted, and a backend whose `Access-Control-Allow-Headers` does not cover `traceparent` causes the browser to refuse the request outright. Not untraced — refused. The API call never happens.

The library will not let that happen at runtime: it probes each listed origin at startup and connects only the ones that pass. But it can only report the result, and a silently unconnected backend is a bug the user should hear about now, from you, not discover in three weeks. So probe from the terminal too, where the response headers are actually readable.

### The probe

Pick a route that **exists** — an API path the frontend really calls. Probing `/` is the standard mistake: it often 404s ahead of the CORS middleware and tells you nothing.

```sh
curl -s -D - -o /dev/null -X OPTIONS "https://api.acme.com/v1/things" \
  -H "Origin: https://app.acme.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: traceparent" \
  | grep -i "^access-control-"
```

Read the answer against this table. The last two rows are the ones people get wrong.

| Response | Verdict |
| --- | --- |
| `allow-headers` names `traceparent` | Safe. List the origin. |
| `allow-headers: *`, `allow-origin: *` | Safe. A wildcard origin cannot receive credentialed requests at all, so there are none to break. List it. |
| `allow-headers: *`, `allow-credentials: true`, `allow-origin` echoed | **Not safe.** A wildcard is illegal for a credentialed request, so cookie-bearing calls carrying `traceparent` will be blocked while the same calls without it succeed. Do not list it. Ask for `traceparent` to be named explicitly. |
| `allow-headers` omits `traceparent` | Not safe. Do not list it. Offer to open a PR widening that backend's CORS. |
| No `access-control-*` headers at all | Inconclusive — you probably probed a route that 404s before CORS runs. Find a real route and probe again. |

List only the origins that passed. Never a wildcard: the library rejects `connectTracesTo: ["https://*"]` outright, because it would attach trace headers to every third-party request the page makes — the payment provider, the CDN, analytics — and any one of them refusing the header breaks that request.

Tell the user, in one line per origin, which backends will be trace-joined and which will not and why. For the ones that will not, offer the CORS one-liner for their framework.

## 6. Name the actions worth naming

Clicks and navigations are captured already, and the click span carries the element and its xpath. What it cannot carry is what the click *meant*.

Add `recordAction` at the handful of places that have a name in the product's own vocabulary — the step someone would actually search for:

```ts
recordAction("ran-workflow", { workflowId });
recordAction("submitted-onboarding", { step: 3 });
```

Three to six of these is right for a first pass. Find them by looking for the mutations, the submit handlers, and whatever the product's own analytics already tracks. Do not wrap every button.

Also wrap handled errors that would otherwise vanish:

```ts
catch (error) { recordError(error, { where: "checkout" }); }
```

## 7. Write the tests

Instrumentation is code that fails silently, so it gets tests. Write these into the repo's existing test setup (vitest, jest, bun test — match what is there). Four tests, in descending order of what they catch:

**7a. It starts, and every listed backend is connected.** The one that catches a broken endpoint, a bad token shape, a wildcard, and CORS drift:

```ts
import { startRum } from "@onepatch/rum";
import { rumOptions } from "../src/rum-config"; // export the options object so tests can see it

test("RUM starts and connects every backend it was told to", async () => {
  const status = await startRum(rumOptions);
  expect(status.started).toBe(true);
  expect(status.error).toBeUndefined();
  for (const backend of status.backends) {
    expect(backend.allowed, `${backend.origin}: ${backend.detail}`).toBe(true);
  }
});
```

Refactor phase 3 so the options live in one exported object rather than inline at the call site — that is what makes this test possible, and it is the only structural change this phase asks for.

This test reaches the network. If the repo's unit tests must stay offline, keep the assertions on the options and move the backend loop into a separate pre-deploy check; say which you did.

**7b. Identity reaches the spans.** Assert `identifyUser` is called on the auth path, with the fields you wired, and that sign-out clears them. Mock `@onepatch/rum` and assert on the call.

**7c. It boots once, on the client.** Assert the init module is imported exactly once, and — for Next/Nuxt/SvelteKit — that it is on a client-only path. A grep-shaped test is fine and catches a real regression: someone adding a second `startRum` in a provider.

**7d. No wildcard creeps into `connectTracesTo`.** A one-line assertion that no entry contains `*`. The library enforces this at runtime; the test makes a well-meaning "let's just match all our subdomains" edit fail in CI instead of in production.

Run them. Report the actual result — if 7a fails because a backend is unconnected, that is the finding, not a test to relax.

## 8. Verify in a real browser

Tests are not proof that telemetry arrives.

1. Boot the app (`bun dev`).
2. Open it, sign in, click something, navigate once.
3. In the console, confirm the session id: `OnePatchRum.sessionId()` if you used the script bundle, otherwise check for the SDK's debug output with `debug: true` temporarily set.
4. Ask the user to check their OnePatch workspace: *"you should see a service called `<service>-web` with `click` and `documentLoad` spans within about ten seconds."*
5. Check the spans carry an environment and a version, not blanks. A page that reports as `service.version: ""` is instrumented but unattributable, and that is only obvious now — later it just looks like the data is bad.

If nothing arrives:

- A CORS error on the ingest URL in the console → the endpoint is wrong. Check the host.
- HTTP 401 from the ingest URL → wrong token.
- No requests to the ingest URL at all → `startRum` is not running. It is on a server path, or the module is not imported.
- `startRum` logged a `[onepatch/rum]` error → read it; it names the fix.

If a backend was meant to be trace-joined, confirm it: trigger a request to it, find the span, and check that the backend's own span shares the trace id. One end-to-end trace is worth more than any amount of configuration review.

## 9. Add a browser section to `TELEMETRY.md`

**`TELEMETRY.md` at the repo root is the one place a repo describes what it emits** — written by `otel-instrument`, kept fresh by the `telemetry-docs-freshness` monitor, and front-loaded into the agent's context whenever a telemetry skill runs. The browser is another emitter in that repo, so it goes in that file. Do not start a second document; a repo with two telemetry docs has one that is out of date.

**One doc per repo, describing that repo.** A frontend in its own repo gets a `TELEMETRY.md` that is browser-only — that is complete, not half-written, so don't apologise for it or invent placeholder server sections. If the file doesn't exist yet, create it with just the sections below.

The corollary matters more: **when the frontend is a separate repo, the backend's `TELEMETRY.md` will never mention the browser.** Nothing reads both docs and stitches them together, so the propagation table below is the only place the FE↔BE join is written down. Name the backend *services* there, not just origins, so someone reading from either side can find the other:

```markdown
| `https://api.acme.com` | yes | `Access-Control-Allow-Headers` names `traceparent` — spans join `service.name = acme-api` (repo `acme/backend`) |
```

And say it out loud in phase 10: **the frontend repo has to be connected in OnePatch too.** Both docs reach the agent only because it reads every connected repo; an unconnected frontend repo is invisible no matter how good its doc is.

Append, under a `## Browser (RUM)` heading:

```markdown
## Browser (RUM)

- `service.name`: `<service>-web`
- `deployment.environment.name`: from `<the env var, and where the backend reads the same value>`
- `service.version`: from `<the build's commit sha var>`
- Framework: <framework + version>
- Package: `@onepatch/rum` <version>
- Init: `<file:line>`

### Identity

| Attribute | Source |
|---|---|
| `user.email` | `src/auth/session.ts:41`, on auth state change |
| `org.id` | same |
| `plan` | same |

### Named actions

| Action | Fires when | Attributes | Source |
|---|---|---|---|
| `ran-workflow` | User clicks Run on a workflow | `workflowId` | `src/workflow/RunButton.tsx:88` |

### Automatic spans

`documentLoad`, `documentFetch`, `resourceFetch`, `click`, `HTTP GET`/`POST`, `webvitals`, `visibility`, plus JS errors.

### Trace propagation

| Origin | Joined | Why |
|---|---|---|
| same-origin | yes | automatic |
| `https://api.acme.com` | yes | `Access-Control-Allow-Headers` names `traceparent` |
| `https://legacy.acme.com` | no | wildcard `allow-headers` with credentials — would break cookie-bearing requests |

## Not captured

No session replay, no DOM, no keystrokes, no form contents. Console capture is off.

URLs are recorded as they are, query string and fragment included, because the query is usually where the URL says which thing. `scrubQueryStrings: true` drops both — set it if these URLs carry reset tokens, magic-link codes or email addresses rather than identifiers.
```

Rules: only list what the code actually emits — walk the real `recordAction` call sites, do not copy this template's examples. Include file:line for every hand-written row. Keep the "Not captured" section; it is the section a privacy reviewer opens first.

## 10. Close the loop

Tell the user:

> Your frontend now reports to `<slug>.logger.onepatch.dev` as `<app-name>`. Commit the `TELEMETRY.md` changes alongside the instrumentation so OnePatch picks up what your actions mean.
>
> Things to ask in your OnePatch workspace:
> - "what did <user email> do in their last session?"
> - "which pages threw errors in the last hour?"
> - "show me sessions where checkout failed"

If they have not connected GitHub, point them at the onboarding step — the context engine reads `TELEMETRY.md` from the repo, and you cannot drive that OAuth flow yourself. **If this frontend lives in its own repo, say so specifically:** connecting the backend repo is not enough, and a repo that isn't connected contributes nothing to what the agent knows.

## Don't

- **Don't add session replay.** Not with this package, not alongside it. If the user asks for a video, explain what the action list answers and let them decide; do not quietly install a recorder.
- **Don't put `connectTracesTo` entries in without probing.** Every other shortcut in this skill costs data. This one costs the customer's API calls.
- **Don't proxy the ingest token.** It is designed to be public.
- **Don't turn on `captureConsole` by default.** Console lines carry personal data more often than spans do.
- **Don't set `scrubQueryStrings: true` reflexively.** It reads as the safe choice and is usually the wrong one: it also drops the fragment, so a hash-routed app loses its route entirely and "which page was this?" stops having an answer. Set it when you have looked at the app's real URLs and they carry secrets, not identifiers. Either way, tell the user which way you set it and why.
- **Don't leave `appVersion` as a placeholder.** `"dev"` in production is worse than nothing: it looks answered.
- **Don't instrument a React Native app with this.** It needs browser APIs.
- **Don't leave `debug: true`** in the committed code.
- **Don't call `startRum` inside a React effect** without a module-scope guard — StrictMode runs it twice.
- **Don't report success without phase 8.** A green test suite and zero spans is the normal way this goes wrong.
