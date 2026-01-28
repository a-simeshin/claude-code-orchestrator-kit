# Java Static Analysis Tools

## Overview

Configuration for static analysis with focus on **critical checks only**. Skip naming conventions, minor style issues - focus on real bugs and security.

---

## Spotless + Palantir (Formatting)

### Maven Configuration

```xml
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <version>2.43.0</version>
    <configuration>
        <java>
            <palantirJavaFormat/>
        </java>
        <markdown>
            <includes>
                <include>**/*.md</include>
            </includes>
            <flexmark/>
        </markdown>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>apply</goal>
            </goals>
            <phase>compile</phase>
        </execution>
    </executions>
</plugin>
```

### Commands

```bash
mvn spotless:check    # Verify formatting
mvn spotless:apply    # Auto-fix formatting
```

---

## JaCoCo (Code Coverage 80%)

### Maven Configuration

```xml
<properties>
    <jacoco.version>0.8.12</jacoco.version>
    <jacoco.coverage.minimum>0.80</jacoco.coverage.minimum>
</properties>

<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${jacoco.version}</version>
    <configuration>
        <excludes>
            <!-- Exclude generated and config classes -->
            <exclude>**/*$MockitoMock*</exclude>
            <exclude>**/*Config.class</exclude>
            <exclude>**/*Application.class</exclude>
            <exclude>**/dto/**</exclude>
            <exclude>**/entity/**</exclude>
        </excludes>
    </configuration>
    <executions>
        <execution>
            <id>prepare-agent</id>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
        <execution>
            <id>check</id>
            <phase>verify</phase>
            <goals>
                <goal>check</goal>
            </goals>
            <configuration>
                <rules>
                    <rule>
                        <element>BUNDLE</element>
                        <limits>
                            <limit>
                                <counter>LINE</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>${jacoco.coverage.minimum}</minimum>
                            </limit>
                            <limit>
                                <counter>BRANCH</counter>
                                <value>COVEREDRATIO</value>
                                <minimum>${jacoco.coverage.minimum}</minimum>
                            </limit>
                        </limits>
                    </rule>
                </rules>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### Commands

```bash
mvn test              # Run tests with coverage
mvn jacoco:report     # Generate HTML report
mvn jacoco:check      # Fail if below 80%
```

---

## SpotBugs (Critical Bugs Only)

### Critical Bug Categories to Enable

| Category | Description | Why Critical |
|----------|-------------|--------------|
| `NP` | Null Pointer Dereference | Runtime crashes |
| `RCN` | Redundant Null Check | Dead code, logic errors |
| `OS` | Open Stream | Resource leaks |
| `ODR` | Open Database Resource | Connection leaks |
| `SQL` | SQL Injection | Security vulnerability |
| `XSS` | Cross-Site Scripting | Security vulnerability |
| `DMI` | Dubious Method Invocation | Incorrect API usage |
| `RV` | Return Value Ignored | Silent failures |
| `EI` | Exposing Internal Rep | Encapsulation break |

### Maven Configuration

```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.3.1</version>
    <configuration>
        <effort>Max</effort>
        <threshold>High</threshold> <!-- Only High priority -->
        <excludeFilterFile>spotbugs-exclude.xml</excludeFilterFile>
    </configuration>
</plugin>
```

### spotbugs-exclude.xml (Skip Non-Critical)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<FindBugsFilter>
    <!-- Skip naming and style issues -->
    <Match>
        <Bug pattern="NM_*"/>
    </Match>

    <!-- Skip in generated code -->
    <Match>
        <Source name="~.*Generated.*"/>
    </Match>

    <!-- Skip in DTOs -->
    <Match>
        <Package name="~.*\.dto\..*"/>
    </Match>
</FindBugsFilter>
```

### Commands

```bash
mvn spotbugs:check    # Run analysis
mvn spotbugs:gui      # View results in GUI
```

---

## PMD (Dead Code & Critical Issues)

### Critical Rules Only

| Rule | Description | Why Critical |
|------|-------------|--------------|
| `UnusedPrivateField` | Unused field | Dead code |
| `UnusedPrivateMethod` | Unused method | Dead code |
| `UnusedLocalVariable` | Unused variable | Dead code |
| `EmptyCatchBlock` | Empty catch | Silent failures |
| `EmptyIfStmt` | Empty if | Logic error |
| `AvoidBranchingStatementAsLastInLoop` | break/continue/return at loop end | Logic error |
| `CloseResource` | Resource not closed | Leak |
| `AvoidReassigningParameters` | Parameter reassignment | Confusing |

### Maven Configuration

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-pmd-plugin</artifactId>
    <version>3.21.2</version>
    <configuration>
        <rulesets>
            <ruleset>pmd-ruleset.xml</ruleset>
        </rulesets>
        <excludeRoots>
            <excludeRoot>target/generated-sources</excludeRoot>
        </excludeRoots>
    </configuration>
</plugin>
```

### pmd-ruleset.xml (Critical Only)

```xml
<?xml version="1.0"?>
<ruleset name="Critical Rules"
    xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0
                        https://pmd.sourceforge.io/ruleset_2_0_0.xsd">

    <description>Critical rules only - no style/naming</description>

    <!-- Dead Code -->
    <rule ref="category/java/bestpractices.xml/UnusedPrivateField"/>
    <rule ref="category/java/bestpractices.xml/UnusedPrivateMethod"/>
    <rule ref="category/java/bestpractices.xml/UnusedLocalVariable"/>

    <!-- Empty Blocks -->
    <rule ref="category/java/errorprone.xml/EmptyCatchBlock"/>
    <rule ref="category/java/errorprone.xml/EmptyIfStmt"/>
    <rule ref="category/java/errorprone.xml/EmptyWhileStmt"/>
    <rule ref="category/java/errorprone.xml/EmptyTryBlock"/>
    <rule ref="category/java/errorprone.xml/EmptyFinallyBlock"/>

    <!-- Resource Leaks -->
    <rule ref="category/java/errorprone.xml/CloseResource"/>

    <!-- Logic Errors -->
    <rule ref="category/java/errorprone.xml/AvoidBranchingStatementAsLastInLoop"/>
    <rule ref="category/java/errorprone.xml/MisplacedNullCheck"/>

    <!-- Bad Practices -->
    <rule ref="category/java/bestpractices.xml/AvoidReassigningParameters"/>
</ruleset>
```

### Commands

```bash
mvn pmd:check    # Run PMD analysis
mvn pmd:pmd      # Generate report
```

---

## Error Prone (Compile-Time Checks)

### Maven Compiler Plugin Configuration

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.14.0</version>
    <configuration>
        <release>17</release>
        <compilerArgs>
            <arg>-XDcompilePolicy=simple</arg>
            <arg>-Xplugin:ErrorProne
                -XepDisableAllChecks
                -Xep:NullAway:ERROR
                -Xep:MissingOverride:ERROR
                -Xep:EqualsHashCode:ERROR
                -Xep:MustBeClosedChecker:ERROR
                -Xep:StreamResourceLeak:ERROR
            </arg>
        </compilerArgs>
        <annotationProcessorPaths>
            <path>
                <groupId>com.google.errorprone</groupId>
                <artifactId>error_prone_core</artifactId>
                <version>2.24.1</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

---

## SonarQube (Optional - CI/CD)

### Critical Quality Gates

```yaml
# sonar-project.properties
sonar.projectKey=my-project
sonar.sources=src/main/java
sonar.tests=src/test/java
sonar.java.binaries=target/classes

# Quality Gate thresholds
sonar.qualitygate.wait=true
```

### Focus Areas

| Metric | Threshold | Why |
|--------|-----------|-----|
| Bugs | 0 new | Real bugs |
| Vulnerabilities | 0 new | Security |
| Security Hotspots | Reviewed | Security |
| Code Coverage | >= 80% | Test quality |

Skip: Code Smells (too noisy), Technical Debt (subjective)

---

## Quick Reference Commands

```bash
# Full quality check
mvn clean verify spotless:check spotbugs:check pmd:check

# Auto-fix formatting
mvn spotless:apply

# Coverage report
mvn test jacoco:report
open target/site/jacoco/index.html

# SpotBugs GUI
mvn spotbugs:gui
```

---

## CI/CD Integration

```yaml
# GitHub Actions example
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Quality Checks
        run: |
          mvn verify \
            spotless:check \
            spotbugs:check \
            pmd:check \
            jacoco:check
```
