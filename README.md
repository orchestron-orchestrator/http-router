# http_router

`http_router` is a small HTTP router for Acton.

It is built around a simple idea: register handlers directly on a `Router`, and use `route(prefix, install)` to compose reusable route bundles under a shared prefix.

## Quick start

The router only handles routing. It works with the existing `http.Request` from Acton's `http` module and the same `respond(status, headers, body)` callback you would use without a router.

```acton
import http
import http_router
import logging
import net


def health(ctx: http_router.Context, respond: proc(int, dict[str, str], str) -> None):
    respond(200, {"Content-Type": "text/plain"}, "ok")


def item(ctx: http_router.Context, respond: proc(int, dict[str, str], str) -> None):
    respond(200, {"Content-Type": "text/plain"}, "item:{ctx.param("id")}")


actor main(env):
    logh = logging.Handler("HTTP")
    log = logging.Logger(logh)
    tcpl_cap = net.TCPListenCap(net.TCPCap(net.NetCap(env.cap)))

    r = http_router.Router()
    r.use(http_router.access_log_middleware(log))
    r.use(http_router.recoverer_middleware(log=log))
    r.get("/health", health)
    r.get("/items/{{id}}", item)

    def on_request(server, request, respond):
        r.serve(request, respond)

    def on_error(server, error):
        print("Error: {error}")

    def on_accept(server):
        server.cb_install(on_request, on_error)

    server = http.Listener(tcpl_cap, "::", 8080, on_accept, log_handler=logh)
```

Register routes with `get`, `post`, `put`, `patch`, `delete`, `options`, or the generic `handle(...)`. Use `route(prefix, install)` when you want to compose route bundles under a shared prefix.

Handlers receive a `Context` plus a `respond(...)` callback. The context gives you:

- `ctx.param("id")` for path params
- `ctx.query_param("q")` for query params
- `ctx.set(...)` and `ctx.get(...)` for request-scoped values shared through middleware

## Route pattern examples

Conceptually, route params are names enclosed in curly braces, like `{id}`. In Acton string literals, write them as `{{id}}` or `r"{id}"`.

| Pattern | Meaning | Example match |
| --- | --- | --- |
| `/health` | Exact path match | `GET /health` |
| `/items/{id}` | Capture one segment as `id` | `GET /items/42` |
| `/teams/{team}/devices/{device}` | Capture multiple named segments | `GET /teams/core/devices/r1` |
| `/files/*tail` | Capture the rest of the path as `tail` | `GET /files/docs/setup.txt` |

Examples in code:

```acton
r.get("/health", health_handler)
r.get("/items/{{id}}", item_handler)
r.get("/teams/{{team}}/devices/{{device}}", device_handler)
r.get("/files/*tail", file_handler)
```

Inside handlers:

```acton
ctx.param("id")      # "42"
ctx.param("team")    # "core"
ctx.param("device")  # "r1"
ctx.param("tail")    # "docs/setup.txt"
```

Wildcard segments must be last. `/files/*tail/meta` is invalid.

## Matching rules

Routing happens in two steps:

1. The HTTP method must match.
2. Among routes for that method, the most specific path wins.

Specificity is:

1. Static segment
2. Param segment
3. Wildcard segment

That means:

- `/items/special` beats `/items/{id}`
- `/items/{id}` beats `/items/*tail`
- `POST /items/special` does not block `GET /items/{id}` from handling `GET /items/special`

If two matching routes are equally specific, first registration wins. (TODO: fix this)

For the same HTTP method, overlapping effective patterns are rejected. For example, `/items/{id}` conflicts with `/items/{slug}`.

## Composition

Reusable modules should export an `install(g: http_router.RouteGroup)` function. The caller decides where that bundle lives by installing it under a prefix.

```acton
# api_routes.act
import http_router


def install(g: http_router.RouteGroup):
    g.get("/info", info_handler)
    g.get("/items/{{id}}", item_handler)

    def install_admin(admin: http_router.RouteGroup):
        admin.get("/stats", stats_handler)

    g.route("/admin", install_admin)
```

```acton
# main.act
import http_router
import api_routes


r = http_router.Router()
r.get("/health", health_handler)
r.route("/api/v1", api_routes.install)
```

The example above installs:

- `/health`
- `/api/v1/info`
- `/api/v1/items/{id}`
- `/api/v1/admin/stats`

The same bundle can be reused at multiple prefixes:

```acton
r.route("/", api_routes.install)
r.route("/api/v1", api_routes.install)
```

Multiple bundles can also share `"/"` as long as they do not register overlapping routes.

## Middleware

Use router middleware for behavior that should apply everywhere:

```acton
r.use(auth_middleware)
r.use(http_router.access_log_middleware(log))
```

Use group middleware for behavior scoped to one bundle:

```acton
def install(g: http_router.RouteGroup):
    g.use(api_auth_middleware)
    g.get("/ping", ping_handler)
```

Middleware order is:

1. Router middleware
2. Route-group middleware
3. Handler

Scoped middleware is captured when a route or child group is registered. A later `g.use(...)` call does not retroactively change routes that were already installed.

## Request context

`Context` includes the normalized request path, the original raw path, path params, and request-scoped values.

Useful helpers:

- `ctx.param(name, default="")`
- `ctx.query_param(name, default="")`
- `ctx.query()`
- `ctx.set(key, value)`
- `ctx.get(key)`

Example:

```acton
def inject_device(next_handler):
    def wrapped(ctx: http_router.Context, respond: proc(int, dict[str, str], str) -> None):
        ctx.set("device_name", "edge-1")
        next_handler(ctx, respond)
    return wrapped
```

## Errors and defaults

- Missing routes return `404`
- Method mismatches also return `404`
- Invalid paths return `400`
- Uncaught handler failures return `500`

You can replace the default `404` behavior:

```acton
def custom_not_found(ctx: http_router.Context, respond: proc(int, dict[str, str], str) -> None):
    respond(404, {"Content-Type": "text/plain"}, "missing:{ctx.path}")


r.not_found(custom_not_found)
```

The package also includes:

- `http_router.recoverer_middleware()`
- `http_router.access_log_middleware(log)`

## Path handling

Incoming paths are normalized before matching:

- Repeated slashes are collapsed
- `.` path segments are ignored
- `..` is rejected
- Backslashes are rejected
- The query string does not affect route matching

So `/api//v1/./items/42?q=x` matches the same route as `/api/v1/items/42`.

## TODOs

- Stronger route overlap detection
- Subrouter mounts
- Trie-based route matching
- Optional `405 Method Not Allowed` handling
