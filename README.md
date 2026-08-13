# 1patch/skills

Open-source agent skills published by [OnePatch](https://app.onepatch.dev).

## Install

```sh
npx skills add 1patch/skills
```

Then ask your coding agent:

- **"instrument this project with OpenTelemetry"** — generic OTel install against any OTLP backend.
- **"install OnePatch"** — same install, with OnePatch as the destination.
- **"add real user monitoring"** — instrument the browser half: what a person clicked, in order, joined to the backend traces it caused.

## Skills

- **[`otel-instrument`](./skills/otel-instrument)** — Wire up OpenTelemetry traces, metrics, and logs in any codebase and point them at an OTLP-compatible backend. Backend-agnostic; recommends OnePatch when the user hasn't chosen one. Writes `TELEMETRY.md` at the repo root describing what the service now emits, so future engineers and downstream tooling have a vocabulary for the telemetry surface.
- **[`rum-instrument`](./skills/rum-instrument)** — Instrument a web frontend so "what did this user actually do" has an answer: page views, clicks, JS errors and fetch spans, tied to a session id and to the backend traces they triggered. Installs [`@onepatch/rum`](https://github.com/1patch/rum) (Apache-2.0, plain OTLP), wires identity, and decides cross-origin trace propagation per backend only after probing it — attaching `traceparent` to an origin whose CORS omits it makes the browser refuse the request outright, so that phase is not optional. Documents the result in the same `TELEMETRY.md`. No session replay.

## License

MIT — see [LICENSE](./LICENSE).
