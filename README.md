# cache-optimization-matrix

A test repo to measure how dependency caches behave in CircleCI, across 5
package managers.

## Projects

Each `projects/<name>` folder has a small manifest or lockfile that installs
the real published package. This runs each ecosystem's real install command
and cache.

| Project | Ecosystem | Install command | Cache path |
|---|---|---|---|
| flask, pydantic, black | Python (pip) | `pip install --user -r requirements.txt` | `~/.cache/pip` |
| serde, rayon | Rust (cargo) | `cargo fetch` | `~/.cargo/registry` |
| gson, guava | Java (maven) | `mvn dependency:resolve` | `~/.m2/repository` |
| zod (yarn), rxjs (npm) | Node | `yarn install` / `npm ci` | `~/.cache/yarn`, `~/.npm` |
| lo, hugo | Go | `go mod download` | `~/go/pkg/mod/cache/download` |

## Workflows

Both workflows run on every push, so one pipeline gives a side by side
comparison.

- **`baseline`**: the cache key ends in `{{ epoch }}` and there is no
  fallback key, so the cache never hits.
- **`optimized`**: the cache key is a checksum of the project's lockfile or
  manifest, so it hits on every run after the first one while dependencies
  don't change.

## What we found

- **hugo:** caching only `~/go/pkg/mod/cache/download`, instead of all of
  `~/go/pkg/mod`, cut the stored cache from 423 MiB to 232 MiB and the
  restore time from 7.4s to 3.0s. We have not measured the extra time later
  steps spend extracting modules, so we don't know yet if the job is faster
  overall.
- **Changing what a cache saves needs a new key.** `save_cache` skips a key
  that already exists, so the hugo change only worked after we renamed the
  key (`v2-`). The old cache stays in storage until the org's retention
  period ends.
- **rxjs:** using `npm install` instead of `npm ci` changed
  `package-lock.json` during the install. The key is a checksum of that file,
  so the saved key never matched the next restore. `npm ci` fixed it.
- **zod (not explained):** during runs with several branches at the same
  time, one save was skipped and the next run found no cache. Later, two runs
  with no change to `yarn.lock` computed different checksums. We don't know
  the cause, so we don't claim a fix.

## Limits

Only dependency caches were tested, with small projects and a small number
of runs. Build, linter, and toolchain caches were not tested.
