# cache-optimization-matrix

A dependency-install cache test bed spanning 5 package-manager ecosystems, used
to measure the before/after impact of cache-key strategy changes against the
[Cache Optimization Candidate Tree](../circleci-cli).

## Projects

Each `projects/<name>` directory holds a minimal manifest/lockfile that pulls
in the real published package for that project, so each ecosystem's actual
dependency-install command and cache behavior is exercised.

| Project | Ecosystem | Install command | Cache path |
|---|---|---|---|
| flask, pydantic, black | Python (pip) | `pip install --user -r requirements.txt` | `~/.cache/pip` |
| serde, rayon | Rust (cargo) | `cargo fetch` | `~/.cargo/registry` |
| gson, guava | Java (maven) | `mvn dependency:resolve` | `~/.m2/repository` |
| zod (yarn), rxjs (npm) | Node | `yarn install` / `npm ci` | `~/.cache/yarn`, `~/.npm` |
| lo, hugo | Go | `go mod download` | `~/go/pkg/mod` |

## Workflows

- **`baseline`** — cache key is `{{ epoch }}` (changes every run → guaranteed
  miss on every run). Models the "cache key has a volatile token" leaf of the
  decision tree.
- **`optimized`** — cache key is a checksum of the project's own
  manifest/lockfile (stable across runs when dependencies don't change).
  Models the "fully optimized" leaf.

Both workflows run on every push, so a single pipeline run gives a direct
before/after timing comparison across all 11 projects.
