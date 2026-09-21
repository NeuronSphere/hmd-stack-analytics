# hmd-stack-analytics

Airflow, Trino and Superset with what they need -- the smallest useful
NeuronSphere analytics platform, installable into a local NeuronSphere
environment with one command and free to use: no tenant, no login, no token.

```sh
nsctl env start
nsctl stack add analytics --apply
```

That is the whole install. The first line brings up the local substrate
(Docker is the only prerequisite; see [nsctl](https://github.com/neuronsphere/hmd-cli-neuronsphere)),
the second fetches this stack from `ghcr.io/neuronsphere/stacks/analytics`,
verifies every pinned RepoClass against its digest, declares the instances
in the environment and deploys them. `nsctl env status` prints where the
three UIs answer once the deploy settles.

| UI       | Where                                        | Sign in                         |
|----------|----------------------------------------------|---------------------------------|
| Airflow  | `http://airflow.local.neuronsphere.io`       | `airflow` / `airflow`           |
| Superset | `http://superset.local.neuronsphere.io`      | `admin` / `admin`               |
| Trino    | `localhost:19033` (JDBC/CLI) or `http://trino.local.neuronsphere.io` | any user, no password; catalogs `hive`, `graph` |

## What is in it

Three roots, and the eight companions they cannot run without. Nothing else:
no transform engine, no ClickHouse, no observability stack.

| Instance                    | RepoClass                     | Role                                                    |
|-----------------------------|-------------------------------|---------------------------------------------------------|
| `airflow`                   | `hmd-app-airflow`             | Workflow orchestration (web, scheduler, workers, pgbouncer) |
| `trino`                     | `hmd-inf-trino`               | Federated SQL; `hive` and `graph` catalogs               |
| `superset`                  | `hmd-inf-superset`            | Dashboards and SQL Lab over Trino                        |
| `hive-metastore`            | `hmd-inf-hive-metastore`      | Table metadata for Trino's `hive` catalog                |
| `redis`                     | `hmd-inf-redis`               | Superset cache and Celery broker                         |
| `secret-key`                | `hmd-inf-superset-secret-key` | Superset's `SECRET_KEY`                                  |
| `trino-users`               | `hmd-inf-credentials`         | The Trino user Airflow connects as                       |
| `airflow-logs`              | `hmd-inf-s3bucket`            | Airflow task logs                                        |
| `superset-db-account`       | `hmd-database-account`        | Superset's Postgres database and user                    |
| `airflow-db-account`        | `hmd-database-account`        | Airflow's Postgres database and user                     |
| `hive-metastore-db-account` | `hmd-database-account`        | The metastore's Postgres database and user               |

Five more roles are **bound**, not bundled: the stack reuses what every
local environment already provides -- the cluster (`eks-cluster`), the
compute/ingress substrate (`local-neuronsphere`), the environment database
(`environment-db`), External Secrets (`ext-secrets`) and the global graph
(`global-graph`). Exact pins live in [`neuronsphere.lock`](neuronsphere.lock);
the declared instance configuration in
[`meta-data/manifest.json`](meta-data/manifest.json).

Every layer carries a licence declaration. The stack itself and all eleven
companion RepoClasses are Apache-2.0.

## Profiles

Each root is a profile of the same name, and all three are on by default:

```sh
nsctl stack add analytics --profile trino            # Trino and the metastore only
nsctl stack add analytics --profile superset,trino   # no Airflow
nsctl stack add analytics --env dev --name superset=bi
```

`nsctl stack list` shows what is declared; `nsctl stack remove analytics`
undeclares it (nothing is torn down until `nsctl env apply`).

## How it is built

This repository is a NeuronSphere RepoClass with an empty build: its
manifest's `local` section and its lock *are* the product. It was derived
from a running local platform with

```sh
nsctl stack init hmd-stack-analytics --from-env local --select superset,trino,airflow
```

which walks each root's dependency roles through the environment's graph,
classifies every instance reached as a companion or a bound role, and
records the environment it read as `meta-data/reference-bom.json`. Two
things in that BOM were then edited by hand: the ClickHouse chain --
reachable only through Trino's *optional* `clickhouse` / `clickhouse-user`
roles -- was pruned, and the two-part versions the environment declares for
`hmd-database-account`, `hmd-inf-credentials` and `hmd-inf-s3bucket` (`0.1`,
a deploy-time range) were replaced by the concrete versions
`nsctl artifact versions <class> --spec "== 0.1.*"` resolves, since a stack
pins exact versions.

[`.github/workflows/stack.yml`](.github/workflows/stack.yml) does the rest
(NERD019 in hmd-cli-neuronsphere):

- **verify**, on every pull request: `nsctl repoclass validate`, then
  `nsctl stack build`, which fetches each pinned zip from the Artifact
  Librarian, checks digests and assembles an OCI image layout that is
  uploaded as a workflow artifact.
- **release**, on every push to `main`: builds again and pushes the layout
  to `ghcr.io/neuronsphere/stacks/analytics` with the next version.
- **refresh**, weekly and on demand: re-derives from the reference BOM and
  opens a pull request when anything moved.

The build needs a credential for the Artifact Librarian, so the four
secrets `HMD_SERVICES_ISSUER`, `HMD_SERVICE_CLIENT_ID`,
`HMD_SERVICE_CLIENT_SECRET` and `HMD_ARTIFACT_LIBRARIAN_URL` must be set on
the repository. Pull requests from forks do not see them and cannot build.

To re-derive locally (offline, from the checked-in BOM) and check that the
manifest still matches:

```sh
nsctl stack init hmd-stack-analytics --path . --from-bom meta-data/reference-bom.json \
  --select superset,trino,airflow --diff
```

## Licence

Apache-2.0. See [LICENSE](LICENSE).
