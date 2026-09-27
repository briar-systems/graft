# graft example

A small but real Laurel application, hosted in hedge through graft: a supervisor
and several workers, each serving on a thread of its own, all entering the one
application. It has a JSON route, a route with a typed path parameter, a route
that decodes the query string, a route that reads the raw request body, an HTML
form protected by a CSRF token, a session cookie, one middleware, and an
observer. Beside the application, hedge serves a static file and a health check
from its own configuration. Every one of those uses the real API of laurel,
hedge and graft. Nothing here is a mock.

## Run it

The example is its own Mach project with its own dependencies, so it does not
inherit the repository's `dep/`. It needs mach 6.3.0 or newer.

```sh
cd example
mach dep pull .
mach build . --profile release
LISTEN_ADDRESS=127.0.0.1:8080 mach run . --profile release -- hedge.toml
```

`mach dep pull` realizes the pins committed under `example/dep/` and copies
graft from this working tree. `mach run` forwards everything after `--` to the
program, which is how the example receives its configuration path, and a second
argument `--quiet` stops the per-request log. The first build takes a few
minutes and a few gigabytes of memory; after that only the last command is
needed.

The listen address comes from the environment, through
`address = "${ENV:LISTEN_ADDRESS}"` in `hedge.toml`, which `graft.host.run`
answers from the process environment. A platform that assigns only a port sets
`LISTEN_ADDRESS=0.0.0.0:$PORT`. The host block answers every name
(`server_name = "*"`), since a deployment's public name is not known here, so
the example runs unchanged behind a platform's router.

The server prints `graft: serving site on 4 workers` once every worker serves.
It stops on SIGINT or SIGTERM, drains the workers and the application, stops
the application, and prints `graft: stopped`.

## What each route proves

Run these against the running server. The responses are what this example actually
returned, not what it is supposed to return.

**A JSON route.** The handler owns its bytes and its media type; Laurel does not
serialize anything for you.

```sh
curl -s http://127.0.0.1:8080/api/hello
# {"message":"hello from laurel"}
```

**A typed path parameter.** `/api/echo/:id` declares a `u64` decoder. Decoding
happens during dispatch, before the handler runs, so the handler receives a
number and never sees the text.

```sh
curl -s http://127.0.0.1:8080/api/echo/42
# {"id":42}

curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/api/echo/abc
# 400
```

A value that does not fit a `u64` is refused the same way; the handler is never
entered.

**A session cookie.** The form page creates a session on first visit and counts
visits in it. The cookie is AEAD-protected and its contents are opaque to the
client.

```sh
curl -s -D headers.txt http://127.0.0.1:8080/form > page.html
grep -i '^set-cookie' headers.txt
# set-cookie: laurel_demo_session=TFMBAQ...; Path=/; Max-Age=3600; Secure; HttpOnly; SameSite=Lax

cookie=$(grep -i '^set-cookie:' headers.txt | sed 's/^[Ss]et-[Cc]ookie: //; s/;.*//')
curl -s -b "$cookie" http://127.0.0.1:8080/form | grep -o 'visits: [0-9]*'
# visits: 2
```

Laurel refuses a session cookie that is not `Secure` and `HttpOnly`, so the example
cookie carries both. Browsers treat `http://localhost` as a trustworthy origin
and will store it; `curl` will not send a `Secure` cookie over plain HTTP, which
is why the lines above pass the cookie back explicitly with `-b`.

**A form POST with CSRF.** The form page issues a token bound to the session.
The POST parses a bounded urlencoded body, looks the token up, and verifies it
against the session it was issued for.

```sh
token=$(grep -o 'name="csrf_token" value="[^"]*"' page.html | sed 's/.*value="//; s/"//')
curl -s -b "$cookie" -X POST -d "name=laurel&csrf_token=$token" \
    http://127.0.0.1:8080/form
# <!doctype html><title>laurel demo</title><p>accepted: laurel</p>
```

Every way of getting it wrong is refused with 403:

```sh
curl -s -o /dev/null -w '%{http_code}\n' -b "$cookie" -X POST \
    -d 'name=x' http://127.0.0.1:8080/form                      # 403, no token
curl -s -o /dev/null -w '%{http_code}\n' -b "$cookie" -X POST \
    -d 'name=x&csrf_token=AAAA' http://127.0.0.1:8080/form      # 403, bad token
curl -s -o /dev/null -w '%{http_code}\n' -X POST \
    -d "name=x&csrf_token=$token" http://127.0.0.1:8080/form    # 403, no session
```

**A query string.** `GET /api/search` decodes the query with `query.parse`, the
same urlencoded decoder a form body uses, into storage the handler owns. It reads
`q` with `form.find` and types `limit` with `form.typed` and the `u64` decoder
that types path parameters, so an absent `limit` takes the default and one that
is not a number is refused before anything is answered.

```sh
curl -s 'http://127.0.0.1:8080/api/search?q=hello+w%C3%B6rld%22%3C&limit=3'
# {"q":"hello wörld\"\u003c","limit":3}

curl -s 'http://127.0.0.1:8080/api/search?q=x'
# {"q":"x","limit":10}

curl -s -w ' %{http_code}\n' 'http://127.0.0.1:8080/api/search?q=x&limit=abc'
# invalid route parameter 400

curl -s -w ' %{http_code}\n' 'http://127.0.0.1:8080/api/search?q=a&&b'
# malformed query 400
```

**A raw body.** `POST /api/raw` reads the whole body with `raw.Body` as its
exact bytes, as a webhook handler does before it checks a signature, and answers
with them. The first read usually suspends, like the form's. The example's limit is
4096 bytes: a body of exactly that fits, and one byte more is refused with 413.

```sh
head -c 3000 /dev/urandom > in.bin
curl -s -X POST --data-binary @in.bin -o out.bin http://127.0.0.1:8080/api/raw
cmp in.bin out.bin && echo identical
# identical

head -c 4097 /dev/urandom | curl -s -w ' %{http_code}\n' -X POST --data-binary @- \
    http://127.0.0.1:8080/api/raw
# request body is too large 413
```

A chunked body with no declared length reads the same way.

**A static file and a health check.** Both are hedge's own services, `static`
and `fixed`, configured in `hedge.toml` and served by the workers on the same
listener without entering the application.

```sh
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/static/index.html
# 200

curl -s http://127.0.0.1:8080/healthz
# ok
```

**An unmatched path.**

```sh
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/nope
# 404
```

**The middleware and the observer** are visible in the responses and the log.
Every response carries the security headers the middleware applies after the
handler returns:

```sh
curl -s -D - -o /dev/null http://127.0.0.1:8080/api/hello | grep -i 'x-frame\|content-security'
# content-security-policy: default-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'
# x-frame-options: DENY
```

The observer prints one line per completed request, naming the worker thread
that served it, and, for a failed one, the private detail that never reaches the
client:

```
demo: api.hello -> 200 on worker thread 140055119134704
demo: request failed: kind 3 status 403: the form carried no csrf_token field
```

**Several workers.** Every request is served by one of the four workers, and the
kernel spreads new connections across them. 200 requests, each on a connection
of its own, landed on all four:

```sh
for i in $(seq 1 200); do curl -s -o /dev/null http://127.0.0.1:8080/api/hello; done
# in the server's log, counted by thread:
#   60 140055110746096
#   43 140055119134704
#   56 140055127523312
#   41 140055138009072
```

## How a Laurel application is hosted

**Laurel owns no listener, no socket, and no event loop in this build.** It
defines the application layer above `mach-http`: routing, middleware, sessions,
forms, errors, rendering, and lifecycle. It never names its host.

**Hedge is the server.** [Hedge](https://github.com/briar-systems/hedge) accepts
connections, speaks HTTP/1.1, HTTP/2 and HTTP/3, terminates TLS, serves files,
proxies, and dispatches matched routes into a *service*. An in-process
application is one service kind among several, and its configuration names it:

```toml
[service.app]
kind = "laurel"
application = "site"
```

`application = "site"` is a name, not a path. The executable registers the
assembled application under that name, and hedge resolves the name against its
registry when it builds each worker's services.

**graft binds the two.** [graft](../README.md) adapts laurel to hedge's host
contract (hedge's `doc/HOSTING.md`), and neither laurel nor hedge knows about
the other. `src/bin/main.mach` is the whole of what an embedder writes: an
`assemble` callback, a `release` callback, and one call to `host.run`, which

1. reads the configuration and seals it into a configuration generation
2. calls `assemble` with the message limits hedge commits every response under,
   so the application renders its errors within them
3. mounts the application under its name, with graft's request bridge as its
   handler and graft's lifecycle bridge as its lifecycle
4. composes the process with the worker count `server.workers` asks for, and
   hands it to hedge's supervisor, which starts the application before any
   worker binds a listener, polls its readiness, and serves until a signal
5. drains the workers and the application toward the same deadline, stops the
   application after the last worker has stopped, calls `release`, and answers
   the exit status

The supervisor thread takes the signals, reloads the configuration on SIGHUP and
drives certificate renewal. Each worker serves on a thread of its own, with its
own listeners, io runtime, timers and buffer pool. The static files and
`/healthz` are hedge's own `static` and `fixed` services, served by the workers
without entering the application. A reload rebuilds hedge's services from the
new configuration and resolves `application = "site"` against the same registry
again, so the running application serves the new generation unchanged and is
never restarted.

## The threading contract

Every worker is handed the same registry, so **one application instance serves
every worker at once**. Its handlers, middleware and callbacks run on several
threads concurrently. This is what that means for each part of it.

**Per-request state belongs to one worker.** graft keeps the request's context,
recorder, execution cursor and outcome in the request arena of the worker
serving it, and dispatch writes its route captures there too. A request is
entered, suspended and resumed only by the worker that admitted it. Nothing per
request is shared, and a handler's `slot` and the memory it takes from
`request_context.alloc` are private to its request.

**What Laurel shares is safe to share.** Everything the `app.App` holds is either
fixed at assembly or synchronized:

- admission (`max_active_requests`) is one atomic counter
- the router, the middleware stack, the error mapper, providers, the security
  headers and origin policies, the vocabulary and the CSRF protector are
  written only when they are initialized or released, and are read-only while
  requests run. Dispatch writes only its own locals and the request's captures.
- the session manager's state is atomic, and the in-memory session store, the
  nonce and replay guards, the session and CSRF key rings, and the CSRF ring
  registry each take their own mutex

**The lifecycle runs on the supervisor's thread.** `app.start`,
`app.poll_ready`, `app.drain` and `app.stop` are hedge's supervisor steps, and
`assemble` and `release` run on the thread that called `host.run`, before the
first worker starts and after the last one has stopped. Never call them from a
handler.

**The application's own state is the application's to synchronize.** Laurel
calls whatever the application hands it (handlers, middleware, the observer,
entropy sources, a session store or codec, an authenticator, a renderer,
providers) from every worker at once, with the same `ctx` pointer and the same
`app_state`. Anything those reach and change must be atomic or locked. The
example's request and observer counts are `std.sync.atomic` counters for exactly
this reason, and a store or cache of your own needs a lock of its own. A value
written only in `assemble` and never after needs nothing.

### What graft does for the application

graft runs Laurel's router itself, so routes with typed and wildcard parameters
dispatch exactly as they would under any other host. It enters the application
at the request headers, before any body byte has been read, which is why
`POST /form` is written as a handler that can suspend: the form parser polls the
body, a pending read leaves the chain with its token (`handler.suspend`), and
graft parks the call until more body arrives, then resumes only that step.
Nothing that already ran runs again. An exchange that dies while a step is
suspended is abandoned, and every middleware exit half that is owed still runs.
The request identifier laurel reports is hedge's trace id, so both logs name
the same request.

The request state lives in the request arena, whose bound the application
declares: `host.Application.memory_bytes`, 64 KiB here. Hedge claims that
memory in chunks as the request needs it and returns it when the exchange
settles, so an idle connection holds none of it.

Connections are not pooled up front. Hedge grows its connection storage with
what is actually connected, and `server.limits.max_connections` is an optional
policy cap rather than a storage size, so this example leaves it unset. The one
ceiling the application owns is `app.Limits.max_active_requests`, past which
Laurel refuses admission.

## Layout

| path | what it is |
|---|---|
| `src/bin/main.mach` | the executable: assemble, release, and `host.run` |
| `src/app.mach` | the application: routes, middleware, sessions, CSRF, observer |
| `src/handlers.mach` | the six route handlers |
| `hedge.toml` | workers, listener, host, services, routes |
| `public/static/` | the static file |

## Versions

The example builds against graft's working tree through `path = "../"`, so it
shows whether the graft in this checkout still hosts laurel under hedge. It
selects the exact laurel, mach-http and std releases graft pins (`=0.21.0`,
`=0.24.0`, `=9.2.0`), and hedge 0.14.0 comes through graft.
