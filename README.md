# super-pom — Shared Parent POM for `com.org.llm` Services

This repository is the **corporate parent POM** for all LLM platform services.
It sits at the top of every service's Maven inheritance chain.

## Architecture

![Maven BOM + Parent POM Design](design.svg)

```
org.springframework.boot:spring-boot-starter-parent:4.1.0
        │  inherits
        ▼
  com.org.llm:super-pom:1.0.0   ◄──── imports ──── com.org.learning:learning-bom:1.0.0
  (this repo)
        │  parent
        ├──── llm-gateway
        ├──── llm-chat
        ├──── llm-rag-pipeline
        └──── (all other com.org.llm services)
```

## How to inherit

```xml
<parent>
    <groupId>com.org.llm</groupId>
    <artifactId>super-pom</artifactId>
    <version>1.0.0</version>
    <relativePath/>
</parent>
```

No `<dependencyManagement>`, `<repositories>`, or plugin declarations are needed
in child modules — everything flows down from this POM.

## What this POM provides

### Compiler settings

| Property | Value |
|---|---|
| `java.version` | 25 |
| `maven.compiler.release` | 25 |

### Active plugins (run automatically in every child module)

| Plugin | Phase | Purpose |
|---|---|---|
| `spring-boot-maven-plugin` | package | Executable fat-jar + `build-info` actuator endpoint |
| `git-commit-id-maven-plugin` | initialize | Generates `git.properties` for the `/info` actuator |
| `maven-compiler-plugin` | compile | Wires Lombok + Spring config-processor annotation paths |
| `maven-enforcer-plugin` | validate | Enforces build-environment rules (see below) |
| `maven-surefire-plugin` | test | Unit tests with Java 25 `--add-opens` flag |
| `maven-failsafe-plugin` | integration-test + verify | Integration tests with Java 25 `--add-opens` flag |

### Plugin-management (opt-in — child modules declare to activate)

| Plugin | Use-case |
|---|---|
| `jacoco-maven-plugin` | Code-coverage report (prepare-agent + report) |
| `spotless-maven-plugin` | Code formatting |
| `build-helper-maven-plugin` | Extra source directories |
| `avro-maven-plugin` | Avro schema code generation |
| `openapi-generator-maven-plugin` | OpenAPI client/server stub generation |
| `dependency-check-maven` | Base config overrideable per child module |

To opt into JaCoCo, a child module only needs:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <!-- prepare-agent + report executions inherited from pluginManagement -->
        </plugin>
    </plugins>
</build>
```

## Profiles (inherited by every child module)

| Profile | Command | What it does |
|---|---|---|
| `security-scan` | `mvn verify -Psecurity-scan` | OWASP `dependency-check-maven` CVE scan; fails build on CVSS >= 8. Needs network access to the NVD feed — set `NVD_API_KEY` env var to avoid rate limiting. HTML + JSON reports in `target/`. |
| `mutation-test` | `mvn test -Pmutation-test` | PIT mutation testing (JUnit 5). Scope runs with `-Dpitest.targetClasses=com.org.foo.*` and `-Dpitest.targetTests=...` to keep them fast. HTML report in `target/pit-reports/`. |

Child modules no longer need their own `security-scan` profile — declare nothing
and run the command above from the module directory.

## Enforcer rules

The `maven-enforcer-plugin` runs at the `validate` phase and fails immediately if:

- **`requireJavaVersion [25,)`** — the build JDK is older than Java 25.
  Caught early so an incompatible JDK never produces a confusing compile error.
- **`requireMavenVersion [3.9,)`** — Maven older than 3.9 is used.
  Ensures all developers share reproducible build semantics.
- **`banDuplicatePomDependencyVersions`** — the same dependency appears more
  than once with conflicting versions in a `<dependencies>` block.
  Catches copy-paste errors that would otherwise silently produce the wrong jar.

## How to suppress an OWASP false positive

1. Inspect the HTML report at `target/dependency-check-report.html` to confirm
   the CVE is a false positive.
2. Add a suppression entry in `owasp-suppressions.xml` at the root of the
   affected module (or at the repo root to apply everywhere):

```xml
<suppressions xmlns="https://jeremylong.github.io/DependencyCheck/dependency-suppression.1.3.xsd">
    <suppress>
        <notes>False positive — library does not use the vulnerable code path.</notes>
        <packageUrl regex="true">^pkg:maven/com\.example/my\-lib@.*$</packageUrl>
        <cve>CVE-2024-12345</cve>
    </suppress>
</suppressions>
```

3. Commit the file. The OWASP plugin execution references it via
   `<suppressionFiles>owasp-suppressions.xml</suppressionFiles>`.

## Build order

Install these two POMs before building any service:

```bash
# 1. Install the BOM first
cd /path/to/maven-bom && mvn install

# 2. Install the parent
cd /path/to/super-pom && mvn install

# 3. Build any service
cd /path/to/llm-chat && mvn package
```
