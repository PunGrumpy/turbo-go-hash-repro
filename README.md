# Turborepo Go Hash

Minimal reproduction for a Turborepo bug: with `futureFlags.experimentalGoWorkspaces` enabled, a Go module's `#build` task gets a different hash depending on which task you run, so `turbo run build` and `turbo run typecheck` keep separate cache entries for the same task.

Filed as [vercel/turborepo#14044](https://github.com/vercel/turborepo/issues/14044).

## The graph

Two `go.work` members and no `package.json` in either. `apps/api` is a `main` package that requires `packages/lib`. Both `build` and `typecheck` declare `dependsOn: ["^build"]`.

## Reproduce

Requires Go 1.22 or newer on PATH.

```bash
pnpm install
pnpm turbo run build --dry=json | jq -c '.tasks[] | select(.taskId|endswith("apps/api#build")) | {hash, dependencies}'
pnpm turbo run typecheck --dry=json | jq -c '.tasks[] | select(.taskId|endswith("apps/api#build")) | {hash, dependencies}'
```

The same task ID comes back twice, with a different dependency set and a different hash:

```json
{"hash":"506702f84b985303","dependencies":[]}
{"hash":"30ad4204c95cc6bf","dependencies":["example.com/repro/packages/lib#build"]}
```

Under `build`, the library module's own `#build` is absent from the graph and the `^build` edge goes with it. Under `typecheck`, both exist.

## What it costs

```sh
pnpm turbo run build       # cache miss, executing 506702f84b985303
pnpm turbo run typecheck   # cache miss, executing 30ad4204c95cc6bf
pnpm turbo run build       # cache hit,  replaying  506702f84b985303
```

Alternating the two commands recompiles the Go binary every time. Your hashes will differ; what matters is that the two differ from each other.

## Expected

One task ID, one hash, one cache entry, regardless of which task you asked for.
