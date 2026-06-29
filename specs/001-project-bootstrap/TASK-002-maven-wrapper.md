# TASK-002 — Add Maven Wrapper

## Status

Planned

## Parent story

`US-001-project-bootstrap.md`

## Summary

Add the Maven Wrapper to the repository so that any developer or CI agent can build the project without requiring a global Maven installation. The wrapper pins the Maven version used by the project.

## Technical scope

This task includes:

- `.mvn/wrapper/maven-wrapper.properties` with a pinned Maven version.
- `mvnw` shell script (Linux/macOS).
- `mvnw.cmd` batch script (Windows).
- Verifying that `.gitignore` already excludes `.mvn/wrapper/maven-wrapper.zip` and `.mvn/timing.properties` (it does — confirmed in existing `.gitignore`).
- Verifying that `.mvn/wrapper/maven-wrapper.jar` is **not** excluded (it must be committed if distributed mode requires it, or omitted if the wrapper downloads it).

## Out of scope

This task does not include:

- Any changes to `pom.xml` or module structure.
- CI/CD pipeline usage of the wrapper.

## Implementation notes

- **Recommended approach**: generate the wrapper using the Maven Wrapper plugin from within the project root (requires a temporary Maven installation, or use the IntelliJ IDEA "Add Maven Wrapper" action):
  ```bash
  mvn wrapper:wrapper -Dmaven=3.9.x
  ```
- **Maven version to pin**: `3.9.x` (latest stable 3.x release at time of implementation).
- **Distribution URL**: use the official Apache Maven binary ZIP from `https://repo.maven.apache.org/maven2/`.
- The `.mvn/wrapper/maven-wrapper.jar` should be committed if the project uses the "jar" distribution type. If using "download-only" (no JAR), the wrapper scripts fetch Maven on first use.
- Prefer the **no-jar** approach (set `distributionType=bin` in `maven-wrapper.properties`) to avoid committing a binary JAR.

### maven-wrapper.properties reference

```properties
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.x/apache-maven-3.9.x-bin.zip
wrapperUrl=https://repo.maven.apache.org/maven2/org/apache/maven/wrapper/maven-wrapper/3.3.x/maven-wrapper-3.3.x.jar
distributionType=bin
```

## Expected files

```text
hermes-api-platform/
├── mvnw
├── mvnw.cmd
└── .mvn/
    └── wrapper/
        └── maven-wrapper.properties
```

## Acceptance criteria

- [ ] `mvnw` and `mvnw.cmd` exist at the repository root.
- [ ] `.mvn/wrapper/maven-wrapper.properties` contains a pinned `distributionUrl`.
- [ ] `.\mvnw.cmd --version` (Windows) or `./mvnw --version` (Linux/macOS) prints the Maven version without requiring a global Maven installation.
- [ ] `mvnw` has executable permissions on Linux/macOS (`chmod +x mvnw`).
- [ ] `.mvn/wrapper/maven-wrapper.zip` is not committed (confirmed by `.gitignore`).

## Validation

Windows:

```powershell
.\mvnw.cmd --version
```

Linux/macOS:

```bash
./mvnw --version
```

Expected output (example):

```text
Apache Maven 3.9.x (...)
Java version: 21.x.x, ...
```

## Commit suggestion

```text
chore: add Maven Wrapper (Maven 3.9.x)
```

## Notes

- `mvnw` must have Unix line endings (`LF`). If generated on Windows, verify with a tool like `dos2unix` or configure Git's `autocrlf` accordingly to avoid CI failures on Linux runners.
- The existing `.gitignore` already excludes `.mvn/timing.properties` and `.mvn/wrapper/maven-wrapper.zip`. No changes to `.gitignore` are needed for this task.
