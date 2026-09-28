# Changelog

## [0.3.0] - 2026-09-28

### Dependencies

- **Breaking.** hedge `^0.17` (v0.17.0), laurel `^0.23` (v0.23.0), mach-http `^0.25` (v0.25.1) and mach-std `^9.4` (v9.4.1), committed as gitlinks with crypto v0.26.0, tls v0.16.0, quic v0.25.1 and acme v0.14.0. The example follows. Resolution is flat, so an application on graft moves to these with it. hedge 0.17 declares ACME certificates only as `[acme.certificate.<id>]` tables and refuses a route path without its leading `/`. laurel 0.23's decoders answer `invalid parameter`. std 9.4.1 maps a guard page below every linux thread stack.
- **Breaking.** Requires mach 6.5 (`mach = "^6.5"`), since mach-http 0.25.1 does, and CI seeds mach v6.5.0.

## [0.2.0] - 2026-09-28

graft bridges every laurel provider over hedge's host contract 1.7: tasks, secrets, configuration and telemetry (#4).

### Added

- The task bridge, `graft.bridge.task` (#4, #13). A hosted application's laurel tasks run on the task thread hedge's supervisor owns (`hedge.task`), one hedge task per laurel task. `host.Process` makes a hedge task facility, or adapts the program's when `Options.process.tasks` is set, and hands it to hedge through `composition.Options.tasks`. The lifecycle bridge drains the application's tasks beside `app.drain` toward the supervisor's deadline and stops them after the application. From the first drain the provider refuses registrations, triggers and runs that would begin, waits for admitted runs and cuts one off at the deadline, where it counts as abandoned.
- The secret bridge, `graft.bridge.secret` (#4, #14). `laurel.secret.Source` over `hedge.secret.source` and `secret.borrow`, so a secret stays secret-typed from hedge's store to the application's use and never passes through graft.
- The configuration bridge, `graft.bridge.settings` (#4, #14). `laurel.config.Provider` over `hedge.settings`, read through a view the lifecycle's start step takes and hedge's optional `reload` step retakes once a reload publishes. `READ_SECRET` and `READ_MISMATCH` read as `READ_INVALID`, and reads before the start step are invalid.
- The telemetry bridge, `graft.bridge.telemetry` (#4, #14). `laurel.observability.Telemetry` over `hedge.observe`: a counter adds to and a gauge sets `app_<name>` with its labels sorted, and an event is a log record. The request bridge writes each request's span as a record under hedge's active telemetry, so it carries hedge's trace id with hedge's span as its parent.
- `graft.bridge.providers` holds the four bridges. Its `entry` is `Setup.providers`, the one entry an application lists in its `provider.Set`, and it answers laurel's four provider keys. `mount` binds them to the application it registers, and `Process.finish` closes them (#14).

### Dependencies

- hedge `^0.16` (v0.16.0), laurel `^0.22` (v0.22.0), mach-http `^0.25` (v0.25.0) and mach-std `^9.3` (v9.3.0), committed as gitlinks with crypto v0.25.0, tls v0.15.0, quic v0.24.0 and acme v0.13.0 (#15). The example selects the same releases by caret range, where it pinned 0.1.0's exactly. hedge 0.16 refuses a key file group or other can read.

### Known gaps

- Health checks are not bridged: laurel has no health surface beyond its lifecycle readiness, which already reaches hedge's readiness check for the application (#14).
- laurel's WebSocket upgrade is accepted under graft, but graft hands hedge no tunnel owner, so nothing commits the 101 and an upgrade does not complete.

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
