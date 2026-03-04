# ci-templates

Repositorio de plantillas CI/CD reutilizables para microservicios Spring Boot con GitHub Actions.

---

## 📑 Índice

- [Descripción General](#descripción-general)
- [Arquitectura del Pipeline](#arquitectura-del-pipeline)
- [Workflows Disponibles](#workflows-disponibles)
  - [PR Validation Pipeline](#1-pr-validation-pipeline-pr-ms-spring_bootyml)
  - [Build & Test](#2-build--test-_build-test-javayml)
  - [Docker Scan](#3-docker-scan-_docker-scanyml)
  - [PR Report](#4-pr-report-_pr-reportyml)
  - [Merge and Tag](#5-merge-and-tag-merge-and-tagyml)
  - [Reverse Merge](#6-reverse-merge-reverse-mergeyml)
- [Actions Reutilizables](#actions-reutilizables)
  - [gradle-version](#gradle-version)
  - [compute-next-tag](#compute-next-tag)
- [Cómo Usar en tu Proyecto](#cómo-usar-en-tu-proyecto)
- [Mensajes y Salidas del Pipeline](#mensajes-y-salidas-del-pipeline)

---

## Descripción General

Este repositorio contiene workflows y actions de GitHub Actions diseñados para estandarizar el proceso de CI/CD de microservicios basados en **Spring Boot + Gradle**. El pipeline se ejecuta automáticamente en cada **Pull Request** hacia `develop` o `release`.

### Flujo completo

```
PR abierto → Validación PR → Build & Test → Docker Scan → Reporte PR → (merge) → Tag automático
```

---

## Arquitectura del Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    PR Validation Pipeline                        │
│                  (pr-ms-spring_boot.yml)                        │
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ validate_pr  │───▶│ build_test   │───▶│ docker_scan  │       │
│  │              │    │              │    │              │       │
│  │ • Versión    │    │ • Compilar   │    │ • Build img  │       │
│  │ • Commits    │    │ • Tests      │    │ • Trivy scan │       │
│  │ • Tag calc   │    │ • Quality    │    │ • Vuln check │       │
│  └──────────────┘    │ • Coverage   │    └──────┬───────┘       │
│                      │ • JAR        │           │               │
│                      └──────────────┘           │               │
│                                                  │               │
│                      ┌──────────────────────────┘               │
│                      ▼                                           │
│               ┌──────────────┐                                   │
│               │ report_pr    │                                   │
│               │ _comment     │                                   │
│               │              │                                   │
│               │ • Resumen PR │                                   │
│               └──────────────┘                                   │
└─────────────────────────────────────────────────────────────────┘

              Tras merge del PR:
┌──────────────────────┐
│   merge_and_tag      │
│ • Crea tag RELEASE   │
│ • Borra rama feature │
└──────────────────────┘
```

---

## Workflows Disponibles

### 1. PR Validation Pipeline (`pr-ms-spring_boot.yml`)

**Trigger:** `pull_request` hacia `develop` o `release`
**Tipo:** Workflow principal orquestador

Este es el workflow principal que orquesta todo el pipeline de validación de un PR. Ejecuta los siguientes jobs en secuencia:

| Job | Descripción | Dependencia |
|-----|-------------|-------------|
| `validate_pr` | Valida versión Gradle, tag esperado y cantidad de commits | Ninguna |
| `build_test` | Compila, ejecuta tests, calidad y cobertura | `validate_pr` |
| `docker_scan` | Construye imagen Docker y escanea vulnerabilidades | `build_test` |
| `report_pr_comment` | Publica un comentario resumen en el PR | Todos (always) |

**Inputs opcionales:**

| Input | Default | Descripción |
|-------|---------|-------------|
| `java_version` | `"21"` | Versión de Java a utilizar |
| `promote_branch` | `"release"` | Rama de promoción (no usado en PR) |

#### Job: `validate_pr`

Realiza tres validaciones críticas:

1. **Lectura de versión Gradle** — Ejecuta `./gradlew -q printVersion` para obtener la versión actual del proyecto.
2. **Cálculo de tag esperado** — Busca el último tag con patrón `X.Y.Z-RELEASE` e incrementa el patch (`+1`).
3. **Validación de versión** — La versión en `build.gradle` debe coincidir exactamente con el tag calculado.
4. **Validación de commits** — El PR debe contener exactamente **1 commit**.

---

### 2. Build & Test (`_build-test-java.yml`)

**Tipo:** Workflow reutilizable (`workflow_call`)

Se encarga de compilar el proyecto y ejecutar todas las categorías de tests:

| Paso | Descripción |
|------|-------------|
| Setup Java | Configura JDK Temurin + cache de Gradle |
| Run Tests | `./gradlew unitTest componentTest integrationTest blackBoxTest` |
| Print Test Summary | Resumen de tests por categoría (Unit, Component, Integration, BlackBox) |
| Quality Checks | Ejecuta `check` + `jacocoTestReport` (Checkstyle, PMD, JaCoCo) |
| Build JAR | `./gradlew clean bootJar` |
| Upload Artifacts | Sube JAR y reportes como artefactos de GitHub |

**Output:** `version` — La versión del proyecto leída de Gradle.

---

### 3. Docker Scan (`_docker-scan.yml`)

**Tipo:** Workflow reutilizable (`workflow_call`)

Construye la imagen Docker del microservicio y la escanea en busca de vulnerabilidades:

| Paso | Descripción |
|------|-------------|
| Download JAR | Descarga el artefacto JAR del job anterior |
| Build Docker | Construye la imagen con el tag de versión |
| Trivy Scan | Escanea vulnerabilidades (OS + libraries) en formato SARIF |
| Upload SARIF | Sube resultados a GitHub Security / Code Scanning |
| Vulnerability Gate | **Bloquea** el merge si hay vulnerabilidades HIGH o CRITICAL |

**Inputs:**

| Input | Default | Descripción |
|-------|---------|-------------|
| `image_name` | `"ms-factura"` | Nombre de la imagen Docker |
| `version` | *(requerido)* | Tag de versión para la imagen |

---

### 4. PR Report (`_pr-report.yml`)

**Tipo:** Workflow reutilizable (`workflow_call`)

Genera un comentario automático en el PR con el resumen de todos los jobs:

- Estado general del pipeline (✅ EXITOSO / ❌ FALLÓ)
- Detalles de cada job con sus evaluaciones
- Versión de Gradle y tag esperado
- Cantidad de commits

---

### 5. Merge and Tag (`merge-and-tag.yml`)

**Trigger:** `pull_request` cerrado (merged) hacia `develop` o `release`

Se ejecuta **automáticamente una vez que el PR es mergeado**:

1. Calcula el tag esperado (igual que en la validación).
2. Crea el tag `X.Y.Z-RELEASE` y lo pushea al repositorio.
3. Elimina la rama feature ya mergeada (protege `main`, `develop`, `release`).

---

### 6. Reverse Merge (`reverse-merge.yml`)

**Trigger:** `workflow_dispatch` (ejecución manual)

Permite eliminar un tag en caso de error o rollback:

| Input | Descripción |
|-------|-------------|
| `tag_to_reverse` | Tag a eliminar (ej: `1.0.5-RELEASE`) |

---

## Actions Reutilizables

### `gradle-version`

Lee la versión del proyecto desde Gradle.

```yaml
- uses: KelvinVargasUG/ci-templates/.github/actions/gradle-version@main
```

**Output:** `version` — Versión del proyecto (ej: `1.0.5-RELEASE`).

### `compute-next-tag`

Calcula el siguiente tag semántico basado en el último tag existente.

```yaml
- uses: KelvinVargasUG/ci-templates/.github/actions/compute-next-tag@main
```

**Inputs opcionales:**

| Input | Default | Descripción |
|-------|---------|-------------|
| `pattern` | `*.*.*-RELEASE` | Patrón de tags a buscar |
| `suffix` | `-RELEASE` | Sufijo del tag |

**Outputs:**

| Output | Ejemplo | Descripción |
|--------|---------|-------------|
| `last_tag` | `1.0.4-RELEASE` | Último tag encontrado |
| `expected_tag` | `1.0.5-RELEASE` | Siguiente tag calculado |

---

## Cómo Usar en tu Proyecto

Crea un archivo `.github/workflows/CI_CD.yml` en tu microservicio:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ "develop" ]
  pull_request:

jobs:
  pipeline:
    permissions:
      contents: read
      security-events: write
      pull-requests: write
    uses: KelvinVargasUG/ci-templates/.github/workflows/pr-ms-spring_boot.yml@main
    with:
      java_version: "21"
      promote_branch: "release"

  call-merge:
    uses: KelvinVargasUG/ci-templates/.github/workflows/merge-and-tag.yml@main
    with:
      branch: "develop"
```

---

## Mensajes y Salidas del Pipeline

A continuación se muestran los mensajes que el pipeline genera en cada escenario para identificar rápidamente el resultado.

### ✅ Validación exitosa de versión

```
Current version (Gradle): 1.0.5-RELEASE
Last tag found: 1.0.4-RELEASE
Expected calculated tag: 1.0.5-RELEASE
✅ OK: Version matches the expected tag.
```

### ❌ Versión no coincide con el tag esperado

```
Current version (Gradle): 1.0.3-RELEASE
Last tag found: 1.0.4-RELEASE
Expected calculated tag: 1.0.5-RELEASE
❌ Version not incremented correctly.
   - Current version: 1.0.3-RELEASE
   - Expected tag   : 1.0.5-RELEASE
Update the version in Gradle to match the expected tag.
```

### ✅ Commits válidos en el PR

```
Base SHA: abc123
Head SHA: def456
Commits in PR: 1
✅ PR has exactly 1 commit.
```

### ❌ Demasiados commits en el PR

```
Base SHA: abc123
Head SHA: def456
Commits in PR: 3
❌ PR must have exactly 1 commit. It has 3.
```

### ✅ Tests exitosos (resumen)

```
========================================
 ✅ UNIT TESTS
========================================
  Total   : 42
  Passed  : 42
  Failures: 0
  Errors  : 0
  Skipped : 0

========================================
 ✅ COMPONENT TESTS
========================================
  Total   : 12
  Passed  : 12
  Failures: 0
  Errors  : 0
  Skipped : 0

========================================
 ✅ OVERALL: 54/54 passed
========================================
```

### ❌ Tests fallidos

```
========================================
 ❌ UNIT TESTS
========================================
  Total   : 42
  Passed  : 40
  Failures: 2
  Errors  : 0
  Skipped : 0

========================================
 ❌ OVERALL: 40/42 passed
========================================
❌ Test failures or errors detected.
```

### 📈 Cobertura de código

```
📈 JaCoCo LINE coverage: 92.35% (covered=487, missed=40)
```

### ✅ Docker scan sin vulnerabilidades

```
✅ JAR artifact downloaded: build/libs/ms-factura-1.0.5-RELEASE.jar
✅ Docker image built: ms-factura:1.0.5-RELEASE
CRITICAL=0 HIGH=0 has_vulns=false

## 🐳 Docker Scan Results
Image: ms-factura:1.0.5-RELEASE
- CRITICAL: 0
- HIGH: 0
Gate Status: ✅ PASSED
```

### ❌ Docker scan con vulnerabilidades

```
CRITICAL=2 HIGH=5 has_vulns=true

## 🐳 Docker Scan Results
Image: ms-factura:1.0.5-RELEASE
- CRITICAL: 2
- HIGH: 5
Gate Status: ❌ BLOCKED

❌ HIGH/CRITICAL vulnerabilities detected. Blocking merge.
```

Además se publica un comentario en el PR:

```
## 🔐 Trivy: HIGH/CRITICAL Vulnerabilities Detected

📦 Image: `ms-factura:1.0.5-RELEASE`
- CRITICAL: **2**
- HIGH: **5**

➡️ View findings in **Security → Code scanning**: <link>
```

### 📋 Comentario de Reporte en el PR

Cuando todos los jobs finalizan, se publica automáticamente un comentario resumen:

#### Todos los jobs exitosos:

```
## ✅ Reporte de Pipeline

### Estado General: ✅ EXITOSO

---

### 📋 Job: Validación de PR
Estado: ✅ success
- ✔️ Versión en Gradle: 1.0.5-RELEASE
- ✔️ Tag esperado: 1.0.5-RELEASE
- ✔️ Cantidad de commits: 1

---

### 🏗️ Job: Build & Test
Estado: ✅ success
- ✔️ Compilación de código con Gradle
- ✔️ Ejecución de pruebas unitarias
- ✔️ Cobertura de código (JaCoCo - mínimo 90%)

---

### 🔐 Job: Escaneo de Docker
Estado: ✅ success
- ✔️ Construcción de imagen Docker
- ✔️ Escaneo de vulnerabilidades con Trivy
- ✔️ Validación de vulnerabilidades críticas
```

#### Algún job falló:

```
## ❌ Reporte de Pipeline

### Estado General: ❌ FALLÓ

---

### 📋 Job: Validación de PR
Estado: ✅ success

---

### 🏗️ Job: Build & Test
Estado: ❌ failure

---

### 🔐 Job: Escaneo de Docker
Estado: ⏭️ skipped
```

### 🏷️ Tag creado tras merge

```
Last tag: 1.0.4-RELEASE
Expected next tag: 1.0.5-RELEASE
Creando tag 1.0.5-RELEASE y subiéndolo...
Deleting branch feature/my-feature
```

### Checkstyle / PMD (si hay violaciones)

```
=== CHECKSTYLE VIOLATIONS ===
  [warning] src/main/java/com/example/Service.java:25:5 — Missing Javadoc (...)
Total Checkstyle violations: 1

=== PMD VIOLATIONS ===
  [P3] src/main/java/com/example/Service.java:30-35 [UnusedPrivateMethod] ...
Total PMD violations: 1
```

---

## Requisitos del Proyecto

Para que el pipeline funcione correctamente, el microservicio debe cumplir:

1. **Gradle** — Tener una tarea `printVersion` que imprima la versión del proyecto.
2. **Tests** — Tareas Gradle: `unitTest`, `componentTest`, `integrationTest`, `blackBoxTest`.
3. **Quality** — Configuración de Checkstyle, PMD y JaCoCo integrada en el `build.gradle`.
4. **Docker** — Un `Dockerfile` válido en la raíz del proyecto.
5. **Versionamiento** — Seguir el formato `X.Y.Z-RELEASE` e incrementar el patch en cada PR.
6. **Commits** — Cada PR debe contener exactamente **1 commit** (usar squash si es necesario).