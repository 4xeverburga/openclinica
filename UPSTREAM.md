# Upstream provenance

This repository is a **shallow mirror** of an open-source project, kept for internal
migration / modernization demos and dataset building. It is **not** affiliated with,
endorsed by, or maintained by the original authors.

| | |
|---|---|
| Upstream | https://github.com/OpenClinica/OpenClinica |
| Commit | `69cccaf` (2024-06-10) |
| License | GNU LGPL — stated in [`README.md`](README.md) (note: the upstream repo contains **no `LICENSE` file**) |
| Clone depth | 1 (upstream history is truncated) |
| Local changes | added this `UPSTREAM.md` only |

## About the project

OpenClinica is a clinical trial / electronic data capture (EDC) system. Maven
multi-module project (`core`, `web`, `ws`), Java 7 source/target level, JSP views.

## Runtime requirements (verified)

- **Requires PostgreSQL** (default, `dbType=postgres`) **or Oracle**. There is **no
  embedded/in-memory database option** anywhere in the build or tests.
- Schema is managed by **Liquibase** (`core/src/main/resources/migration/master.xml`,
  which includes 16 release changelogs).
- Tests expect a live PostgreSQL: `jdbc:postgresql://<host>:5432/openclinica-TEST-3.12`,
  user `clinica` / password `clinica`.
- Verified caveat: upstream pins the 2011-era driver `postgresql:9.1-901-1.jdbc4`, which
  **cannot authenticate** against modern PostgreSQL (SCRAM-SHA-256, "authentication type 10").
  Either configure `pg_hba.conf` with `md5`/`password` auth, or upgrade the driver to >= 42.x.

Rights remain with the original authors. Redistributed under the terms of the GNU LGPL.
