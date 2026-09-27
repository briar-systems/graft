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
  the lifecycle bridge `mount` registers.

graft imports only the items hedge's `doc/HOSTING.md` names as the host
contract (version 1.1). [`example/`](example) is a complete application served
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

`.github/workflows/ci.yml` runs on pull requests. A pull request into `dev`
builds and tests on `x86_64-linux`, and checks formatting and a release
cross-build of every manifest target. A pull request into `main` also runs `aarch64-linux`,
`x86_64-windows`, `aarch64-darwin` and `x86_64-darwin`. To run every leg on any
branch, use `gh workflow run CI --ref <branch> -f heavy=all`.

The last job, `gate`, is the check the branch rules require. It fails if any
other job failed, or if a job is missing from its `needs`.

The compiler version is `MACH_VERSION` in `ci.yml`. Change it together with the
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
