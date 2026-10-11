# AGENTS — Running OpenClinica locally + test policy

OpenClinica Community 3.18-SNAPSHOT (JSP / Spring 3.2 / Java 7, Maven multi-module
`core`/`web`/`ws`). Upstream: https://github.com/OpenClinica/OpenClinica (LGPL, see
`UPSTREAM.md`). This repo is a HEAD-only shallow mirror.

## 0. Prerequisites

| Thing | Path / command | Notes |
|---|---|---|
| **JDK 8** (mandatory) | `/usr/lib/jvm/jdk-8.0.202-oracle-x64` | Java 7 source/target; never inherit shell `JAVA_HOME` (machine default is JDK 21). |
| **PostgreSQL 9.6** | podman container `openclinica-pg`, port **5433** | 9.6 pinned: the 2011 driver `postgresql:9.1-901-1.jdbc4` cannot do SCRAM (PG 10+). Local system PG needs sudo — don't use it. |
| **Tomcat 9** | `/home/ever/tools/apache-tomcat-oc` | Dedicated instance, ports 8089 (HTTP) / 8105 / 8109. 8080 = lotto Tomcat, 8088 = Mamute. |

## 1. Quick start (or just run the setup script)

Harness scripts live in the bench repo, NOT here — this repo stays a
pristine upstream mirror (only `UPSTREAM.md` + this `AGENTS.md`):

```bash
cd /home/ever/Documents/Me/Work/2026/Kyndryl/Tinka/DemoModernizacion/jsp-decouple-bench/environments/openclinica
./scripts/setup-e2e.sh          # idempotent: container + DB + Tomcat + admin credential
# app: http://localhost:8089/OpenClinica/
# login: root / (password in .env — gitignored, in the bench env dir)
```

Manual build (only needed after code changes):

```bash
JAVA_HOME=/usr/lib/jvm/jdk-8.0.202-oracle-x64 mvn -B -q -DskipTests \
  -DdbType=postgres -DdbUser=clinica -DdbPass=clinica -Ddb=openclinica -DdbPort=5433 -DdbHost=127.0.0.1 \
  -DWEBAPP=OpenClinica -DWEBAPP.lower=openclinica \
  -Dcatalina.home=/home/ever/tools/apache-tomcat-oc \
  package
cp web/target/OpenClinica-web-3.18-SNAPSHOT.war /home/ever/tools/apache-tomcat-oc/webapps/OpenClinica.war
```

Gotchas baked into that command (all verified):
- The root pom's `<filters>` block is **commented out upstream** — every `${...}` in
  `datainfo.properties` resolves from `-D` system properties at build time.
- `sysURL` is hardcoded to port 8080 in `web/src/main/resources/datainfo.properties`;
  sed it to 8089 before building, `git restore` after (the WAR keeps the patched value).
- The WAR **must** be deployed as `OpenClinica.war` — the app derives its webapp name
  from the deployed directory name.
- The classpath `datainfo.properties` inside the WAR is the primary config (Spring
  `util:properties` reads it directly); the external `$CATALINA_HOME/openclinica.config/`
  copy is only a partial override — don't rely on it.
- Liquibase auto-creates the schema at startup (~122 tables) via
  `applicationContext-core-db.xml` → `classpath:migration/master.xml`.

## 2. Admin credentials (gitignored, bench-side)

- **Upstream seeds `root` / `12345678`** (MD5) in the tracked Liquibase changelog
  `core/src/main/resources/migration/2.5/changeLogDataInsert.xml`. That seed is public
  upstream knowledge — it is NOT a secret and cannot be removed without patching upstream.
- The **real** credential lives in `environments/openclinica/.env` in the bench repo
  (**gitignored**; template: `.env.example` next to it).
  `environments/openclinica/scripts/bootstrap-admin.sh` applies it: sets `root`'s
  password to unsalted SHA-1 hex (the app's current encoder) and resets
  `passwd_timestamp` to today (the seed's 2006 timestamp otherwise forces a
  reset-password page on first login).
- **Run `bootstrap-admin.sh` after every fresh database** (new container = seed returns).
  It is idempotent — safe to run anytime.
- Password hashing: `OpenClinicaPasswordEncoder` = SHA-1 (current) + MD5 (legacy
  fallback), both unsalted. New accounts created via the UI get a random 8-char
  password emailed in plaintext (`mailHost` is a placeholder → mail won't deliver locally).

## 3. Test taxonomy — READ BEFORE RUNNING (destructiveness disclosed)

All scripts live in `../jsp-decouple-bench/environments/openclinica/scripts/`.

| Script | Destructive? | What it does | When to run |
|---|---|---|---|
| `scripts/setup-e2e.sh` | **No** — idempotent | Starts container/DB/Tomcat only if missing, creates DB only if absent, applies bootstrap, verifies admin login. Auto-generates `.env` on a fresh checkout. | **Before every e2e suite run.** |
| Regular e2e tests (yours) | **No** | Run against the live, already-bootstrapped app; read the password from `.env`; clean up the entities they create. | Per suite. |
| `scripts/test-admin-from-scratch.sh` | **YES — drops the `openclinica` database and restarts Tomcat** | Proves the full credential lifecycle on a fresh DB: seed authenticates (and forces reset) → bootstrap rotates → seed rejected → gitignored password logs in. | Opt-in only: after credential/bootstrap changes, or to reset a polluted DB. Anything you created in the app is lost. |

**Contract for e2e tests:** never drop/recreate the database, never restart Tomcat,
never hardcode passwords — source `local-admin.properties`. If a test needs clean
state, delete what it created (targeted cleanup), or ask the human to run the
destructive from-scratch script.

## 4. Logs & ports

| What | Where |
|---|---|
| App | `http://localhost:8089/OpenClinica/` |
| Tomcat log | `/home/ever/tools/apache-tomcat-oc/logs/catalina.out` |
| PostgreSQL | `podman exec openclinica-pg psql -U clinica -d openclinica` |
| Data dir | `/home/ever/tools/apache-tomcat-oc/openclinica.data/` |

Known quirks: the login/home pages fetch a live RSS feed from openclinica.com
(cosmetic, degrades gracefully); the app intermittently stalls a few seconds on
keep-alive reuse — a reload recovers it.
