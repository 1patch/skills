---
name: rum-instrument
description: Instrument a web frontend so you can answer "what did this user actually do" — page views, clicks, JS errors and fetch spans, tied to a session id and to the backend traces they triggered. Trigger phrases include "add RUM", "set up real user monitoring", "instrument the frontend", "add browser monitoring", "track what users are doing", "why did this user see an error", "add session tracking", "instrument my React/Next/Vue app", or any variant naming OnePatch as the destination ("install OnePatch in my frontend", "add OnePatch RUM"). Installs `@onepatch/rum`, wires identity, decides trace propagation per backend after probing it, writes the tests, and documents the result in the repo's `TELEMETRY.md`. For backend/server instrumentation use `otel-instrument` instead.
---

# Instrument a frontend with `@onepatch/rum`

Output: pages, clicks, failed requests, JS errors — in order, per session, per user, joined to backend traces. An action list, not a video: no DOM, no session replay, no keystrokes. The package wraps the OTel browser SDKs and speaks OTLP; backend work is `otel-instrument`, same ingest endpoint.

Phases in order. The closing checklist gates "done".

## 1. Discover

| Signal | Framework | Entry |
| --- | --- | --- |
| `next.config.*` | Next.js | `instrumentation-client.ts` (15+) or a client component |
| `vite.config.*` + `react` | Vite React | `src/main.tsx` |
| `react-scripts` | CRA | `src/index.tsx` |
| `nuxt.config.*` | Nuxt | `plugins/onepatch-rum.client.ts` |
| `svelte.config.*` | SvelteKit | `src/hooks.client.ts` |
| `vue` / `angular.json` | Vue / Angular | `src/main.ts` |
| plain `<script>` | none | prebuilt bundle (phase 3) |

`expo` / `react-native` → not this skill: `@onepatch/rum` wraps the browser SDK and needs `document` (React Native defines `window`, so `startRum` throws instead of no-opping). The app is still instrumentable — wire the OTel JS SDK directly per `otel-instrument` § Expo / React Native, and `otel-instrument` for their backend.

Monorepo: instrument one frontend; say which before writing anything.

Locate the auth/session source now (session hook, auth context, `me` query) — phase 3 requires it.

## 2. Ingest configuration

Need `ingestUrl` shaped `https://<slug>.logger.onepatch.dev` and an `op_…` `ingestToken`. Sources, in order:

1. User pasted the onboarding payload — extract `https://[a-z0-9-]+\.logger\.onepatch\.dev` and `op_[A-Za-z0-9_-]+`.
2. Repo already has them — `otel-instrument`'s bootstrap default or `OTEL_EXPORTER_OTLP_ENDPOINT`, token nearby. Reuse.
3. Neither — ask the user to sign up at `app.onepatch.dev` and paste the payload. Never guess a URL or use a placeholder.

`op_…` is a write-only, append-only, single-tenant bearer (Sentry-DSN class): commit it, inline it, `NEXT_PUBLIC_*` / `VITE_*`. Never proxy it. Refuse any non-`op_`-shaped key for the browser.

## 3. Install and initialise

```sh
bun add @onepatch/rum   # or npm / pnpm / yarn — 0.2.0 or newer
```

Call `startRum` **once** per page load, **client-side** (it no-ops off-browser), **early** — before the app's own fetches.

- `user` — required; identity resolver (phase 4).
- `appName` — `<service>-web`; becomes `service.name`. Backend `acme-api` ⇒ `acme-web`.
- `environment` — read from wherever the backend reads its own; never hand-typed.
- `appVersion` — the commit sha (`NEXT_PUBLIC_VERCEL_GIT_COMMIT_SHA`, `VITE_COMMIT_SHA` ← `$GITHUB_SHA`, `git rev-parse --short HEAD`); becomes `service.version`. None available → wire one in now.

```ts
// Next.js 15+: instrumentation-client.ts at the project root.
// Other frameworks: top of the entry module.
import { startRum } from "@onepatch/rum";

startRum({
  ingestUrl: process.env.NEXT_PUBLIC_ONEPATCH_INGEST_URL!,
  ingestToken: process.env.NEXT_PUBLIC_ONEPATCH_INGEST_TOKEN!,
  appName: "<service>-web",
  environment: process.env.NEXT_PUBLIC_APP_ENV!,
  appVersion: process.env.NEXT_PUBLIC_COMMIT_SHA!,
  user: async () => (await getSession())?.user ?? null, // phase 4
  connectTracesTo: [/* phase 5 decides — leave out until then */],
});
```

Next.js 13–14: client component in the root layout, `startRum` at module scope behind a `let booted` guard (StrictMode double-runs effects in dev).

No bundler: `https://unpkg.com/@onepatch/rum/dist/onepatch-rum.min.js` in `<head>` before app scripts; `OnePatchRum.startRum({…})`.

## 4. Identity

Wire from where the app already knows who is signed in — grep `analytics.identify(`, `posthog.identify(`, `Sentry.setUser(`, `datadogRum.setUser(`, or the auth provider's `onAuthStateChange` / `useUser` / session context. Shapes:

```ts
user: { id: session.userId, email: session.email }  // known synchronously
user: async () => (await me())?.user ?? null        // resolver, awaited once
user: "anonymous"                                   // only if the app has no accounts at all
```

A resolver returning `null` is fine (login page, cold load); `(await startRum(...)).identified` reports which happened. The first span batch waits up to 3s for the resolver so page-load spans carry the user — don't work around that delay.

Pass names, not only ids: `id`, `email`, `name`, `orgId`, `orgName` map to `user.*` / `org.*`; other keys pass through as written.

`identifyUser` on every later change — sign-in, workspace switch, sign-out — on the same path the app tells its other analytics, mounted **above** any auth gate:

```ts
identifyUser({ id: user.id, orgId: user.orgId, plan: user.plan });
identifyUser({ id: null, email: null, orgId: null });  // sign-out
```

Explicit `null` clears an attribute; an omitted key stays stamped on later spans.

Never record secrets or PII beyond identity: no tokens, full addresses, or user-typed free text in attributes.

## 5. `connectTracesTo` — probe before you list

Same-origin joins automatically: leave the option out entirely.

Cross-origin: `traceparent` forces a preflight, and a backend whose `Access-Control-Allow-Headers` misses it gets its requests **refused**, not merely untraced. The library probes each listed origin at startup and connects only passes, but probe from the terminal too, against a route the frontend really calls (`/` usually 404s ahead of CORS and proves nothing):

```sh
curl -s -D - -o /dev/null -X OPTIONS "https://api.acme.com/v1/things" \
  -H "Origin: https://app.acme.com" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: traceparent" \
  | grep -i "^access-control-"
```

| Response | Verdict |
| --- | --- |
| `allow-headers` names `traceparent` | Safe — list it. |
| `allow-headers: *`, `allow-origin: *` | Safe — wildcard origin can't receive credentialed requests, so none can break. |
| `allow-headers: *`, `allow-credentials: true`, `allow-origin` echoed | **Not safe** — wildcard is illegal for credentialed requests; cookie-bearing calls with `traceparent` get blocked. Ask for `traceparent` named explicitly. |
| `allow-headers` omits `traceparent` | Not safe — offer a PR widening that backend's CORS. |
| no `access-control-*` at all | Inconclusive — you probed a route that 404s before CORS. Find a real one. |

List only passing origins. No wildcards (the library rejects them; they'd header every third-party request). Tell the user, one line per origin: joined or not, why; offer the CORS one-liner for the ones that aren't.

## 6. Named actions

Clicks and navigations are automatic (element + xpath). Add what the click meant:

```ts
recordAction("ran-workflow", { workflowId });
catch (error) { recordError(error, { where: "checkout" }); }
```

3–6 for a first pass: mutations, submit handlers, whatever the product's analytics already tracks. Wrap handled errors that would otherwise vanish. Don't wrap every button.

## 7. Tests

Use the repo's existing runner. Export the options as one object (`src/rum-config.ts`) — the only structural change. Four tests:

**a.** Starts, and every listed backend connects:

```ts
import { startRum } from "@onepatch/rum";
import { rumOptions } from "../src/rum-config";

test("RUM starts and connects every backend it was told to", async () => {
  const status = await startRum(rumOptions);
  expect(status.started).toBe(true);
  expect(status.error).toBeUndefined();
  for (const backend of status.backends) {
    expect(backend.allowed, `${backend.origin}: ${backend.detail}`).toBe(true);
  }
});
```

Reaches the network. Offline-only suites: assert on the options, move the backend loop to a pre-deploy check, say which you did.

**b.** `status.identified === true` for a signed-in session. Mock the package: `identifyUser` fires on the auth path with the wired fields; sign-out passes explicit `null`s, not omitted keys.

**c.** Init module imported exactly once, on a client-only path. Grep-shaped is fine.

**d.** No wildcard in `connectTracesTo`.

Run them; report the real result. 7a failing on an unconnected backend is the finding, not a test to relax.

## 8. Verify in a real browser

Boot, sign in, click, navigate once. Session id via `OnePatchRum.sessionId()` (script bundle) or temporary `debug: true`. Have the user check their workspace: `<service>-web` with `click` and `documentLoad` spans within ~10s. Confirm environment and version are non-blank.

Read `user.id` off a `documentLoad` from the very first page. Whole session anonymous → the resolver returned `null` (session request not in flight at boot, or provider not mounted). Names on later spans only → something calls `identifyUser` instead of passing `user`.

Nothing arrives: CORS error on the ingest URL = wrong host; 401 = wrong token; zero requests to it = `startRum` never ran. A `[onepatch/rum]` error names its own fix.

Trace-joined backend: trigger a request, confirm the FE and BE spans share a trace id.

## 9. `TELEMETRY.md`

One telemetry doc per repo, at the root — written by `otel-instrument`, kept fresh by the `telemetry-docs-freshness` monitor. Append under `## Browser (RUM)`; never start a second document.

Frontend in its own repo: a browser-only `TELEMETRY.md` is complete — no placeholder server sections. Its propagation table is then the only written FE↔BE join, so name the backend *services* (and repos), not just origins.

```markdown
## Browser (RUM)

- `service.name`: `<service>-web`
- `deployment.environment.name`: from `<env var, and where the backend reads the same value>`
- `service.version`: from `<the build's commit sha var>`
- Framework: <framework + version> · Package: `@onepatch/rum` <version> · Init: `<file:line>`

### Identity

| Attribute | Source |
|---|---|
| `user.id`, `user.email` | `user` resolver at init, `src/rum-config.ts:12` |
| `org.id`, `org.name` | `identifyUser` on workspace switch, `src/auth/session.ts:41` |

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
| `https://api.acme.com` | yes | `allow-headers` names `traceparent` — joins `service.name = acme-api` (repo `acme/backend`) |
| `https://legacy.acme.com` | no | wildcard `allow-headers` with credentials — would break cookie-bearing requests |

## Not captured

No session replay, no DOM, no keystrokes, no form contents. Console capture is off.

URLs are recorded with query string and fragment. `scrubQueryStrings: true` drops both — set it if these URLs carry reset tokens, magic-link codes or email addresses rather than identifiers.
```

List only what the code emits — walk the real `recordAction` sites. `file:line` on every hand-written row. Keep "Not captured".

## 10. Close the loop

> Your frontend now reports to `<slug>.logger.onepatch.dev` as `<app-name>`. Commit the `TELEMETRY.md` changes alongside the instrumentation so OnePatch picks up what your actions mean.
>
> Things to ask in your OnePatch workspace:
> - "what did <user email> do in their last session?"
> - "which pages threw errors in the last hour?"
> - "show me sessions where checkout failed"

GitHub not connected → point at the onboarding step; you can't drive that OAuth flow. Frontend in its own repo → say explicitly that connecting the backend repo is not enough.

## Before you report done — the checklist

Each item: a challenge, a verification, the evidence that passes. An item you can't close is a finding to report, not a row to skip.

| # | Challenge | Verify with | Passes when |
|---|---|---|---|
| 1 | Ingest accepts the token | `curl -s -o /dev/null -w '%{http_code}' -X POST "<ingestUrl>/v1/traces" -H "Authorization: Bearer <op_…>" -H "content-type: application/json" -d '{"resourceSpans":[]}'` | `2xx`. `401` = token, `404`/DNS = host. Fix before anything else. |
| 2 | The scrub decision saw the real auth routes | Grep the router for reset / OAuth / SSO / magic-link callbacks: `oobCode`, `?code=`, `token=`, `state=` | Every hit scrubbed (`scrubQueryStrings` or `ignoreUrls`) or provably credential-free; `TELEMETRY.md` names the routes you read. |
| 3 | Env labels are the backend's own strings | Read the literal `deployment.environment` values the backend emits | Every deploy target emits an exact member of that set — never a label the backend doesn't emit, or emits for a different cluster. |
| 4 | Non-prod traffic has a deliberate destination | Trace where a localhost or preview session's spans go | Init is prod-gated, or the user agreed dev sessions ship, distinctly labelled. |
| 5 | Identity survives sign-out and user switch | Exercise logout (and workspace switch) in the running app | Explicit `null`s flow through `identifyUser`; no storage key carries the previous user's id or org into the next session. |
| 6 | Tests pass under the repo's own runner | The test command already in `package.json` | Green output, runner named. |
| 7 | One span carries who, where, which build | Phase 8, real browser | `user.id`, environment, `service.version` all non-blank on a first-page `documentLoad`. |
| 8 | The trace join is proven, not configured | One API call per origin in `connectTracesTo` | FE and BE spans share a trace id. |
| 9 | The user sees it in their workspace | Ask them | They confirm `<service>-web` with spans — where *their* queries read, not just an export that returned 200. |
| 10 | No repo-wide side effects | `git diff` the whole branch | Workarounds (engine checks, pins, config files) scoped to this change, not blanket switches. |

## Don't

- **No session replay** — not with this package, not alongside it. If the user wants video, explain what the action list answers and let them decide.
- **No unprobed `connectTracesTo` entries** — an origin whose CORS preflight rejects `traceparent` makes the browser block the application's own requests to it.
- **Never proxy the ingest token** — it is designed to be public.
- **No `captureConsole` by default** — console lines carry personal data more often than spans.
- **No reflexive `scrubQueryStrings: true`** — it also drops the fragment, so a hash-routed app loses its route. Set it after reading the app's real URLs, when they carry secrets rather than identifiers — and say which way you set it.
- **No `user: "anonymous"` to get past a type error** — use the async resolver when the session is not available synchronously; `"anonymous"` is only for apps with no accounts.
- **No placeholder `appVersion`** — a constant like `"dev"` gives every deploy the same `service.version`, so error-rate changes can't be attributed to a release.
- **No `@onepatch/rum` in React Native** (`otel-instrument` § Expo / React Native wires it instead), no committed `debug: true`, no `startRum` in a React effect without a module-scope guard.
- **No success report without phase 8 and the checklist** — a passing test suite does not prove spans are exported; only observing them does.
