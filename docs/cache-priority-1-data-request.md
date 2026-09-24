# Data request: projects that can add a cache

**Goal:** identify projects and orgs where adding a cache is an easy, big win.

All the data needed is available today (config, steps, step times, pipeline counts).

## Include a job when all rules are true

| Rule | Starting threshold |
|---|---|
| The job has an install or build step, and the repo has a lockfile | Step command is one of: `npm ci`, `yarn install`, `pnpm install`, `pip install -r`, `poetry install`, `bundle install`, `mvn`, `go mod download`, `cargo fetch`, `cargo build` |
| That step is slow | Median step time is 30s or more (last 30 days) |
| The job runs often | 20 or more runs per week |
| Dependencies do not change much | The lockfile changes in less than 20% of runs |
| The job has no cache today | No `restore_cache` or `save_cache` steps, and no CircleCI function or orb that caches |

## Exclude

- Projects that use Gradle, Bazel, or Turborepo. They are a better fit for the build tool cache server.

## Columns to return (one row per job)

- Org ID and org name
- Project slug and job name
- Install step command and lockfile type
- Median install step time (seconds)
- Runs per week
- Lockfile change rate (% of runs)
- Function available: yes if a CircleCI function with caching exists for this language (today: `setup-go`, `setup-node`, `setup-python`, `setup-ruby`)
- Estimated minutes saved per month

## Sort

First the rows with "Function available: yes", because adding a cache there is one step. Then by estimated minutes saved per month, highest first:

```
minutes saved per month = runs per month × (1 − lockfile change rate) × (median step time − 10s) / 60
```

The 10s is a guess for the time to restore the cache. We cannot measure it before a cache exists.

## Notes

- The thresholds are a starting point. If the list is too long or too short, change the step time and runs per week first.
- Please confirm table names and coverage with the data team.
- CircleCI functions are still in development. Check with the functions team before suggesting them to customers.
