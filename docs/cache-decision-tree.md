# Cache Optimization Candidate Tree (draft v2)

**Status: draft, will be replaced by a checklist.** A review found that some
steps in this tree are too simple:

- A key that changes every run is only broken when there is no stable
  fallback key. With a fallback key, it restores the most recent cache.
- The 20% hit rate limit has no evidence behind it. A low hit rate can still
  be worth it when each hit saves a lot of time.
- Many problems can happen at the same time, so a single path through a
  tree is not enough.
- The tree was written for jobs with hand-written cache steps. CircleCI
  functions (like `setup-go`) have no cache steps, so the second question
  does not fit them.

```mermaid
flowchart TD
    A["Project / CI Job Step"] --> B{"Dependency-install eligible?"}
    B -->|No| NC["NOT A CANDIDATE<br/>Time: n/a · Cost: n/a"]
    B -->|Yes| C{"Has save_cache / restore_cache steps?"}
    C -->|No| AC["ADD CACHE<br/>Time: faster · Cost: cheaper"]
    C -->|Yes| D{"① Cache key broken by construction?<br/>(changes every run with no fallback ·<br/>install step changes the keyed file)"}
    D -->|Yes| FK["FIX CACHE KEY<br/>Unlocks: faster + cheaper"]
    D -->|No| E{"Empirical hit rate &lt; 20%?"}
    E -->|Yes| BC["BROKEN CACHE<br/>Time: no change · Cost: wasted"]
    E -->|No| F{"Hit vs miss runtime about the same?"}
    F -->|Yes| NCM["NEUTRALIZED BY CMD<br/>Time: no change · Cost: wasted"]
    F -->|No| G{"Payload large vs actual need?"}
    G -->|Yes| CH["CACHE HARMFUL<br/>Time: slower · Cost: pricier"]
    G -->|No| H{"② Cache storage/egress above<br/>guardrail size?"}
    H -->|Yes| OC["OVERSIZED CACHE<br/>Time: fine · Cost: review retention<br/>(new key needed to change contents)"]
    H -->|No| FO["NO ISSUE FOUND<br/>in the tested runs"]

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

## Changes from v1

**① "Fix cache key" covers more cases.** v1 only checked for a key that
changes every run. We also found a second case: an install command that
changes the file used for the key. In rxjs, `npm install` rewrote
`package-lock.json`, so the saved key never matched the next restore.

An earlier version of this doc also listed "key not scoped by branch" as a
cause. We removed it because the evidence was not clear (see the zod note in
the README).

**② "Oversized cache" needs a new key to change.** `save_cache` skips a key
that already exists. So a change to what a cache saves only takes effect
with a new key. For hugo, the change worked only after we renamed the key to
`v2-`. A new key does not delete the old cache. It stays in storage until
the org's retention period ends.
