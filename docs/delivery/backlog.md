# Backlog — Logística y Recolección de Plásticos

Prioridad: **P0** MVP · **P1** post-MVP cercano · **P2** futuro. Fases en [implementation-plan.md](implementation-plan.md).

## MVP (P0)
### Fase 1
- [x] T1.1 Modelar entidades y diagrama de BD (Usuario, Cliente, Punto, Camión, Ruta, Parada, Pesaje)
- [x] T1.2 Configurar repositorios (backend, frontend, móvil)
- [x] T1.3 Docker Compose con FastAPI + PostgreSQL
### Fase 2
- [x] T2.1 Auth y roles (chofer/admin)
- [x] T2.2 CRUD clientes, condominios y puntos con GPS
- [x] T2.3 CRUD camiones y asignación
- [x] T2.4 Integración Routes API con `optimizeWaypointOrder` (≤25 puntos)
- [x] T2.5 Endpoint de rutas (crear, asignar, estado de paradas)
- [x] T2.6 Endpoint de pesaje idempotente (UUID cliente)
- [x] T2.7 Exportación CSV de trazabilidad
### Fase 3
- [x] T3.1 Layout y login del panel (Tailwind + shadcn/ui)
- [x] T3.2 Gestión comercial
- [x] T3.3 Asignación de rutas
- [x] T3.4 Dashboard de progreso y kilos
- [x] T3.5 Descarga de planillas CSV
### Fase 4
- [x] T4.1 Proyecto Ionic + Capacitor, login
- [x] T4.2 Vista de ruta activa + deep link Waze/Google Maps
- [x] T4.3 Formulario de pesaje
- [x] T4.4 Cola offline y sincronización
- [ ] T4.5 Empaquetado iOS/Android
### Fase 5
- [ ] T5.1 Pruebas físicas sin señal
- [ ] T5.2 Validación de rutas vs. realidad
- [ ] T5.3 Despliegue a producción

## Deuda técnica surgida en la implementación
- [ ] D-01 Migraciones con Alembic (hoy `create_all`)
- [ ] D-02 Endurecer auth: secreto por entorno, refresh/rotación de tokens, rate limit en login
- [ ] D-03 Validar integración Google Routes con clave real y cuota
- [ ] D-04 Omitir parada en modo offline (hoy requiere conexión)
- [ ] D-05 Tests e2e automatizados (Playwright) del flujo admin → chofer → dashboard
- [ ] D-06 Lint/format (ruff, eslint) en CI
- [ ] D-07 Pipeline de CI/CD completo por repo (ver [cicd-strategy.md](cicd-strategy.md)): `ci.yml` + `release.yml` en recycling-backend/web/mobile, `deploy.yml` adaptado en recycling-infra
- [ ] D-08 Configurar credenciales Oracle Cloud reales en `recycling-infra` (ver [oracle-credentials-setup.md](../infrastructure/oracle-credentials-setup.md)) y ejecutar `terraform plan`/`apply` contra Oracle real
- [ ] D-09 Primer release versionado (`v0.1.0`) por repo y primer despliegue manual a la VM Oracle
- [ ] D-10 Activar required status checks (`ci.yml`) en branch protection de recycling-backend/web/mobile una vez exista CI

## Post-MVP
| ID | Ítem | Prioridad | Notas |
|---|---|---|---|
| B-01 | Tracking GPS en segundo plano (plugin nativo) | P1 | Web apaga GPS con pantalla bloqueada |
| B-02 | Categorización de material (PET 1, HDPE 2…) | P1 | Base de datos de alta pureza |
| B-03 | Certificado automático de disposición final (Ley REP) | P1 | Al pesar en el punto |
| B-04 | Validación QR/NFC en contenedor | P2 | Hora exacta de llegada/salida |
| B-05 | Pesaje IoT por Bluetooth | P2 | Web Bluetooth limitado en iOS; plugin nativo |
| B-06 | App de usuarios con EcoPuntos y dashboard de impacto | P2 | Condominios/oficinas/personas |
| B-07 | Solicitud de retiro on-demand | P2 | Retiros extraordinarios |
| B-08 | Motor VRP propio: OR-Tools + OSRM/GraphHopper | P2 | Capacidad por camión y ventanas horarias |
| B-09 | Marketplace de lotes en tránsito | P2 | Venta a plantas recicladoras |
| B-10 | Alerta de capacidad al 90 % y desvío a planta | P2 | |
| B-11 | PostGIS | P2 | |
| B-12 | Reportes de sostenibilidad para auditorías | P1 | |

## Riesgos abiertos
Ver [architecture.md](../architecture/architecture.md) sección 7.

## Preguntas abiertas
Ver [architecture.md](../architecture/architecture.md) sección 8.
