# Cache Optimization Candidate Tree — v2

Amended after validating the tree against 20+ real CircleCI runs in this repo.
Two leaves (marked ① ②) gained a diagnostic or a fix step the field test
showed was missing.

```mermaid
flowchart TD
    A["Project / CI Job Step"] --> B{"Dependency-install eligible?"}
    B -->|No| NC["NOT A CANDIDATE<br/>Time: n/a · Cost: n/a"]
    B -->|Yes| C{"Has save_cache / restore_cache steps?"}
    C -->|No| AC["ADD CACHE<br/>Time: faster · Cost: cheaper"]
    C -->|Yes| D{"① Cache key broken by construction?<br/>(volatile token · install step<br/>mutates cache · unscoped<br/>across branches)"}
    D -->|Yes| FK["FIX CACHE KEY<br/>Unlocks: faster + cheaper"]
    D -->|No| E{"Empirical hit rate &lt; 20%?"}
    E -->|Yes| BC["BROKEN CACHE<br/>Time: no change · Cost: wasted"]
    E -->|No| F{"Hit vs miss runtime about the same?"}
    F -->|Yes| NCM["NEUTRALIZED BY CMD<br/>Time: no change · Cost: wasted"]
    F -->|No| G{"Payload large vs actual need?"}
    G -->|Yes| CH["CACHE HARMFUL<br/>Time: slower · Cost: pricier"]
    G -->|No| H{"② Cache storage/egress above<br/>guardrail size?"}
    H -->|Yes| OC["OVERSIZED CACHE<br/>Time: fine · Cost: review TTL<br/>+ bump cache key"]
    H -->|No| FO["FULLY OPTIMIZED<br/>Time: faster · Cost: cheaper"]

    classDef candidate fill:#eceae6,stroke:#7a8088,color:#3a3f46;
    classDef warn fill:#f7e9dd,stroke:#c96a30,color:#7a3d15;
    classDef critical fill:#f3e2ec,stroke:#a8447a,color:#6e2450;
    classDef good fill:#e3f1ec,stroke:#2f9c78,color:#175c42;
    classDef amended stroke-width:3px,stroke:#a8447a;

    class NC candidate
    class AC,FK,BC,NCM,OC warn
    class CH critical
    class FO good
    class D,H amended
```

Each leaf: Time impact + Cost impact.

## Changelog

**① "Fix cache key" diagnostic expanded.** Previously only checked for a
volatile token (`.Revision`, timestamp). Two more root causes produce the
same broken-cache symptom without ever showing a volatile token: an install
command that mutates the restored files before `save_cache` runs (checksum
drifts between restore and save), and a cache key not scoped to the
branch/concurrency dimension, letting parallel runs race for the same cache
object.

Evidence: zod/npm — `npm install` rewrote `package-lock.json` in place,
permanently breaking the cache; separately, zod's unscoped key raced across
concurrent branches and failed to restore.

**② "Oversized cache" fix step added.** Shrinking the cached payload alone
doesn't shrink the cache — `save_cache` no-ops when a key already has a
saved archive, so a stale, oversized archive keeps getting restored forever
unless the key itself changes.

Evidence: hugo — the payload fix alone did nothing until the key was bumped
to `v2-`, which produced a fresh 232MiB archive (down from a stale 423MiB
one).
