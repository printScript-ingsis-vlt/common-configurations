# Common workflows

Este repositorio concentra los workflows reutilizables de GitHub Actions de los microservicios de Snippet Searcher. Su objetivo es que los repositorios de aplicación definan solo sus triggers y datos propios, mientras que la lógica repetida de validación, publicación y notificación viva en un único lugar.

Los workflows de este repositorio no se ejecutan solos: todos usan `workflow_call` y son invocados por un workflow pequeño en cada microservicio.

## Flujo general

```text
push o merge a dev/prod
        |
        v
     check.yml
  validación de código
        |
        v
    publish.yml
 construye y publica
 imagen Docker en GHCR
        |
        v
    notify.yml
 avisa a snippet-searcher-infra
        |
        v
snippet-searcher-infra
registra el digest y en las VMs
despliega el stack en Docker Swarm
```

Los pull requests ejecutan solamente la validación. La publicación de una imagen y la notificación a infraestructura suceden únicamente después de un `push` exitoso a `dev` o `prod`.

## Workflows disponibles

| Workflow | Responsabilidad | Se ejecuta después de |
|---|---|---|
| `check.yml` | Ejecutar validaciones Gradle o Node/Vite | Trigger del repositorio consumidor |
| `publish.yml` | Construir y publicar una imagen Docker inmutable en GHCR | `check.yml` exitoso |
| `notify.yml` | Enviar a infra el digest de la imagen publicada | `publish.yml` exitoso |

### `check.yml`

Recibe el input opcional `project_type`. Su valor por defecto es `gradle`, por lo que los microservicios existentes no requieren cambios.

Para `gradle`, prepara Java 21 con cache de Gradle y ejecuta:

```text
./gradlew check detekt test
```

Para `node`, prepara Node 20 con cache de npm y ejecuta:

```text
npm ci
→ npm run lint
→ npm run build
```

### `publish.yml`

Publica una imagen Docker en GitHub Container Registry (GHCR)

Recibe estos inputs:

| Input | Obligatorio | Default | Descripción |
|---|---:|---|---|
| `image_repository` | Sí | — | Repositorio GHCR completo de la imagen |
| `environment` | Sí | — | `dev` o `prod` |
| `docker_context` | No | `.` | Directorio usado como contexto de Docker |
| `dockerfile` | No | `./Dockerfile` | Ruta del Dockerfile |

Valida que:

- el ambiente sea exactamente `dev` o `prod`;
- la branch desde la que se invocó coincida con ese ambiente;
- la imagen sea una de las autorizadas;
- el contexto y el Dockerfile existan dentro del repositorio.

Las imágenes autorizadas son:

```text
ghcr.io/printscript-ingsis-vlt/snippet-service
ghcr.io/printscript-ingsis-vlt/permissions
ghcr.io/printscript-ingsis-vlt/print-script-engine
ghcr.io/printscript-ingsis-vlt/printscript-ui
```

La imagen se etiqueta con branch y SHA completo del commit:

```text
ghcr.io/printscript-ingsis-vlt/permissions:dev-<sha>
ghcr.io/printscript-ingsis-vlt/permissions:prod-<sha>
```

El workflow expone dos outputs:

| Output | Uso |
|---|---|
| `image` | Referencia inmutable con digest: `ghcr.io/...@sha256:...` |
| `tag` | Referencia con tag: `ghcr.io/...:dev-<sha>` |

Infra debe usar siempre el output `image`, porque el digest identifica de forma inmutable el contenido publicado.

```yaml
secrets:
  gpr_token: ${{ secrets.GPR_TOKEN }}
```

El Dockerfile recibe ese valor como secreto de BuildKit, por lo que no queda guardado dentro de la imagen.

### `notify.yml`

Notifica al repositorio central de infraestructura que una imagen fue publicada

Inputs:

| Input | Obligatorio | Default | Descripción |
|---|---:|---|---|
| `service` | Sí | — | `snippet-service`, `permissions`, `print-script-engine` o `printscript-ui` |
| `environment` | Sí | — | `dev` o `prod` |
| `image` | Sí | — | Output `image` de `publish.yml` |
| `infra_repository` | No | `printscript-ingsis-vlt/snippet-searcher-infra` | Repo receptor |

El workflow valida que el servicio, el ambiente y la branch sean válidos. También exige que la referencia de imagen corresponda al servicio indicado y tenga un digest `@sha256:`.

Después envía este evento a GitHub:

```json
{
  "event_type": "image-published",
  "client_payload": {
    "schema_version": 1,
    "service": "permissions",
    "environment": "dev",
    "image": "ghcr.io/printscript-ingsis-vlt/permissions@sha256:...",
    "commit": "<sha>",
    "source_repository": "printscript-ingsis-vlt/permissions",
    "source_run_id": "<run-id>"
  }
}
```


## Ejemplo de uso desde un microservicio

Cada microservicio debe conservar un workflow propio, por ejemplo `.github/workflows/pipeline.yml`. Ese workflow define branches y encadena los workflows comunes:

```yaml
name: CI, publish image and notify infrastructure

on:
  pull_request:
    branches: [dev, prod]
  push:
    branches: [dev, prod]

permissions:
  contents: read
  packages: write

jobs:
  check:
    uses: printScript-ingsis-vlt/common-configurations/.github/workflows/check.yml@main

  publish:
    needs: check
    if: github.event_name == 'push'
    uses: printScript-ingsis-vlt/common-configurations/.github/workflows/publish.yml@main
    with:
      image_repository: ghcr.io/printscript-ingsis-vlt/permissions
      environment: ${{ github.ref_name }}

  notify:
    needs: publish
    if: github.event_name == 'push'
    uses: printScript-ingsis-vlt/common-configurations/.github/workflows/notify.yml@main
    with:
      service: permissions
      environment: ${{ github.ref_name }}
      image: ${{ needs.publish.outputs.image }}
    secrets:
      INFRA_DISPATCH_TOKEN: ${{ secrets.INFRA_DISPATCH_TOKEN }}
```
