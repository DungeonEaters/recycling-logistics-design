# Documento Guía: MVP Plataforma de Logística y Recolección de Plásticos

## 1. Visión General del Proyecto
Desarrollar la primera versión funcional (MVP) de una plataforma tecnológica que optimice la recolección punto a punto de plásticos. El sistema conectará a los choferes en terreno con una central administrativa, digitalizando el registro de pesaje, optimizando las rutas de recolección y generando la trazabilidad necesaria para el cumplimiento normativo (Ley REP en Chile).

## 2. Stack Tecnológico Definido
La arquitectura está pensada para ser rápida de desarrollar, económica en su etapa inicial, pero con bases sólidas para escalar. Se utilizará un ecosistema basado en contenedores Docker para estandarizar los entornos.

* **Backend & API:** Python con **FastAPI**. (Asíncrono, rápido y excelente para manejar peticiones simultáneas de choferes en ruta).
* **Base de Datos:** **PostgreSQL** (Robusta, relacional y con soporte para datos geoespaciales con PostGIS si se requiere a futuro).
* **Frontend (Panel Admin):** **React o Astro**, estilizado con Tailwind CSS y componentes de shadcn/ui para una interfaz B2B limpia y profesional.
* **Aplicación Móvil (Choferes):** Aplicación web empaquetada como app híbrida usando **Capacitor**. Interfaz construida con componentes de Ionic UI para garantizar una experiencia táctil nativa.
* **Motor de Ruteo (MVP):** API de **Google Maps (Routes API)** utilizando la función `optimizeWaypointOrder` para resolver el vendedor viajero (VRP) en lotes de hasta 25 puntos por ruta.

## 3. Alcance del MVP (Funcionalidades Core)

### A. Panel de Administración Web (Operaciones / B2B)
* **Gestión de Puntos de Retiro:** CRUD de clientes, edificios y condominios con sus coordenadas GPS.
* **Asignación de Rutas:** Interfaz para seleccionar los puntos del día, enviarlos a la API de Google Maps para optimización y asignar la ruta.
* **Dashboard de Monitoreo:** Vista en tiempo real del progreso de la recolección (puntos visitados vs. pendientes) y kilos acumulados.
* **Exportación de Datos:** Generación de reportes básicos en CSV para que los clientes B2B puedan certificar sus cuotas de reciclaje.

### B. Aplicación Móvil (Choferes)
* **Autenticación:** Login seguro para el personal en terreno.
* **Vista de Ruta Activa:** Listado ordenado de los puntos a visitar en el día, con botón de "Navegar" hacia Waze/Maps.
* **Registro en Punto (Check-in):** Formulario rápido para ingresar los kilos de plástico recolectados en el lugar.
* **Tolerancia Offline Básica:** Capacidad de guardar el pesaje localmente si no hay señal de internet, sincronizando automáticamente después.

## 4. Estimación de Tiempos y Fases de Desarrollo
Estimación para un equipo de 2 personas (1 Full-Stack, 1 Frontend): **10 a 12 semanas**.

* **Fase 1 (Semanas 1-2):** Modelado de datos, configuración de repositorios y Docker Compose.
* **Fase 2 (Semanas 3-5):** Motor Backend en FastAPI, base de datos y endpoints. Integración con Google Maps API.
* **Fase 3 (Semanas 6-8):** Panel Administrativo web consumiendo la API. Vistas de monitoreo.
* **Fase 4 (Semanas 9-11):** App Móvil Híbrida con Ionic UI y Capacitor. Lógica offline.
* **Fase 5 (Semana 12):** Pruebas en terreno (QA), validación de rutas y paso a producción.

## 5. Estrategia de Despliegue y Presupuesto Inicial (Mensual)
Diseñado para mantener los costos operativos fijos por debajo de los $30 USD mensuales.

1. **Infraestructura Backend:** Servidor VPS básico (Hetzner Cloud o DigitalOcean) con Docker. ~$5 - $10 USD/mes.
2. **Alojamiento Frontend:** Vercel o Cloudflare Pages. $0 USD (Plan gratuito).
3. **Mapas y Rutas (Google Maps):** Utilizando el crédito gratuito mensual de $200 USD. $0 USD.
4. **Dominio y Correos corporativos:** ~$15 USD/mes.