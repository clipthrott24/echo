---
name: testing-echo-standalone
description: How to build, boot and smoke-test Spinnaker Echo standalone (no Orca/Front50/Fiat/Redis) for runtime verification, including a stub Front50 to get /health UP and how to work around SLF4J logging being silent.
---

# Running Spinnaker Echo standalone for end-to-end verification

Echo is a headless backend (no UI). Verify it by booting the built distribution and probing
its HTTP surface; use a GUI terminal (`konsole`) plus Chrome if a recording is needed.

## Build a runnable distribution

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64   # Java 11 for pre-upgrade branches
./gradlew :echo-web:installDist
# -> echo-web/build/install/echo/bin/echo
```

`./gradlew :echo-web:bootRun` also works but `installDist` gives you a stable
`lib/` directory you can inspect or patch for diagnostics.

## Minimal standalone config

Echo reads `${user.home}/.spinnaker/` (or `SPRING_CONFIG_ADDITIONAL_LOCATION`).
Actuator base-path is `/`, so endpoints are `/health`, `/info`, `/env`, **not** `/actuator/*`.

`~/.spinnaker/echo.yml`:

```yaml
server: {port: 8089, address: 127.0.0.1}
spinnaker: {baseUrl: http://localhost:9000}
front50:   {baseUrl: http://localhost:8080}   # required: @Value("${front50.base-url}")
orca:      {baseUrl: http://localhost:8083}
services:
  fiat: {enabled: false, baseUrl: http://localhost:7003}
scheduler: {enabled: false}
redis: {enabled: false}
sql: {enabled: false}
igor: {enabled: false}
management:
  endpoints: {web: {exposure: {include: "*"}}}
  endpoint: {health: {show-details: always}, env: {enabled: true}}
```

Launch:

```bash
JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64 \
SPRING_CONFIG_ADDITIONAL_LOCATION=$HOME/.spinnaker/ \
  ./echo-web/build/install/echo/bin/echo
```

The Spring context starts fine with **no** Redis, SQL, Orca, Front50 or Fiat running.

## Getting /health to report UP

`PipelineCache` (`MonitoredPoller`) polls Front50 `/pipelines?restricted=false` every 30 s and
drives the `monitoredPollerHealth` indicator. Without Front50, `/health` is **503 / DOWN** even
though the app is healthy. Run a 20-line stub so the health assertion is meaningful:

```python
# front50_stub.py -> serve on 127.0.0.1:8080, return [] with Content-Type: application/json for any GET
```

Within ~30 s `/health` flips to `{"status":"UP", ... "monitoredPollerHealth":{"status":"UP"}}`.

## Endpoints worth probing

| Endpoint | Expected |
|---|---|
| `GET /health` | 200, `"status":"UP"` (needs the Front50 stub) |
| `GET /info` | 200 `{}` |
| `GET /env/java.version` | 200, the JVM version actually running |
| `POST /` | 200, empty body — `HistoryController` → `EventPropagator` → notification agents. Body: `{"details":{"source":"test","type":"x","application":"testapp"},"content":{}}` |

## Logging may be silent — check for this first

If stdout shows only the ASCII banner plus
`SLF4J(W): No SLF4J providers were found`, the app is running with a NOP logger and you will
see **no** Spring startup log at all. Cause: `org.slf4j:slf4j-api` resolving to 2.x while
Spring Boot 2.7.x still ships `logback-classic` 1.2.x (which only provides the slf4j-1.7
`StaticLoggerBinder`). Check with:

```bash
./gradlew :echo-web:dependencyInsight --configuration runtimeClasspath --dependency slf4j-api
ls echo-web/build/install/echo/lib/ | grep -E 'slf4j-api|logback-classic'
```

Real fix belongs in the build — add a strict constraint in the root `build.gradle`, which is
the approach that was verified to work on this repo:

```groovy
constraints {
  implementation("org.slf4j:slf4j-api") { version { strictly "1.7.36" } }
}
```

After that, `echo-web/build/install/echo/lib/` should contain `slf4j-api-1.7.36.jar` alongside
`logback-classic-1.2.13.jar` and **zero** `slf4j-api-2.*` jars, and a healthy boot produces
~70 log lines ending in `Started Application in N seconds`. A silent boot of ~15 lines is the
tell-tale symptom. (Moving to `logback-classic` 1.3.x / Boot 3 is the other direction.)

To *observe* logs without changing the build (diagnostics only — never use a patched copy as
evidence that the shipped artifact works), patch a copy of the dist:

```bash
cp -r echo-web/build/install/echo /tmp/echo-diag
rm /tmp/echo-diag/lib/slf4j-api-2.*.jar
cp ~/.gradle/caches/modules-2/files-2.1/org.slf4j/slf4j-api/1.7.36/*/slf4j-api-1.7.36.jar /tmp/echo-diag/lib/
sed -i 's/slf4j-api-2\.[0-9.]*\.jar/slf4j-api-1.7.36.jar/' /tmp/echo-diag/bin/echo
```

**Gotcha:** copy the jar from `~/.gradle/caches/modules-2/...`, never from
`~/.gradle/caches/jars-*/` — the latter are Gradle-instrumented and fail at runtime with
`NoClassDefFoundError: org/gradle/internal/classpath/Instrumented`.

## Verifying the JDK level of the artifacts

```bash
javap -verbose -cp echo-web/build/classes/java/main com.netflix.spinnaker.echo.Application | grep 'major version'
# 65 = Java 21, 61 = 17, 55 = 11
```

Check javac, Groovy and Kotlin outputs separately (`build/classes/java|groovy|kotlin/main`) —
they are configured independently and can drift.

Negative control that proves the bytecode level is real:

```bash
/usr/lib/jvm/java-11-openjdk-amd64/bin/java -cp "echo-web/build/install/echo/lib/*" \
  com.netflix.spinnaker.echo.Application
# expect: UnsupportedClassVersionError ... class file version 65.0 ... up to 55.0
```

## Environment notes

- Maven Central can return HTTP 429 from some networks; a local nginx mirror with
  `/etc/hosts` mapping for `repo.maven.apache.org` and a CA in the Java truststore may already
  be configured — do not dismantle it.
- Comparing dependency resolution against an older commit: `git worktree` breaks the Gradle
  `.git/hooks` lookup. Use `git archive <sha> | tar -x -C /tmp/<dir>` instead.
- `pkill -f 'com.netflix.spinnaker.echo.Application'` can match and kill the tool's own shell.
  Prefer `kill $(pgrep -f 'install/echo/bin/echo')` or an explicit PID.
- **Always confirm port 8089 is free before booting.** A leftover Echo from an earlier run makes
  the new boot die with `APPLICATION FAILED TO START / Port 8089 was already in use`, which is
  easily misread as a regression. Check and clean up with:
  ```bash
  ss -ltnp | grep 8089           # shows the holding PID
  kill -9 <pid>; sleep 3
  ```
  This matters most across repeated verification runs, where a stale *diagnostic* copy
  (e.g. `/tmp/echo-diag`) may still be holding the port and serving `/health`, so you would be
  probing the wrong process entirely. Verify the classpath in `ss -ltnp` output points at
  `build/install/echo`, not `/tmp/echo-diag`.
- To prove logging is live at runtime (not just at boot), snapshot the log line count after
  `Started Application`, hit `/health`, then re-check: new `DispatcherServlet` init lines and
  Retrofit `---> HTTP GET .../pipelines` / `<--- HTTP 200` poll lines should appear.

## Devin Secrets Needed

None — everything above runs fully locally.
