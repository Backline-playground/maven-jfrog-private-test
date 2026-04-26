# maven-jfrog-private-test

End-to-end test repo for **BKLN-2092** — validates Backline's `JFrogRegistryMavenPackageRegistryHandler` correctly authenticates against JFrog when a Maven project depends on a private artifact.

## Topology

```
this repo (pom.xml)
   └── com.backline.test:demo-lib:1.0.0          ← lives in JFrog: maven-test-local
          └── org.apache.logging.log4j:log4j-core:2.14.1   ← Log4Shell (CVE-2021-44228)
                                                              served via maven-test virtual
                                                              from the dummy-maven Central proxy
```

## JFrog setup (already provisioned)

| Repo | Type | Contents |
|------|------|----------|
| `maven-test-local` | local hosted | `com.backline.test:demo-lib:1.0.0` (the private artifact) |
| `dummy-maven` | remote | proxy of `repo1.maven.org/maven2` |
| `maven-test` | virtual | unions `maven-test-local` + `dummy-maven` |

Backline JFrog integration URL: `https://backlineai.jfrog.io/artifactory/maven-test/`

## Why this exercises the new handler

1. Trivy scan of `pom.xml` flags `log4j-core:2.14.1` (a transitive dep of `demo-lib`).
2. Backline SCA Planning needs to resolve the upgrade. To compute `demo-lib`'s effective POM, it has to fetch the POM from the customer's Maven registry.
3. That fetch runs in the runner sandbox. Without auth, JFrog rejects the request (`maven-test` has `forceMavenAuthentication: true`).
4. The runner calls `JFrogRegistryMavenPackageRegistryHandler.GetPackageRegistryInitCommands(...)`, which writes `$HOME/.m2/settings.xml` with a Bearer token and a mirror of `*` → `https://backlineai.jfrog.io/artifactory/maven-test/`.
5. `mvn dependency:tree` (or whatever planning runs) succeeds; without our handler this would 401.

## Files

- `pom.xml` — single dep on `com.backline.test:demo-lib:1.0.0`
- `src/main/java/.../App.java` — uses `Demo.greet()` from the private lib
- `trivy-report.json` — pre-generated finding for Log4Shell, suitable for ingestion as the SCA scanner result
