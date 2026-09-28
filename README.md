# graft

graft hosts [laurel](https://github.com/briar-systems/laurel) applications
in-process in [hedge](https://github.com/briar-systems/hedge), as the reference
binding on hedge's public host contract
([laurel#183](https://github.com/briar-systems/laurel/issues/183)).

## Use

```mach
use graft.host;

# assemble the laurel application against what hedge commits responses under,
# and release it once hedge has drained and stopped it
fun assemble(ctx: ptr, setup: *host.Setup) *app.App { ... }
fun release(ctx: ptr, assembled: *app.App) bool { ... }

#[symbol("main")]
fun main(argc: i64, argv: **u8) i64 {
    ret host.run("hedge.toml", "site", host.Application{ctx: nil,
        assemble: assemble, release: release, memory_bytes: 65536});
}
```

- `host.run(path, name, application)` is the whole program: it loads and seals
  the configuration, assembles the application, mounts it under `name`,
  composes the process, supervises it until SIGINT or SIGTERM drains it, and
  answers the exit status (`EXIT_OK`, `EXIT_CONFIG` 78, `EXIT_RUNTIME` 70,
  `EXIT_DRAIN_INCOMPLETE` 75). `host.run_with` takes `host.Options` for the
  resolver, the capabilities, hedge's own process options and the log label.
- `host.Process` is the same sequence in steps (`start`, `serve`,
  `request_stop`, `request_reload`, `finish`), for a program that drives the
  supervisor itself.
- `host.mount(registry, name, application, memory_bytes)` registers an assembled
  application into a registry a program already composes with.
- `graft.bridge.request` and `graft.bridge.lifecycle` are the request bridge and
  the lifecycle bridge `mount` registers. The lifecycle bridge drains the task
  provider the application's provider set resolves beside the application,
  toward the same deadline, and stops it after the application. Its `reload`
  step, which hedge calls once a reload has published, moves the application's
  settings to the new generation.
- `Setup.providers` is the provider entry graft supplies, for the application
  to list in its `provider.Set`. It answers laurel's provider keys with
  graft's bridges over hedge's facilities for hosted code:
  - `laurel.task` with `graft.bridge.task`, so the application's tasks run on
    the task thread hedge's supervisor owns. graft makes that facility, or
    adapts the one a program passes in `Options.process.tasks`.
  - `laurel.secret` with `graft.bridge.secret`, which borrows the secrets the
    configuration's `[application.<name>.secrets]` grants the application.
    Each value stays secret-typed from hedge's store to the application's use.
  - `laurel.config` with `graft.bridge.settings`, which reads the
    application's `[application.<name>.settings]`. hedge's read statuses 1 to 4
    are laurel's, and a secret reference or a kind mismatch reads as
    `READ_INVALID`.
  - `laurel.telemetry` with `graft.bridge.telemetry`. A counter or a gauge is
    the series `app_<name>` labelled with the application, and an event is a
    log record attributed to it. The request bridge also writes each request's
    span as a record under hedge's trace for it, with hedge's span as its
    `parent_id`.

  `mount` binds these to the application it registers.
  A program that calls `mount` itself makes a `graft.bridge.providers.Providers`
  over the task facility it hands hedge and lists its `entry` in the
  application's provider set.

graft imports only the items hedge's `doc/HOSTING.md` names as the host
contract (version 1.7). [`example/`](example) is a complete application served
this way.

## Build

```sh
mach dep pull .
mach build .
mach test . --lib tests --timeout 5m
```

The tests run a real process over a local socket, on Linux.

## Workflow

`dev` is the default branch. Work branches from it as `feat/<issue>` or
`fix/<issue>` and merges back through a pull request. `main` only takes release
merges from `dev`. A `hotfix/<issue>` branches from `main` and merges into both.

Both branches require a pull request and a passing `gate` check. Neither can be
deleted or force-pushed, and pull requests merge with a merge commit. Repository
admins can bypass these rules to cut a release. Once a `v*` tag is pushed, only
an admin can move or delete it.

Commits follow [Conventional Commits](https://www.conventionalcommits.org), with
the issue number as the scope: `fix(#12): reject a negative length`.

Issues are labeled on independent axes:

| axis | labels |
| --- | --- |
| semver magnitude | `patch`, `minor`, `major` |
| kind of work | `feature`, `fix`, `removal`, `chore`, `performance` |
| where, omitted for core code | `testing`, `tooling`, `doc` |
| severity and state | `critical`, `blocked`, `security` |
| discussion | `discussion` |

## CI

`.github/workflows/ci.yml` runs the org's shared `mach-lib.yml` on pull
requests. A pull request into `dev` builds and tests on `x86_64-linux`, checks
formatting, builds the example and cross-builds every manifest target in
release. A pull request into `main` also runs `aarch64-linux`,
`x86_64-windows`, `aarch64-darwin` and `x86_64-darwin`, and an `aarch64-linux`
runner without FEAT_DIT runs the tests under `qemu-aarch64 -cpu max`. To run
every leg on any branch, use `gh workflow run CI --ref <branch> -f heavy=all`.

The last job, `gate`, is the check the branch rules require. It fails if any
other job failed.

The compiler version is `mach-version` in `ci.yml`. Change it together with the
`mach` range in `mach.toml`.

## Releases

1. Set `version` in `mach.toml`, add its `## [X.Y.Z]` section to `CHANGELOG.md`
   and merge that into `dev`.
2. Merge `dev` into `main` through a pull request, which runs every leg.
3. Tag `main` and push the tag: `git tag -a vX.Y.Z -m "graft X.Y.Z" && git push origin vX.Y.Z`.

`.github/workflows/cd.yml` calls the org's shared `mach-release.yml`, which
checks that the tag matches the manifest version and has a CHANGELOG section,
runs every CI leg, and publishes a GitHub release with that section as its
notes. A tag with a prerelease part, such as `v1.0.0-rc.1`, is published as a
prerelease. A dispatch of `Release` rehearses the same path under a throwaway
tag.

## License

MIT. See [LICENSE](LICENSE).
