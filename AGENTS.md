# AGENTS.md

## Repository Overview

MOSIP's **NFIQ 1.0** (NIST Fingerprint Image Quality) implementation — a
from-scratch **Java re-implementation** of NIST's C algorithm, not a vendored
copy. Package layout (`org.mosip.nist.nfiq1.{mindtct,mlp,imagetools,common,util}`)
mirrors NIST's original C modules for cross-referencing, but there is no
`patches/`, no bundled `.c`/`.h`, no CMake/autotools.

`nfiq1.0/` is the only real module (single Maven project). `nfiq2.0/`
contains only an empty placeholder (`test.txt`) — no NFIQ 2.0 implementation
here; MOSIP components needing NFIQ2 scoring consume
[NIST's NFIQ2](https://github.com/usnistgov/NFIQ2) directly. One module, one
root `AGENTS.md` — no per-module split needed.

## Technology Stack

- **Language**: Java 21, `--enable-preview` set in the POM — build and run
  with a JDK that supports it (required at runtime too, see below).
- **Build**: Maven, `nfiq1.0/pom.xml`, artifact `io.mosip:nfiq1.0`.
- **Test**: JUnit 5 + Mockito.
- **Key libs**: `jai-imageio-jpeg2000` (JP2), `jnbis` (WSQ),
  `io.mosip.kernel:kernel-bom` / `biometrics-util`, Lombok, SLF4J +
  `slf4j-log4j12`. **Log4j 1.x is EOL with no fixes planned** — known risk,
  don't extend it (no remote/JMS/socket appenders — those carry known
  deserialization risks); track migrating to Log4j 2/Logback.
- **Coverage**: JaCoCo. **Static analysis**: SonarCloud via the `sonar`
  Maven profile (not active by default).

## Build & Test Commands

```bash
cd nfiq1.0
mvn clean install -Dgpg.skip=true   # build (skips GPG signing, per README.md)
mvn test                             # tests only
mvn test -Dtest=Nfiq1HelperTest      # single test class
```

Run the sample app after a build (from `nfiq1.0/target`, jar version from
`pom.xml`, e.g. `0.1.1-SNAPSHOT`). Preview classes require `--enable-preview`
at runtime too:

```bash
# POSIX
java --enable-preview -cp "nfiq1.0-0.1.1-SNAPSHOT.jar:lib/*:test-classes" \
  org.mosip.nist.nfiq1.test.NfiqApplication "imgfile=info_jp2.iso" "logs=0"
```
```bat
:: Windows cmd.exe
java --enable-preview -cp nfiq1.0-0.1.1-SNAPSHOT.jar;lib\*;test-classes org.mosip.nist.nfiq1.test.NfiqApplication "imgfile=info_jp2.iso" "logs=0"
```

`nfiq1.0/runJP2.bat`/`runWSQ.bat` run the same harness against bundled
sample ISOs (`info_jp2.iso`/`info_wsq.iso`, copied to `target/` at the
`validate` phase). They hardcode the jar version — update if `pom.xml`'s
`<version>` moved — and, as checked in, they **don't** pass
`--enable-preview` despite needing it; prefer the command above for local
runs.

`NfiqApplication` args: `imgfile` (JP2/WSQ ISO path), `logs` (`0`=score
only, `1`=quality map). Prints a score 1 (best)–5 (worst) plus confidence.

## Configuration

- Logging: `nfiq1.0/src/main/resources/log4j.properties` (log4j 1.x style).
- `nfiq1.0/src/main/resources/znorm.dat` — Z-normalization coefficients
  (`Nfiq1ZNormalization.java`); binary/numeric reference data, don't
  delete/reformat.
- No Spring config, no HTTP endpoints, no DB — plain library JAR, no
  secrets to worry about in the repo itself.
- CI publishing secrets (`OSSRH_USER`, `OSSRH_SECRET`, `OSSRH_TOKEN`,
  `GPG_SECRET`, `SLACK_WEBHOOK`, `SONAR_TOKEN`, `ORG_KEY`) live only in
  `.github/workflows/push-trigger.yml` as GitHub Actions secrets — never
  hardcode them.

## Project Structure Notes

```text
nfiq/
├── nfiq1.0/                     # the only real module (Maven project)
│   ├── pom.xml
│   ├── README.md
│   ├── info_jp2.iso, info_wsq.iso   # sample fingerprint ISOs
│   ├── runJP2.bat, runWSQ.bat
│   └── src/
│       ├── main/java/org/mosip/nist/nfiq1/{mindtct,mlp,imagetools,common,util}/
│       ├── main/resources/      # log4j.properties, znorm.dat
│       └── test/java/org/mosip/nist/nfiq1/   # *Test.java, 1:1 with main classes
│           └── test/NfiqApplication.java  # runnable sample app
├── nfiq2.0/test.txt             # empty placeholder — no real code
├── licenses/                    # third-party license texts
├── .github/workflows/push-trigger.yml
└── README.md
```

Test classes shadow main classes 1:1 (e.g. `Block.java` → `BlockTest.java`)
— check/update the matching test when changing a main class.

## Development Workflow

CI (`push-trigger.yml`) triggers on: release published, PR
opened/reopened/synchronize (any base branch), `workflow_dispatch`, and
pushes to `MOSIP*`/`develop*`/`master`/`1.*`/`release*`. Always runs the
Maven build (`mosip/kattu`, `SERVICE_LOCATION: ./nfiq1.0`); also runs
`publish_to_nexus` and `sonar_analysis` on non-PR events. No `paths:`
filter — any repo change triggers the full workflow.

Before opening a PR, run `mvn clean install -Dgpg.skip=true` from
`nfiq1.0/` locally. Keep the `mindtct`/`mlp` layout aligned with NIST's
original module structure for cross-referencing.

## Pull Request Guidelines

- Verify the default branch first — `gh repo view mosip/nfiq --json
  defaultBranchRef`. GitHub reports `master` as default, but `develop` is
  the active integration branch recent feature PRs target; open feature
  PRs against `develop` unless backporting to a release branch.
- Reference the tracking issue (`#<issue-number>: <summary>` title style).
- Sign off commits (`git commit -s`).
- Keep changes scoped to `nfiq1.0/` unless the PR is about root-level
  concerns (CI, licensing, top-level docs).
- Don't bump `nfiq1.0/pom.xml`'s `<version>` as a side effect — version
  bumps are their own PRs.
- If touching `runJP2.bat`/`runWSQ.bat`/`NfiqApplication`, verify the
  hardcoded jar filename still matches `pom.xml`'s `<version>`.

## Repository-Specific Considerations

- `licenses/` filenames carry trailing invisible Unicode marks (inherited
  from how they were checked in) — expected; don't "fix" without checking
  downstream references first.
- `nfiq1.0/logs/nfiq1.log` is a tracked-but-empty log file — don't grow it
  with local run output in an unrelated commit.
- `--enable-preview` is set in both compiler and surefire `argLine` — any
  JVM invocation you add must place `-D` system properties **before**
  `-jar`/`-cp`, not after.
- This is a Maven dependency (`io.mosip:nfiq1.0`) consumed by Biometric
  SDK, Registration Processor, ID Authentication, and MDS — treat public
  method signatures in `org.mosip.nist.nfiq1` as an API surface.

## Agent rules

### Do

1. Verify the default branch (`gh repo view mosip/nfiq --json
   defaultBranchRef`) before branching — don't assume.
2. Build/test from `nfiq1.0/` via `mvn clean install -Dgpg.skip=true`.
3. Keep new/changed classes under the existing
   `mindtct`/`mlp`/`imagetools`/`common`/`util` package layout.
4. Add/update a matching `*Test.java` for every changed main class.
5. Sign off commits and reference the tracking issue in titles.
6. Treat `nfiq2.0/` as an empty placeholder — flag it if a task assumes
   NFIQ 2.0 code exists here.

### Do not

1. Don't describe this repo as a vendored/patched NIST C copy — it's an
   original Java implementation; don't invent a `patches/` folder or C
   build system.
2. Don't put `-D` JVM system properties after `-jar`/`-cp`.
3. Don't bump `nfiq1.0/pom.xml`'s `<version>` as a side effect.
4. Don't assume this is a Spring Boot service — no config, no HTTP layer.
5. Don't commit build output (`nfiq1.0/target/`) or modify the tracked
   sample ISO files.
