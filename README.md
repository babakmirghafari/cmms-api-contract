# cmms-api-contract

OpenAPI 3.1 contract for the CMMS (Computerized Maintenance Management System) — single source of truth for all API interfaces between the Angular 21 frontend and Spring Boot 4 backend.

---

## How It Works

```
┌──────────────────────────────────────────────────────────────────┐
│                    cmms-api-contract                              │
│                                                                   │
│  OpenAPI 3.1 spec  ──► openapi-generator  ──► Java artefacts     │
│  api.yaml +             Maven plugin           HealthApi          │
│  schemas/*.yaml                                HealthApiDelegate  │
│                         mvn deploy             HealthApiController│
│                             │                  *Dto models        │
│                             ▼                                     │
│                     GitHub Packages                               │
│                      (Maven registry)                             │
└─────────────────────────────┬─────────────────────────────────────┘
                              │  imported as Maven dependency
              ┌───────────────┴──────────────┐
              ▼                              ▼
   cmms-backend                    cmms-frontend
   Spring Boot 4                   Angular 21
                                   openapi-generator-cli reads api.yaml
   Implements:                     Generates:
   HealthApiDelegate               TypeScript services + models
   (business logic only)           (no manual HTTP calls)
```

---

## Project Structure

```
src/main/resources/openapi/
├── api.yaml                  # Entry point — info, servers, security, tags, paths, component refs
└── schemas/
    └── common.yaml           # ErrorResponseDto, PagedResponseDto, AuditInfoDto
```

---

## Defining the API

### Adding an Endpoint

1. Add the path to `api.yaml` under `paths:`.
2. If the resource has 3+ schemas or any schema has 8+ properties, create `schemas/{resource}.yaml`.
3. Aggregate all schema `$ref`s in `api.yaml` under `components.schemas`.
4. Run `mvn clean install` to verify generation succeeds.
5. Open a PR — `pr-validation.yml` checks for breaking changes via oasdiff.

### Naming Conventions

| Schema type          | Pattern                        | Example                     |
|----------------------|--------------------------------|-----------------------------|
| Read/response model  | `{Resource}Dto`                | `AssetDto`                  |
| Create request body  | `Create{Resource}RequestDto`   | `CreateAssetRequestDto`     |
| Update request body  | `Update{Resource}RequestDto`   | `UpdateAssetRequestDto`     |
| Embedded object      | `{Concept}Dto`                 | `LocationDto`               |
| Shared cross-cutting | Descriptive + `Dto`            | `ErrorResponseDto`          |

All schema names **must** end in `Dto`. No exceptions.

### Schema Split Threshold

| Condition                                              | Action                      |
|--------------------------------------------------------|-----------------------------|
| Cross-cutting shared type                              | `schemas/common.yaml`       |
| Resource has 3+ schemas                                | `schemas/{resource}.yaml`   |
| Any single schema has 8+ properties                    | `schemas/{resource}.yaml`   |
| 1–2 schemas with ≤ 7 properties each                   | Inline in `api.yaml`        |

---

## Building Locally

**Prerequisites:** Java 25, Maven 3.9+

```bash
mvn clean install
```

Generated sources appear in `target/generated-sources/openapi/`.

**Validate the spec with oasdiff:**
```bash
oasdiff breaking src/main/resources/openapi/api.yaml src/main/resources/openapi/api.yaml
```

---

## Publishing to GitHub Packages

### One-time `~/.m2/settings.xml` Setup

```xml
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>YOUR_GITHUB_USERNAME</username>
      <password>YOUR_GITHUB_TOKEN</password>
    </server>
  </servers>
</settings>
```

### Manual Publish

```bash
mvn clean deploy
```

### Release Tagging

```bash
git tag v1.0.0
git push origin v1.0.0
```

CI publishes automatically on every push to `main`.

---

## Consuming in Spring Boot

### Maven Dependency

```xml
<dependency>
  <groupId>com.cmms</groupId>
  <artifactId>cmms-api-contract</artifactId>
  <version>1.0.0</version>
</dependency>
```

### `settings.xml` for GitHub Packages

```xml
<server>
  <id>github</id>
  <username>YOUR_GITHUB_USERNAME</username>
  <password>YOUR_GITHUB_TOKEN</password>
</server>
```

### Repository Declaration in Consumer's `pom.xml`

```xml
<repositories>
  <repository>
    <id>github</id>
    <url>https://maven.pkg.github.com/babakmirghafari/cmms-api-contract</url>
  </repository>
</repositories>
```

### Delegation Pattern

The contract JAR ships three generated classes per resource tag:

- `HealthApi` — `@RequestMapping` interface
- `HealthApiDelegate` — pure business method interface (you implement this)
- `HealthApiController` — `@Controller` that wires the two (auto-scanned by Spring)

Spring Boot projects implement `HealthApiDelegate` only:

```java
// Your handler — implements the generated delegate interface
// No @RestController, @RequestMapping, or @PathVariable needed here

@Service
@RequiredArgsConstructor
public class HealthHandler implements HealthApiDelegate {

    @Override
    public ResponseEntity<HealthResponseDto> getHealth() {
        HealthResponseDto response = new HealthResponseDto();
        response.setStatus(HealthResponseDto.StatusEnum.UP);
        response.setVersion("1.0.0");
        response.setTimestamp(OffsetDateTime.now());
        return ResponseEntity.ok(response);
    }
}
```

**Root package requirement:** Your `@SpringBootApplication` must scan the contract package:

```java
@SpringBootApplication(scanBasePackages = "com.cmms")
public class CmmsApplication { ... }
```

---

## Consuming in Angular (npm)

### Install

```bash
npm install @babakmirghafari/cmms-api-client@1.0.0
```

### `.npmrc` Setup

```
@babakmirghafari:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

### Usage Example

```typescript
import { HealthService } from '@babakmirghafari/cmms-api-client';
import { BASE_PATH } from '@babakmirghafari/cmms-api-client';
import { provideHttpClient } from '@angular/common/http';

// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    { provide: BASE_PATH, useValue: 'http://localhost:8080/cmms/v1' }
  ]
};

// component
@Component({ ... })
export class HealthComponent {
  constructor(private healthService: HealthService) {}

  checkHealth() {
    this.healthService.getHealth().subscribe(response => {
      console.log('Status:', response.status);
    });
  }
}
```

---

## Versioning

| Change type                                   | Version bump     |
|-----------------------------------------------|------------------|
| Only additions (new endpoints, optional fields) | Patch: `1.0.0 → 1.0.1` |
| New resources or operations (additive)         | Minor: `1.0.0 → 1.1.0` |
| Removed/renamed fields, changed types          | Major: `1.0.0 → 2.0.0` |

Maven and npm versions are kept in lockstep. The npm job reads the version from `pom.xml` at runtime.

---

## Breaking Change Detection

`pr-validation.yml` runs `oasdiff` on every PR against `origin/main`. Breaking changes are blocked.

Examples of breaking changes detected:
- Removing an endpoint or operation
- Removing a required field
- Renaming a field
- Changing a field type
- Removing an enum value

---

## CI/CD Pipeline

```
push to main
    │
    ├── publish-maven job
    │       mvn clean verify
    │       mvn deploy
    │       → com.cmms:cmms-api-contract:1.0.0
    │         at maven.pkg.github.com/babakmirghafari/cmms-api-contract
    │
    └── publish-npm job
            openapi-generator-cli (typescript-angular)
            npm run build
            npm publish
            → @babakmirghafari/cmms-api-client@1.0.0
              at npm.pkg.github.com
```

Both jobs run in parallel. Both must be green before downstream scaffolds proceed.

---

## Published Artefacts

| Type  | Coordinate                                   | Registry URL                                                                 |
|-------|----------------------------------------------|------------------------------------------------------------------------------|
| Maven | `com.cmms:cmms-api-contract:1.0.1`           | https://maven.pkg.github.com/babakmirghafari/cmms-api-contract               |
| npm   | `@babakmirghafari/cmms-api-client@1.0.1`     | https://github.com/babakmirghafari/cmms-api-contract/packages                |
