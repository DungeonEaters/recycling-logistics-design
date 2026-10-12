# Estrategia de CI/CD — Logística y Recolección de Plásticos (MVP)

Define cómo se construyen, versionan, publican y despliegan las imágenes del proyecto tras el paso a polyrepo. Complementa [architecture.md](../architecture/architecture.md) (ADR-013/014/015) y [decisions.md](../architecture/decisions.md).

## 1. Repositorios y responsabilidad

| Repositorio | Responsabilidad en CI/CD |
|---|---|
| [recycling-backend](https://github.com/DungeonEaters/recycling-backend) | Test (pytest), build y publicación de imagen `api` en GHCR |
| [recycling-web](https://github.com/DungeonEaters/recycling-web) | Lint/build (npm), publicación de imagen `web` (Nginx) en GHCR |
| [recycling-mobile](https://github.com/DungeonEaters/recycling-mobile) | Test + build (npm), publicación de imagen `mobile` (Nginx) en GHCR |
| [recycling-infra](https://github.com/DungeonEaters/recycling-infra) | Terraform (provisión Oracle) + workflow de despliegue (pull de imágenes ya publicadas y `docker compose up`) |

Cada repo de aplicación es dueño de su propio ciclo de versiones; no existe un tag o release único de plataforma.

## 2. Versionado: SemVer estricto con flag de canal

Un único workflow manual (`release.yml`, `workflow_dispatch`) construye ambos canales; el canal se elige con el input `channel` (`test` | `release`), no por trigger automático. Ambos siguen [SemVer 2.0](https://semver.org/) estricto.

| Canal | Trigger | Formato de versión (validado por regex) | Tag de imagen GHCR | Tag de Git | GitHub Release |
|---|---|---|---|---|---|
| **test** | Manual (`workflow_dispatch`, input `channel=test`) | `X.Y.Z` o `X.Y.Z-rc.N` / `-beta.N` / `-alpha.N` | `test-X.Y.Z[-suffix]` | No se crea | No |
| **release** | Manual (`workflow_dispatch`, input `channel=release`) | `X.Y.Z` estricto, **sin sufijo** (rechazado si lleva `-rc`/`-beta`/etc.) | `release-vX.Y.Z` + `latest` | Sí — el propio workflow crea y pushea `vX.Y.Z` | Sí — se crea automáticamente con notas generadas |

En ambos casos se publica además un tag inmutable `sha-<commit>` para trazabilidad exacta.

**Por qué este esquema:** un solo workflow con flag evita duplicar lógica de build entre dos triggers distintos y deja explícito, en cada ejecución manual, qué canal se está construyendo. El job `release-channel` valida que la versión no lleve sufijo de pre-release antes de crear el tag de Git y el GitHub Release — así el versionamiento estricto SemVer queda garantizado incluso si alguien se equivoca al tipear el input.

## 3. Flujo completo

```
PR  ──► ci.yml (test/build, sin publicar) ──► merge a main
                                                         │
                         ┌───────────────────────────────┴───────────────────────────────┐
                         │                                                               │
        workflow_dispatch "release.yml"                                   workflow_dispatch "release.yml"
        input: channel=test, version=X.Y.Z-rc.N, ref=rama/commit          input: channel=release, version=X.Y.Z, ref=rama/commit
                         │                                                               │
                build + push GHCR:                                         valida SemVer estricto (sin sufijo)
                test-X.Y.Z-rc.N, sha-<commit>                                             │
                                                                              crea y pushea tag git vX.Y.Z
                                                                                           │
                                                                              build + push GHCR:
                                                                              release-vX.Y.Z, latest, sha-<commit>
                                                                                           │
                                                                              crea GitHub Release vX.Y.Z
                         │                                                               │
                         └───────────────────────────────┬───────────────────────────────┘
                                                          ▼
                                 recycling-infra: deploy.yml (workflow_dispatch manual)
                                 input: environment=test|production, component=all|api|web|mobile,
                                        image_tag=test-X.Y.Z-rc.N | release-vX.Y.Z
                                                          ▼
                                 SSH a la VM Oracle → docker compose -f docker-compose.prod.yml pull/up
```

El despliegue **siempre** es manual (`workflow_dispatch` en `recycling-infra`) y recibe un tag de imagen ya publicado — nunca construye ni redeploya automáticamente al publicar una imagen. Esto da control explícito sobre qué versión corre en cada ambiente.

## 4. Workflows por repo de aplicación

Cada uno de `recycling-backend`, `recycling-web`, `recycling-mobile` implementa el mismo patrón:

- **`.github/workflows/ci.yml`**: en `pull_request` y `push` a `main`. Jobs: test/lint (pytest o `npm test`/`npm run build` según el repo). No publica nada; es el status check requerido para mergear a `main` (nombre exacto del check requerido = nombre del job, p. ej. `Backend (pytest)`, `Web (build)`, `Mobile (test + build)`).
- **`.github/workflows/release.yml`**: `workflow_dispatch` con inputs `channel` (`test`|`release`), `version` (SemVer) y `ref` (rama/commit a construir, default `main`). Dos jobs mutuamente excluyentes vía `if: inputs.channel == 'test' | 'release'`:
  - `test-channel`: valida SemVer (con o sin sufijo), build multi-stage
    `linux/amd64` + `linux/arm64` (QEMU + Buildx), push a GHCR como
    `test-<version>` + `sha-<commit>`. No toca Git.
  - `release-channel`: valida SemVer estricto (sin sufijo), crea y pushea el tag `vX.Y.Z`, build + push a GHCR como `release-v<version>` + `latest` + `sha-<commit>`, crea el GitHub Release con notas autogeneradas.
  - Login a GHCR con `GITHUB_TOKEN` (`permissions: contents: write, packages: write` — `contents: write` es necesario para que `release-channel` pueda pushear el tag y crear el Release).

La imagen nueva del backend es una manifest list multi-arquitectura: OCI A1 descarga
`arm64` y una VM AMD temporal de test descarga `amd64` para el mismo tag inmutable.
Los tags de backend publicados antes de este cambio permanecen solo ARM64 y no sirven
para la VM AMD.

Imagen resultante: `ghcr.io/dungeoneaters/recycling-backend`, `ghcr.io/dungeoneaters/recycling-web`, `ghcr.io/dungeoneaters/recycling-mobile` (ya reflejado en `recycling-infra/.env.production.example`).

## 5. Pull Request template

Los cuatro repos (`recycling-backend`, `recycling-web`, `recycling-mobile`, `recycling-infra`) incluyen `.github/pull_request_template.md` con: descripción, tipo de cambio, cómo se probó y checklist (CI verde, impacto en contrato de API, documentación actualizada, sin secretos en el diff).

## 6. Branch protection y status checks

`main` en los 4 repos ya exige PR + 1 aprobación (ver ADR de protección de ramas en [decisions.md](../architecture/decisions.md)). El job `ci.yml` de cada repo de aplicación está configurado como **required status check** (por nombre de job, no de workflow) en `recycling-backend`, `recycling-web` y `recycling-mobile` (no aplica aún a `recycling-infra`, que no tiene CI de aplicación, solo Terraform).

## 7. Relación con Terraform y despliegue a Oracle

- Los workflows `terraform-plan.yml`/`terraform-apply.yml` de `recycling-infra` provisionan la VM; no están acoplados al versionado de imágenes de aplicación.
- El workflow de despliegue (`deploy.yml`, ver sección de infraestructura en [oracle-cloud-provisioning.md](../infrastructure/oracle-cloud-provisioning.md)) es el único punto donde un `image_tag` de GHCR llega a la VM. Usa `API_IMAGE`/`WEB_IMAGE`/`MOBILE_IMAGE` (definidos en `.env` de la VM, uno por repo) en vez de un único `IMAGE_REPOSITORY` como en el monorepo legado.
- Credenciales OCI necesarias para Terraform: ver [oracle-credentials-setup.md](../infrastructure/oracle-credentials-setup.md).

## 8. Evidencia de ejecución

Ejecución real del pipeline de `release.yml` (canal test, `recycling-backend`), validada end-to-end antes de mergear el PR de la feature:

- Run: https://github.com/DungeonEaters/recycling-backend/actions/runs/38099749354 (`workflow_dispatch`, channel=test, version=0.1.0-rc.1)
- Resultado: `success`
- Imagen publicada: `ghcr.io/dungeoneaters/recycling-backend:test-0.1.0-rc.1`
- Tag adicional: `ghcr.io/dungeoneaters/recycling-backend:sha-9baf631`
- Digest: `sha256:f316b364debdf5885f3584be02bfa838b86335e8697c74f57a486b3f44131ece`

Ver ADR-018 y ADR-020 en [decisions.md](../architecture/decisions.md) para el contexto completo, incluyendo el incidente de bypass de branch protection detectado y revertido en `recycling-backend` (PR de revert: https://github.com/DungeonEaters/recycling-backend/pull/2).
