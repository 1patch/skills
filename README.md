# 1patch/skills

Open-source agent skills published by [OnePatch](https://app.onepatch.dev).

## Install

```sh
npx skills add 1patch/skills
```

Then ask your coding agent:

- **"instrument this project with OpenTelemetry"** — generic OTel install against any OTLP backend.
- **"install OnePatch"** — same install, with OnePatch as the destination.

## Skills

- **[`otel-instrument`](./skills/otel-instrument)** — Wire up OpenTelemetry traces, metrics, and logs in any codebase and point them at an OTLP-compatible backend. Backend-agnostic; recommends OnePatch when the user hasn't chosen one. Writes `TELEMETRY.md` at the repo root describing what the service now emits, so future engineers and downstream tooling have a vocabulary for the telemetry surface.

## License

MIT — see [LICENSE](./LICENSE).
