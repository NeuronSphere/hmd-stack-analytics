# Changelog

## 2026-11-13

- fix: remove a registry pull secret (`docker_config_json`, `dockerRepoSecret`) from the `hmd-inf-ext-secrets` entry of `meta-data/reference-bom.json`; it had been copied from an environment's configuration. Releases up to and including 0.1.4 carry it, so the token it held should be treated as exposed and revoked. Nothing reads these keys from the BOM

## 2026-10-26

- feat: declare the stack as a bundle, `meta-data/bundles/analytics.json` (hmd-ms-deployment NERD0019), written by `nsctl bundle migrate` from the manifest's `local` section, which is removed along with `deploy.dependencies`; the lock is unchanged and still covers every want. The README and CI workflow use `nsctl bundle`, and need an nsctl with bundle support

## 2026-09-24

- chore: bump pinned Airflow (hmd-app-airflow 0.4.351 -> 0.4.352) and Superset (hmd-inf-superset 0.5.239 -> 0.5.240) to their latest published builds

## 2026-09-21

- feat: analytics stack -- airflow, trino and superset derived from the local platform
