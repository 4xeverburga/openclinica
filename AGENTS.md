# AGENTS — running OpenClinica here

OpenClinica Community 3.18-SNAPSHOT: JSP / Spring 3.2 / Java 7, a Maven
multi-module build (`core`, `web`, `ws`). Upstream:
https://github.com/OpenClinica/OpenClinica (LGPL, see `UPSTREAM.md`).

The toolchain is installed already: `JAVA_HOME` is **JDK 8** (Java 7
source/target does not compile on JDK 9+), `mvn`, `java`, `javac`, `jar` and
`psql` are on the path, and Tomcat 9 lives at `$CATALINA_HOME`.

---

## 1. Build

```bash
mvn -B -DskipTests \
  -DdbType=postgres -DdbUser=clinica -DdbPass=clinica \
  -Ddb=openclinica -DdbPort=5432 -DdbHost=db \
  -DWEBAPP=OpenClinica -DWEBAPP.lower=openclinica \
  -Dcatalina.home="$CATALINA_HOME" \
  package
```

The root pom's `<filters>` block is **commented out upstream**, so every
`${...}` placeholder in `datainfo.properties` resolves from `-D` system
properties at build time. That is why the block above passes all of them:

- `dbType`, `dbUser`, `dbPass`, `db`, `dbPort`, `dbHost` — the JDBC settings;
- `WEBAPP` / `WEBAPP.lower` — the context name, used by `sysURL` and friends;
- `catalina.home` — `log.dir` is built from it.

The WAR is `web/target/OpenClinica-web-3.18-SNAPSHOT.war`. Deploy it as
**`OpenClinica.war`**: the app derives its webapp name (and therefore its
context path) from the deployed directory name, so the context must be
`/OpenClinica`.

`mvn -B -o …` works offline once the dependencies are in `~/.m2`.

## 2. The database

PostgreSQL 9.6 at host `db`, port 5432, user `clinica`, password `clinica`,
database `openclinica`.

9.6 is not a preference: the driver this pom pins
(`postgresql:9.1-901-1.jdbc4`) predates SCRAM, which every server from 10 on
defaults to.

**The app creates its own schema.** Liquibase runs at startup
(`applicationContext-core-db.xml` → `classpath:migration/master.xml`, ~983
changesets, ~122 tables) and seeds the reference data as it goes. Do not build
tables by hand; drop and recreate the database if you need a clean seed.

`psql` is available:

```bash
psql -h db -U clinica -d openclinica -c '\dt'
```

## 3. Run

```bash
mkdir -p "$CATALINA_HOME/webapps/OpenClinica"
(cd "$CATALINA_HOME/webapps/OpenClinica" && jar xf /path/to/OpenClinica.war)
"$CATALINA_HOME/bin/catalina.sh" run
```

The app answers on `http://localhost:8080/OpenClinica/` and redirects to
`/OpenClinica/pages/login/login`. Logs go to `$CATALINA_HOME/logs/catalina.out`
(and `$CATALINA_HOME/logs/openclinica/` for the app's own).

Explode the WAR yourself when you need to change the configuration: Tomcat
would otherwise explode it on start, after the properties were already read.

## 4. The config the app reads

`WEB-INF/classes/datainfo.properties` inside the deployed webapp is the primary
configuration — Spring's `util:properties` reads it from the classpath. A copy
under `$CATALINA_HOME/openclinica.config/` only overrides a few `CoreResources`
keys; do not rely on it.

Two values matter most while developing:

- `dbHost` / `db` / `dbUser` / `dbPass` / `dbPort` — where the data lives;
- `sysURL` — the absolute base the app builds links from. It must match the
  address you actually serve on, or redirects and callback URLs point at the
  wrong port. A build made with the flags above sets it to
  `http://localhost:8080/OpenClinica/MainMenu`.

## 5. The seeded account

Liquibase seeds one account, user name `root`, from the tracked changelog
`core/src/main/resources/migration/2.5/changeLogDataInsert.xml` (its password
is the upstream MD5 hash of the published default).

Two quirks follow from that seed:

- the stored `passwd_timestamp` is from 2006, so the app forces a
  reset-password page on the first login;
- the encoder is `ShaPasswordEncoder` (unsalted SHA-1) for current passwords,
  with MD5 kept as a legacy fallback — `OpenClinicaPasswordEncoder` tries both.

To use the account without the reset page, store a fresh SHA-1 and stamp it
today:

```bash
SHA="$(printf '%s' '<password>' | sha1sum | cut -d' ' -f1)"
psql -h db -U clinica -d openclinica -c \
  "UPDATE user_account SET passwd = '$SHA', passwd_timestamp = current_date \
   WHERE user_name = 'root'"
```

New accounts created through the UI get a random 8-character password emailed in
plaintext; `mailHost` in `datainfo.properties` is a placeholder, so that mail
never arrives locally.

## 6. Quirks worth knowing

- **Java 8 only.** A newer JDK fails the compiler before it reaches the app.
- **The webapp name is the context.** Deployed as anything but
  `OpenClinica.war`, every link 404s.
- **The login and home pages fetch a live RSS feed** from openclinica.com.
  Cosmetic and it degrades gracefully, but it makes `domcontentloaded` slow —
  expect timeouts in a browser automation that waits for it.
- **Keep-alive stalls.** The app occasionally pauses for a few seconds when a
  connection is reused; a reload recovers it.
- `core/catalina.home_IS_UNDEFINED/` is residue from a build that ran without
  `-Dcatalina.home` (logback wrote logs relative). Build outputs (`target/`,
  `catalina.home_IS_UNDEFINED/`) are ignored: they do not survive a fresh
  clone and are not part of the code.
- `mvn test` runs the unit tests. There is no browser automation in the
  repository.
