# Extract ops/service Module — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #118 — Extract ops/service module
**Issue group:** #118

**Goal:** Extract all Java source from ops/app into a new ops/service
library module so that Scaffold and future aggregators can embed ops
by depending on `casehub-ops-service` directly.

**Architecture:** Create ops/service as a plain JAR with Jandex index.
All production code, JPA entities, Flyway migrations, and CDI services
move there. ops/app becomes a thin Quarkus shell (application.properties
+ quarkus-maven-plugin). Package rename: `io.casehub.ops.app` →
`io.casehub.ops.service`. Tests split: `@QuarkusTest` tests stay in
app (need Quarkus application context), plain JUnit tests move to
service.

**Tech Stack:** Java 21, Quarkus 3.x, Maven, Flyway, JPA/Panache,
Fabric8 Kubernetes client

## Global Constraints

- Module build order: `api → container → service → app`
- ops/service is a plain JAR (Jandex-indexed), NOT a Quarkus extension
- Package root: `io.casehub.ops.service`
- `casehub-platform-graphql-generator` annotation processor runs in
  service (where `@McpDomain` classes live)
- Flyway migrations live with their entities in service
- `application.properties` stays in app — it's Quarkus application config
- No external modules depend on `casehub-ops-app`, so the package
  rename has zero cross-module impact

---

## Batch 1: Scaffold service module

Safe wrap point after completion — green build with empty new module.

### Task 1: Create service/pom.xml + update parent pom

**Files:**
- Create: `service/pom.xml`
- Modify: `pom.xml` (parent — add module + dependencyManagement entry)

**Interfaces:**
- Produces: `casehub-ops-service` Maven artifact (initially empty)

- [ ] **Step 1: Create service module directory structure**

```bash
mkdir -p service/src/main/java/io/casehub/ops/service
mkdir -p service/src/test/java/io/casehub/ops/service
mkdir -p service/src/main/resources
```

- [ ] **Step 2: Write service/pom.xml**

All production dependencies from app/pom.xml move here, plus the
annotation processor config and Jandex plugin. Test dependencies for
the 58 plain JUnit tests that will move here.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-ops-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-ops-service</artifactId>

    <name>CaseHub Ops :: Service</name>
    <description>Service layer library for ops — APIs, entities, CDI services, K8s
        implementations, SPI stubs, case descriptors, and Flyway migrations. Embeddable
        by any Quarkus application (ops/app standalone or Scaffold aggregator).</description>

    <dependencies>
        <!-- CaseHub foundation -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work-engine-adapter</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-planning</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-persistence-memory</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-scheduler-quartz</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-work</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ras-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-work</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-expression</artifactId>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-blocks-summarisation-yaml</artifactId>
        </dependency>

        <!-- Ops domain modules -->
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-container</artifactId>
            <version>${project.version}</version>
        </dependency>

        <!-- Quarkus (library-compatible) -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-rest-jackson</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-hibernate-orm-panache</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-flyway</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-h2</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-hibernate-validator</artifactId>
        </dependency>

        <!-- Kubernetes -->
        <dependency>
            <groupId>io.fabric8</groupId>
            <artifactId>kubernetes-client</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.fabric8</groupId>
            <artifactId>kubernetes-server-mock</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>io.casehub</groupId>
                            <artifactId>casehub-platform-graphql-generator</artifactId>
                            <version>${casehub.version}</version>
                        </path>
                    </annotationProcessorPaths>
                    <compilerArgs>
                        <arg>-AdomainFilter=ops/applications,ops/approvals,ops/cases,ops/clusters,ops/deployments,ops/reconciliation,ops/security,ops/services</arg>
                        <arg>-AgenerateGraphQL=false</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 3: Update parent pom.xml — add service module**

In the `<modules>` section of `pom.xml`, add `<module>service</module>`
immediately before `<module>app</module>` (service must build before app):

```xml
<modules>
    <module>api</module>
    <module>container</module>
    <module>service</module>
    <module>app</module>
    <module>deployment</module>
    <module>infra</module>
    <module>compliance</module>
    <module>testing</module>
    <module>topology-tests</module>
    <module>examples/deployment-monitoring</module>
</modules>
```

- [ ] **Step 4: Update parent pom.xml — add dependencyManagement entry**

Add to `<dependencyManagement><dependencies>` after the casehub-ops-api
entry:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-ops-service</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 5: Verify empty module compiles**

Run: `mvn --batch-mode -o compile -pl service`
Expected: BUILD SUCCESS (empty module compiles with Jandex)

- [ ] **Step 6: Commit**

```bash
git add service/pom.xml pom.xml
git commit -m "feat(#118): scaffold ops/service module

Refs #118"
```

---

## Batch 2: Extract source + slim app

Must complete all tasks before the build is green. Not a safe wrap
point until Task 4 passes.

### Task 2: Package rename + source extraction

**Files:**
- Rename: `io.casehub.ops.app` → `io.casehub.ops.service` (via `ide_refactor_rename`)
- Move: all 97 main source files from `app/` → `service/` (via `ide_move_file`)
- Move: all resources from `app/src/main/resources/` → `service/src/main/resources/`
  (except `application.properties` which stays in app)

**Interfaces:**
- Consumes: empty service module from Task 1
- Produces: service module with all production source + resources

- [ ] **Step 1: Package rename via ide_refactor_rename**

Use `ide_refactor_rename` on the package `io.casehub.ops.app`:

```
ide_refactor_rename(
  symbol: "io.casehub.ops.app",
  language: "Java",
  newName: "service",
  project_path: "/Users/mdproctor/claude/casehub/ops"
)
```

This renames `io.casehub.ops.app` → `io.casehub.ops.service` across
all source files — package declarations, import statements, and
intra-package references. After this step, all files are still
physically in `app/` but their package is `io.casehub.ops.service`.

Verify: check a few files to confirm package declarations changed.

**⚠ JPQL string literals:** The `@NamedQuery` in `ApplicationEntity`
contains `io.casehub.ops.app.model.ApplicationStatus` in a JPQL string.
IntelliJ may not update string literals. After the rename, search for
any remaining `io.casehub.ops.app` references:

```
ide_search_text(query: "io.casehub.ops.app", project_path: "...")
```

If found in JPQL strings or other string literals, update them manually
to `io.casehub.ops.service`.

- [ ] **Step 2: Move main source files to service module**

After the package rename, all main source files are in:
`app/src/main/java/io/casehub/ops/service/`

Move each file to the corresponding directory under service/ using
`ide_move_file`. Group by package directory:

| Source directory (in app/) | Destination (in service/) | File count |
|---------------------------|--------------------------|------------|
| `.../service/api/` | `service/src/main/java/io/casehub/ops/service/api` | 8 |
| `.../service/authorization/` | `service/src/main/java/io/casehub/ops/service/authorization` | 1 |
| `.../service/case_/` | `service/src/main/java/io/casehub/ops/service/case_` | 9 |
| `.../service/container/` | `service/src/main/java/io/casehub/ops/service/container` | 1 |
| `.../service/entity/` | `service/src/main/java/io/casehub/ops/service/entity` | 5 |
| `.../service/goal/` | `service/src/main/java/io/casehub/ops/service/goal` | 2 |
| `.../service/k8s/` | `service/src/main/java/io/casehub/ops/service/k8s` | 12 |
| `.../service/lifecycle/` | `service/src/main/java/io/casehub/ops/service/lifecycle` | 3 |
| `.../service/lifecycle/ras/` | `service/src/main/java/io/casehub/ops/service/lifecycle/ras` | 4 |
| `.../service/model/` | `service/src/main/java/io/casehub/ops/service/model` | 12 |
| `.../service/persistence/` | `service/src/main/java/io/casehub/ops/service/persistence` | 3 |
| `.../service/rest/` | `service/src/main/java/io/casehub/ops/service/rest` | 2 |
| `.../service/rest/dto/` | `service/src/main/java/io/casehub/ops/service/rest/dto` | 4 |
| `.../service/service/` | `service/src/main/java/io/casehub/ops/service/service` | 15 |
| `.../service/spi/` | `service/src/main/java/io/casehub/ops/service/spi` | 7 |

For each file, call:
```
ide_move_file(
  file: "app/src/main/java/io/casehub/ops/service/<pkg>/<File>.java",
  destination: "service/src/main/java/io/casehub/ops/service/<pkg>",
  project_path: "/Users/mdproctor/claude/casehub/ops"
)
```

After all moves, verify `app/src/main/java/` contains no `.java` files:
```bash
find app/src/main/java -name "*.java" -type f | wc -l
```
Expected: 0

- [ ] **Step 3: Move resources to service module**

Move Flyway migrations and META-INF resources. These are non-code
files, so bash is acceptable:

```bash
# Flyway migrations
mkdir -p service/src/main/resources/db/app/migration
mv app/src/main/resources/db service/src/main/resources/

# MCP domain registration
mkdir -p service/src/main/resources/META-INF
mv app/src/main/resources/META-INF/casehub-mcp-app.properties service/src/main/resources/META-INF/

# Summarisation config
mkdir -p service/src/main/resources/META-INF/summarisation
mv app/src/main/resources/META-INF/summarisation/deployment-monitoring.yaml service/src/main/resources/META-INF/summarisation/
```

Verify `application.properties` is still in `app/src/main/resources/`.

- [ ] **Step 4: Commit checkpoint**

```bash
git add -A
git commit -m "wip(#118): rename package + move source and resources to service

Refs #118"
```

### Task 3: Split tests + slim app pom + fix configs

**Files:**
- Move: 58 test files from `app/src/test/` → `service/src/test/` (via `ide_move_file`)
- Modify: `app/pom.xml` — slim to thin shell
- Modify: `app/src/main/resources/application.properties` — update package ref
- Keep in app: 9 `@QuarkusTest` test files

**Interfaces:**
- Consumes: service module with all production source (Task 2)
- Produces: complete module split — service has source + unit tests,
  app has Quarkus shell + integration tests

- [ ] **Step 1: Move plain JUnit tests to service**

Move all 58 non-`@QuarkusTest` test files from `app/src/test/java/`
to `service/src/test/java/`. The following 9 files stay in app:

**Stay in app (need Quarkus application context):**
1. `entity/ApplicationEntityTest.java`
2. `entity/ClusterReferenceEntityTest.java`
3. `entity/DeploymentRecordEntityTest.java`
4. `rest/ApplicationResourceTest.java`
5. `rest/ApprovalResourceTest.java`
6. `rest/ClusterResourceTest.java`
7. `rest/DeploymentResourceTest.java`
8. `service/ApplicationLifecycleServiceTest.java`
9. `service/ClusterServiceTest.java`

All other test files move. Use `ide_move_file` with the same
source→destination pattern as Task 2, but under `src/test/java`.

After moves, verify app test count:
```bash
find app/src/test/java -name "*.java" -type f | wc -l
```
Expected: 9

```bash
find service/src/test/java -name "*.java" -type f | wc -l
```
Expected: 58

- [ ] **Step 2: Slim app/pom.xml to thin shell**

Replace the entire `app/pom.xml` content with:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-ops-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-ops-app</artifactId>

    <name>CaseHub Ops :: App</name>
    <description>Thin Quarkus application shell for ops. All service logic lives in
        casehub-ops-service; this module adds only the Quarkus application packaging
        and runtime configuration.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-ops-service</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-engine-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-desiredstate-testing</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.fabric8</groupId>
            <artifactId>kubernetes-server-mock</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals><goal>build</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

    <profiles>
        <profile>
            <id>skip-quarkus-tests</id>
            <activation>
                <property>
                    <name>!runQuarkusTests</name>
                </property>
            </activation>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-surefire-plugin</artifactId>
                        <configuration>
                            <excludes>
                                <exclude>**/entity/*Test.java</exclude>
                                <exclude>**/rest/*Test.java</exclude>
                                <exclude>**/service/ApplicationLifecycleService*Test.java</exclude>
                                <exclude>**/service/ClusterServiceTest.java</exclude>
                            </excludes>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project>
```

Key changes from current app/pom.xml:
- All production dependencies removed — single `casehub-ops-service` dep
- Annotation processor config removed (moved to service)
- Test dependencies retained for the 9 `@QuarkusTest` tests
- `skip-quarkus-tests` profile retained

- [ ] **Step 3: Update application.properties — hibernate package ref**

In `app/src/main/resources/application.properties`, update the
Hibernate ORM packages line:

Old: `io.casehub.ops.app.entity`
New: `io.casehub.ops.service.entity`

```properties
quarkus.hibernate-orm.packages=io.casehub.ops.service.entity,io.casehub.ops.compliance,...
```

- [ ] **Step 4: Commit checkpoint**

```bash
git add -A
git commit -m "wip(#118): split tests + slim app pom to thin shell

Refs #118"
```

### Task 4: Build verification

**Files:**
- Potentially modify: any files with remaining broken references

**Interfaces:**
- Consumes: complete module split from Tasks 1-3

- [ ] **Step 1: Full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS — all modules compile, all tests pass

- [ ] **Step 2: If compilation fails — fix remaining references**

Common issues after module extraction:

1. **Stale `io.casehub.ops.app` references in string literals** —
   search and replace in JPQL queries, config keys, CloudEvent types
2. **Missing test dependencies** — if a test in service needs a
   dependency not in service/pom.xml, add it
3. **Quarkus CDI wiring in app tests** — if `@QuarkusTest` tests fail
   because beans moved to service, verify service JAR is on the
   test classpath (it should be via the dependency)
4. **Empty source directories left in app** — clean up any orphaned
   `io/casehub/ops/service/` directories under `app/src/`

- [ ] **Step 3: Verify test counts**

Run: `mvn --batch-mode -o test -pl service`
Expected: 58 tests run, all pass

Run: `mvn --batch-mode -o test -pl app -DrunQuarkusTests=true`
Expected: 9 tests run, all pass (these are the @QuarkusTest tests)

- [ ] **Step 4: Clean up orphaned directories in app**

After all moves, remove empty `io/casehub/ops/service/` tree from
`app/src/main/java/` and `app/src/test/java/` if still present.

```bash
find app/src/main/java -type d -empty -delete 2>/dev/null
find app/src/test/java -type d -empty -delete 2>/dev/null
```

- [ ] **Step 5: Final commit**

```bash
git add -A
git commit -m "feat(#118): extract ops/service module — library + thin app shell

All production source, JPA entities, CDI services, K8s implementations,
Flyway migrations and resources extracted from ops/app into ops/service
library module. ops/app is now a thin Quarkus shell with only
application.properties and quarkus-maven-plugin.

Package rename: io.casehub.ops.app → io.casehub.ops.service.
9 @QuarkusTest integration tests stay in app, 58 unit tests move to service.

Refs #118"
```

---

## References

- [2026-10-06-extract-ops-service-design.md] — design spec this plan implements
- [decisions.md] — D1 (extraction boundary), D2 (Flyway), D3 (naming)
- [app/pom.xml] — current Quarkus application configuration
- [container/pom.xml:62-77] — Jandex plugin reference for library modules
- [app/src/main/resources/application.properties:5] — hibernate-orm.packages with io.casehub.ops.app.entity reference
- [GitHub #118] — focal issue
- [GitHub #117] — deployment profiles (requires embeddable ops)
- [scaffold#53] — wire ops into scaffold (blocked by this)
