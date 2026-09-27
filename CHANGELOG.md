# Changelog

## [0.1.0] - 2026-09-27

The first release. graft hosts laurel applications in-process in hedge, as the reference binding on hedge's public host contract. It replaces the laurel adapter hedge shipped until 0.14.0.

### Added

- `graft.host` (#1). `host.run(path, name, application)` is the whole program around one application: it loads and seals hedge's configuration, assembles the application against the limits hedge commits responses under, mounts it under `name`, composes the process, supervises it with reloads until SIGINT or SIGTERM drains it, releases it, and answers the exit status (`EXIT_OK`, `EXIT_RUNTIME` 70, `EXIT_DRAIN_INCOMPLETE` 75, `EXIT_CONFIG` 78). `host.run_with` takes `host.Options` for the resolver, the capabilities, hedge's own process options and the log label. `host.Process` is the same sequence in steps (`start`, `serve`, `request_stop`, `request_reload`, `finish`), and `host.mount(registry, name, application, memory_bytes)` registers an assembled application into a registry a program already composes with. An application is a `host.Application` of an `assemble` and a `release` callback, a context and its request memory.
- The request bridge, `graft.bridge.request` (#1). Each request routed to the application is dispatched by laurel's router and run through its middleware, with hedge's trace id as laurel's request id. A handler waiting on body parks the call on `WAIT_BODY` under laurel's narrowed deadline and is resumed, never restarted, and a handler past its deadline is timed out through the exchange's scope. The finalizer hands the request back to laurel however the exchange settled, running any middleware exit owed.
- The lifecycle bridge, `graft.bridge.lifecycle` (#1). hedge's `service.Lifecycle` steps over laurel's `app.start`, `app.poll_ready`, `app.drain(deadline)` and `app.stop`.
- `example/`, laurel's hedge-hosted demo moved here, with its `hedge.toml` (#1). Its `main` is two callbacks and one `host.run` call.
- graft imports only the items hedge's `doc/HOSTING.md` names as the host contract, version 1.1. Configuration, secrets, telemetry and bound endpoints for hosted code wait on hedge#420, hedge#421, hedge#422 and hedge#423, and background tasks on graft#4.

### Dependencies

- hedge `^0.14` (v0.14.0), laurel `^0.21` (v0.21.0), mach-http `^0.24` (v0.24.0) and mach-std `^9.2` (v9.2.0), committed as gitlinks with crypto v0.24.1, tls v0.14.0, quic v0.23.0 and acme v0.12.0 (#5). The example pins the same releases exactly. hedge 0.14 routes a hosted application under `kind = "application"`, which the example uses.

### Known gaps

- laurel 0.21's WebSocket upgrade is accepted under graft, but nothing commits the 101, so an upgrade does not complete (hedge#427).
