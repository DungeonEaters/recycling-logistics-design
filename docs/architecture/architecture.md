# Arquitectura — Plataforma de Logística y Recolección de Plásticos (MVP)

## 1) Resumen Ejecutivo
- **Objetivo:** Optimizar la recolección punto a punto de plásticos/material reciclable, digitalizando pesaje, rutas y trazabilidad (Ley REP, Chile).
- **Aplicación:** `recycling-logistics` (API + panel web B2B + app móvil de choferes).
- **Estado:** En implementación (MVP local funcionando)
- **Responsable técnico:** Por definir
- **Última actualización:** 2026-10-05
- **Repositorios de implementación (polyrepo, ver ADR-017):** [recycling-backend](https://github.com/DungeonEaters/recycling-backend), [recycling-web](https://github.com/DungeonEaters/recycling-web), [recycling-mobile](https://github.com/DungeonEaters/recycling-mobile), [recycling-infra](https://github.com/DungeonEaters/recycling-infra) (Compose/Caddy/Terraform). El monorepo original [DungeonEaters/recycling-logistics](https://github.com/DungeonEaters/recycling-logistics) queda legado.
- **Origen:** Conversación con Gemini, ver [conversation-gemini.md](../research/conversation-gemini.md) y [sources/](../research/sources/).

## 2) Contexto y Alcance
### 2.1 Problema a resolver
Un camión llega a un punto (edificio, condominio, empresa), pesa los kilos de plástico y sigue al siguiente. Hoy hay papeleo, rutas manuales y poca trazabilidad. Se necesita: rutas eficientes, registro rápido de pesaje (incluso sin señal) y datos para certificación Ley REP.

### 2.2 Alcance incluido (MVP)
- Panel web: CRUD de clientes/condominios/puntos con GPS, asignación de rutas optimizadas, dashboard de progreso y kilos, exportación CSV de trazabilidad.
- App móvil de choferes: login, ruta activa con navegación a Waze/Google Maps, registro de pesaje, tolerancia offline con sincronización.
- Optimización de ruteo vía Google Maps Routes API (`optimizeWaypointOrder`), lotes de hasta 25 puntos, con optimizador local (vecino más cercano + 2-opt) cuando no hay clave de Google o la API falla.

### 2.3 Fuera de alcance (MVP)
Ideas evaluadas y diferidas (ver [backlog.md](../delivery/backlog.md)):
- App de usuarios finales con gamificación (EcoPuntos) y solicitudes on-demand.
- Pesaje IoT por Bluetooth con balanzas.
- Validación QR/NFC en contenedores.
- Marketplace de lotes en tránsito y control de capacidad del camión.
- Motor VRP propio (OR-Tools + OSRM/GraphHopper).
- Tracking GPS en segundo plano.
- Categorización de material (PET 1, HDPE 2…) y certificados automáticos.

## 3) Requisitos
### 3.1 Funcionales
| ID | Requisito | Módulo |
|---|---|---|
| RF-01 | CRUD de clientes, condominios y puntos de recolección con coordenadas GPS | Web |
| RF-02 | Seleccionar puntos del día, optimizar orden vía API y asignar ruta a camión/chofer | Web |
| RF-03 | Dashboard en tiempo real: puntos visitados vs. pendientes, kilos por camión/jornada | Web |
| RF-04 | Exportar planillas CSV para auditoría y certificados | Web |
| RF-05 | Autenticación JWT con roles `admin` y `driver`; un chofer solo ve y pesa sus rutas | Móvil/API |
| RF-06 | Lista cronológica de visitas con botón de navegación a Waze/Google Maps | Móvil |
| RF-07 | Formulario de pesaje in-situ (kilos recibidos) | Móvil |
| RF-08 | Guardado local de pesajes sin red y sincronización posterior idempotente (lote) | Móvil |
| RF-09 | Omitir parada (requiere conexión en el MVP) | Móvil |

### 3.2 No funcionales
- **Rendimiento:** API asíncrona; muchas peticiones simultáneas de choferes.
- **Seguridad:** Autenticación segura; roles; HTTPS; almacenamiento local seguro de pesajes pendientes.
- **Escalabilidad:** Esquema relacional con camino a PostGIS; camino a motor VRP propio.
- **Mantenibilidad:** Contenedores Docker; frontend y backend desacoplados vía API.
- **Experiencia de uso:** UI táctil nativa (Ionic) para choferes; UI B2B limpia (shadcn/ui).
- **Resiliencia offline:** Ningún pesaje se pierde por falta de señal o limpieza de caché (cola persistida con Capacitor Preferences, no solo IndexedDB).
- **Zona horaria de negocio:** `America/Santiago` para "hoy" y filtros de reportes; los instantes se guardan en UTC.
- **Costo:** Costos fijos mensuales < ~$30 USD en los primeros meses.

## 4) Arquitectura Propuesta
### 4.1 Diagrama de alto nivel
```text
 [App Móvil Choferes]            [Panel Web B2B]
 Capacitor + Ionic UI            Astro/React + Tailwind + shadcn/ui
        │  HTTPS / JSON                 │  HTTPS / JSON
        └──────────────┬────────────────┘
                       ▼
              [API FastAPI (Python)]  ── Routes API ──► [Google Maps Platform]
                       │
                       ▼
                 [PostgreSQL] (PostGIS a futuro)
```

### 4.2 Componentes principales
| Componente | Responsabilidad | Tecnología |
|---|---|---|
| Backend / API | Usuarios/auth, clientes, puntos, camiones, rutas, pesajes, dashboard y exportes; integra ruteo | Python 3.12, FastAPI, SQLAlchemy 2, JWT |
| Base de datos | Persistencia relacional y futura capa geoespacial | PostgreSQL (+PostGIS futuro) |
| Panel Admin | Gestión comercial, asignación de rutas, dashboard, trazabilidad | React 18 + Vite, Tailwind v4, componentes estilo shadcn/ui, TanStack Query, Leaflet |
| App Móvil | Ruta activa, pesaje, offline sync | React + Vite, Ionic UI, Capacitor (Preferences) |
| Motor de ruteo (MVP) | Ordenar paradas (VRP simple) | Google Routes API `optimizeWaypointOrder`; respaldo local NN + 2-opt |
| Infra local | Servicios aislados con hot reload | Docker Compose: `db`, `api`, `web`, `mobile` |
| Infra producción | Imágenes inmutables y TLS | Oracle Cloud ARM64, Docker Compose, GHCR, Caddy, Nginx |
| Entorno y CI/CD | Entorno replicable, validación y publicación versionada (SemVer, ver [cicd-strategy.md](../delivery/cicd-strategy.md)) | Dev Container (Python + Node), GitHub Actions por repo |

### 4.3 Flujo principal
1. Operaciones registra clientes/puntos con GPS en el panel.
2. Selecciona los puntos del día (máx. 25); el backend obtiene el orden óptimo (Routes API o respaldo local).
3. La ruta ordenada se asigna a un camión/chofer.
4. El chofer ve la ruta en la app, navega con Waze/Maps y registra el pesaje en cada punto.
5. Sin señal, el pesaje se guarda localmente y se sincroniza al recuperar red.
6. El dashboard muestra el progreso; se exportan CSV para auditoría/Ley REP.

## 5) Diseño Funcional y Técnico
### 5.1 Casos de uso clave
- Despachar ruta del día (admin).
- Ejecutar ruta y registrar pesaje (chofer).
- Sincronizar pesajes offline (app).
- Exportar trazabilidad por cliente/periodo (admin).

### 5.2 Contratos e interfaces
- **APIs:** REST/JSON FastAPI; contrato vivo en `/docs` (OpenAPI) de [recycling-backend](https://github.com/DungeonEaters/recycling-backend). Recursos: `/auth`, `/users`, `/clients`, `/points`, `/trucks`, `/routes` (+ `/routes/optimize`, `/routes/mine`), `/weighings` (+ `/weighings/batch`), `/dashboard`, `/reports/traceability.csv`.
- **Integraciones externas:** Google Maps Routes API (límite ~25 waypoints intermedios por consulta); deep links a Waze/Google Maps.
- **Sync offline:** el cliente genera el UUID del pesaje; `POST /weighings/batch` responde por ítem `{ok, duplicate, error}`. `ok` o `duplicate` retiran el ítem de la cola; errores de red, 5xx y 401 lo mantienen en cola; errores de validación quedan visibles con reintento manual. Un UUID ya usado por otro chofer se rechaza.

### 5.3 Modelo de datos (preliminar)
- **Entidades:** Usuario (rol), Cliente (tipo: empresa/condominio), Punto de recolección (lat/lng), Camión (patente, capacidad), Ruta (fecha, estado, optimizador usado, origen), Parada de ruta (orden, estado `pending/visited/skipped`), Pesaje (UUID cliente, kilos, material, `weighed_at` del dispositivo, `received_at` del servidor).
- **Relaciones:** Cliente 1—N Punto; Ruta 1—N Parada; Parada → Punto; Ruta → Camión/Chofer; Pesaje → Parada.
- **Trazabilidad:** pesajes inmutables con timestamp y autor; exportables por cliente/periodo. Clientes, puntos y camiones se borran de forma lógica (`active=false`) para conservar el historial; una ruta con pesajes no se puede eliminar. El CSV neutraliza fórmulas (`= + - @`) en campos de texto libre.

## 6) Decisiones de Arquitectura
Resumen; detalle en [decisions.md](decisions.md).
| ID | Decisión | Motivo |
|---|---|---|
| ADR-001 | Python + FastAPI backend | Async, rápido, ecosistema de ruteo (OR-Tools) |
| ADR-002 | PostgreSQL | Relacional, PostGIS a futuro |
| ADR-003 | Frontend desacoplado (React/Astro) | Libertad de UI B2B vs. Streamlit/Django admin |
| ADR-004 | Capacitor + Ionic para móvil | Reutilizar código web, acceso a plugins nativos |
| ADR-005 | Google Maps Routes API en MVP | Rapidez; migrar a OR-Tools + OSRM al crecer |
| ADR-006 | Oracle Cloud Always Free ARM64 + Docker Compose; Caddy como reverse proxy | Costo fijo mínimo y TLS automático |
| ADR-007 | Tailwind + shadcn/ui + Ionic UI | Estilos transversales |
| ADR-008 | Optimizador local como respaldo del ruteo | No bloquear el despacho si falla Google o no hay clave |
| ADR-009 | Pesaje idempotente con UUID generado por el cliente | Reintentos offline sin duplicados |
| ADR-010 | Monorepo para el MVP (reemplazado) | Ver ADR-017: ahora polyrepo |
| ADR-011 | Dev Container + GitHub Actions | Entorno replicable y verificación en cada PR |
| ADR-013 | Imágenes multi-stage Nginx para web y móvil | No ejecutar Vite en producción; imágenes pequeñas e inmutables |
| ADR-014 | GHCR + despliegue SSH desde GitHub Actions | Publicar ARM64 y desplegar el commit exacto en la VM |
| ADR-015 | Caddy como único punto público | TLS Let's Encrypt automático; API/panel/app no exponen puertos directamente |
| ADR-016 | Oracle Cloud Always Free como base del MVP | Costo fijo cero dentro de cuota |
| ADR-017 | Polyrepo (backend/web/mobile/infra) | Versionado y CI/CD independiente por servicio |
| ADR-018 | SemVer con canales test/release + GitHub Releases | Trazabilidad real de versiones publicadas |
| ADR-019 (propuesto) | Servicios Oracle Always Free adicionales | Bastion, Monitoring, backups a Object Storage |

## 7) Riesgos y Mitigaciones
| Riesgo | Impacto | Mitigación |
|---|---|---|
| Pérdida de pesajes por caché borrada del WebView | Alto | Cola en Capacitor Preferences (implementado). Evaluar SQLite si crece el volumen |
| GPS se apaga en segundo plano | Medio | Plugin nativo de geolocalización en background (post-MVP) |
| Google no considera peso/capacidad ni ventanas horarias | Medio | Aceptado en MVP; migrar a OR-Tools |
| Integración con Google Routes sin validar con clave real | Medio | Probar con clave antes de producción; el respaldo local cubre fallas |
| Esquema creado con `create_all` (sin migraciones) | Medio | Introducir Alembic antes de datos reales |
| Auth básica (JWT sin refresh/rotación, secreto de desarrollo) | Medio | Secreto por entorno, expiración corta y refresh antes de producción |
| Límite de ~25 paradas por ruta | Medio | Dividir lotes; migrar a motor propio |
| Costo de Routes API al escalar | Medio | Monitorear cuota de $200/mes; caché de matrices; OSRM |
| Web Bluetooth limitado en iOS | Bajo (post-MVP) | Plugin nativo de BLE |
| Un VPS único sin alta disponibilidad | Medio | Backups de DB; migrar a gestionado al validar |
| Reinicio breve durante `docker compose up` en producción | Medio | Compose con una réplica no garantiza cero downtime; introducir réplicas y un orquestador/balanceador cuando sea requisito |
| Credenciales de producción en repositorio | Alto | `.env` solo en la VM, permisos `600`, secretos/variables en GitHub Actions |
| Certificado TLS no emitido | Alto | DNS de dominio, `driver` y `api` apuntan a la VM; abrir 80/443 en Oracle Cloud y firewall |

## 8) Preguntas Abiertas y Próximos Pasos
- ¿Modelo de negocio: B2B (empresas) o residencial (edificios/casas)? (Gemini lo preguntó; sin respuesta.)
- ~~¿Astro o React para el panel?~~ Resuelto: React + Vite (ADR-003).
- Autenticación: implementados `admin` y `driver`; falta decidir rol `cliente` B2B y refresh de tokens.
- ~~Modelo de datos y contrato del pesaje~~ Resuelto en la implementación (sección 5).
- ¿Requisitos exactos de certificado/planilla Ley REP?
- Tamaño de flota inicial y volumen diario de puntos.

## 9) Criterios de Aceptación del MVP
- [x] Admin crea puntos con GPS y despacha una ruta optimizada a un chofer (local).
- [x] Chofer completa ruta y registra pesajes desde la app (navegador móvil).
- [ ] Pesajes sin señal se sincronizan sin pérdida ni duplicados — verificado en navegador con modo offline simulado; falta prueba física en subterráneo.
- [x] Dashboard refleja progreso y kilos de la jornada.
- [x] CSV de trazabilidad exportable por cliente/periodo.
- [ ] Apps nativas (iOS/Android) empaquetadas con Capacitor.
- [ ] Desplegado en producción con costo fijo < ~$30 USD/mes.

## 10) Referencias
- [implementation-plan.md](../delivery/implementation-plan.md) · [backlog.md](../delivery/backlog.md) · [decisions.md](decisions.md)
- [cicd-strategy.md](../delivery/cicd-strategy.md) — estrategia de CI/CD, versionado SemVer y canales test/release
- [oracle-cloud-provisioning.md](../infrastructure/oracle-cloud-provisioning.md) — provisión de la base Oracle Cloud Always Free, DNS, red y operación
- [oracle-credentials-setup.md](../infrastructure/oracle-credentials-setup.md) — paso a paso para obtener y configurar credenciales OCI
- [conversation-gemini.md](../research/conversation-gemini.md) — contexto original
- Fuentes originales: [sources/](../research/sources/)
- Repositorios de implementación (polyrepo): [recycling-backend](https://github.com/DungeonEaters/recycling-backend) · [recycling-web](https://github.com/DungeonEaters/recycling-web) · [recycling-mobile](https://github.com/DungeonEaters/recycling-mobile) · [recycling-infra](https://github.com/DungeonEaters/recycling-infra) — ver cada README para ejecutar en local (Dev Container en `recycling-infra`) o desplegar en Oracle Cloud. Monorepo legado: [DungeonEaters/recycling-logistics](https://github.com/DungeonEaters/recycling-logistics).
