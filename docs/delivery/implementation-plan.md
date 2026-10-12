# Plan de Implementación — Logística y Recolección de Plásticos (MVP)

Repositorio: [JulioRom/recycling-logistics](https://github.com/JulioRom/recycling-logistics). Duración estimada: **10–12 semanas**, equipo de 2 (1 Full-Stack, 1 Frontend). Diseño en [architecture.md](../architecture/architecture.md); tareas en [backlog.md](backlog.md).

## Estado de fases
| Fase | Semanas | Estado |
|---|---|---|
| 1. Setup y Modelado | 1–2 | Hecho (2026-10-05) |
| 2. Motor Backend y Rutas | 3–5 | Hecho en local; falta validar Google Routes con clave real |
| 3. Panel Web B2B | 6–8 | Hecho en local |
| 4. App Móvil Híbrida | 9–11 | UI y offline hechos en navegador; falta empaquetado nativo con Capacitor |
| 5. Pruebas QA y Lanzamiento | 12 | Pendiente |

## Fase 1 — Setup y Modelado (Semanas 1–2)
- Modelado de entidades y diagrama de BD.
- Repositorios configurados.
- Docker Compose con FastAPI + PostgreSQL.
- **Entregable:** entorno local reproducible y esquema inicial.

## Fase 2 — Motor Backend y Rutas (Semanas 3–5)
- Endpoints de usuarios/auth, clientes, puntos, camiones, rutas, pesajes.
- Integración con Google Maps Routes API (`optimizeWaypointOrder`).
- **Entregable:** API operativa con integración de rutas validada.

## Fase 3 — Panel Web B2B (Semanas 6–8)
- Gestión comercial (CRUD con GPS), asignación de rutas, dashboard, exportación CSV.
- **Entregable:** frontend consumiendo la API; monitoreo y generación de rutas.

## Fase 4 — App Móvil Híbrida (Semanas 9–11)
- UI Ionic, login, ruta activa, pesaje, deep link a Waze/Maps.
- Empaquetado Capacitor (iOS/Android) y almacenamiento offline con sync.
- **Entregable:** app instalable con flujo completo y offline.

## Fase 5 — QA y Lanzamiento (Semana 12)
- Pruebas físicas sin señal (subterráneos), validación de rutas sugeridas.
- QA final y despliegue a producción (VPS + CDN).
- **Entregable:** MVP en producción.

## Infraestructura y presupuesto mensual
| Ítem | Opción | Costo |
|---|---|---|
| Backend, BD y frontends | Oracle Cloud Always Free ARM64 con Docker Compose | $0 (dentro de cuota) |
| Reverse proxy y TLS | Caddy + Let's Encrypt | $0 |
| Mapas y rutas | Google Maps Routes API (crédito $200/mes) | $0 |
| Dominio y correo | Namecheap / Google Workspace | ~$15 USD |
| **Total** | | **< ~$30 USD** |

Alternativas evaluadas: Render/Railway ($0–10), GCP Cloud Run (barato) pero Cloud SQL ~$10–25/mes.

## Bloqueadores y riesgos
Ver sección 7 de [architecture.md](../architecture/architecture.md) y [backlog.md](backlog.md).

## Estado de la implementación (2026-10-05)
- **Hecho:** API FastAPI + PostgreSQL con auth/roles, CRUD, rutas optimizadas, pesaje idempotente, dashboard y CSV; panel web; app de choferes con cola offline; tests (backend 9, móvil 9); Dev Container; CI en GitHub Actions.
- **Verificación:** flujo completo en local, incluido pesaje offline → sincronización, con navegador automatizado.
- **Hecho (preparación de producción):** imágenes ARM64 publicables en GHCR, Compose de producción sin mounts, Caddy/TLS, workflow de despliegue SSH y guía de secretos.
- **Hecho (polyrepo y CI/CD, ver [cicd-strategy.md](cicd-strategy.md) y ADR-017/018 en [decisions.md](../architecture/decisions.md)):** código separado en recycling-backend/web/mobile/infra; branch protection con PR+aprobación en `main` de los 4 repos; diseño de versionado SemVer test/release documentado.
- **Pendiente para producción:** implementar `ci.yml`/`release.yml` en recycling-backend/web/mobile y adaptar `deploy.yml` en recycling-infra (D-07); obtener y configurar credenciales Oracle reales siguiendo [oracle-credentials-setup.md](../infrastructure/oracle-credentials-setup.md) y ejecutar `terraform plan`/`apply` (D-08); configurar DNS; crear `.env` en VM; clave de Google Routes; migraciones (Alembic); endurecimiento de auth (secretos, refresh); `cap add ios/android`; prueba física sin señal; primer release versionado y primer despliegue (D-09); backups de BD.
