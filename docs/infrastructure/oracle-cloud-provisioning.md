# Provisión Oracle Cloud Always Free — Recycling Logistics MVP

## Objetivo

Ejecutar los ambientes `test` y `production` en una VM Ubuntu ARM64 Ampere A1 de Oracle Cloud Always Free. Ambos ambientes comparten la VM por costo, pero mantienen directorios, bases de datos, volúmenes, redes, secretos y subdominios separados.

La infraestructura declarativa y runtime viven en [DungeonEaters/recycling-infra](https://github.com/DungeonEaters/recycling-infra). El código de aplicación vive en sus repositorios de backend, web y móvil.

## Arquitectura objetivo

```text
Internet
  |
  +-- 80/443 --> Caddy (TLS Let's Encrypt)
                    |
                    +-- test: web, mobile, api, PostgreSQL
                    |
                    +-- production: web, mobile, api, PostgreSQL

Estación administrativa -- SSH restringido --> Oracle VM
GitHub Actions -- ARM64 images --> GHCR --> Oracle VM pull
```

Dominios requeridos:

| Ambiente | Panel | App choferes | API |
|---|---|---|---|
| Test | `test.<dominio>` | `driver.test.<dominio>` | `api.test.<dominio>` |
| Producción | `<dominio>` | `driver.<dominio>` | `api.<dominio>` |

Los seis registros DNS deben apuntar a la IP pública de la instancia antes del primer despliegue para que Caddy emita certificados.

## 1. Cuenta, región y límites Always Free

1. Elegir una región con capacidad Ampere A1 disponible. La capacidad Always Free no está garantizada; probar otras Availability Domains o una forma menor si falla.
2. Crear o seleccionar un compartment dedicado, por ejemplo `recycling-logistics`.
3. Confirmar cuota: hasta **4 OCPUs y 24 GB RAM** Ampere A1 distribuidos entre las instancias Always Free de la cuenta.
4. Usar una sola VM para este MVP. No es alta disponibilidad ni aislamiento fuerte entre ambientes.

## 2. Clave SSH exclusiva

No usar la clave personal descargada al crear la instancia como secreto de GitHub Actions.

1. Crear una clave `ed25519` exclusiva para administración/despliegue.
2. Entregar solo la clave pública al Terraform/OCI.
3. Mantener la privada cifrada en un gestor de secretos local. No abrir SSH a los
   rangos dinámicos de GitHub-hosted runners.
4. Guardar la host key verificada con:

```bash
ssh-keyscan -H <ip-o-dns-de-la-vm>
```

No aceptar host keys de forma ciega (`StrictHostKeyChecking=no`) dentro del pipeline.

## 3. Terraform

El repo de infraestructura crea:

- VCN, subred pública, Internet Gateway y route table.
- Security List con 80/443 públicos.
- SSH 22 limitado a `admin_cidrs`, nunca abierto a `0.0.0.0/0`.
- Instancia Canonical Ubuntu ARM64 `VM.Standard.A1.Flex`.
- Forma predeterminada: 4 OCPUs y 24 GB.
- Cloud-init para Docker Engine, Docker Compose plugin, UFW y directorios de despliegue.

Pasos iniciales:

```bash
git clone https://github.com/DungeonEaters/recycling-infra.git
cd recycling-infra/environments/production
cp terraform.tfvars.example terraform.tfvars
chmod 600 terraform.tfvars
terraform init
terraform validate
terraform plan
terraform apply
```

`terraform.tfvars`, claves OCI, claves SSH y `terraform.tfstate` nunca se suben a Git.

Antes de automatizar Terraform para más de una persona, migrar el estado a Object Storage cifrado y definir controles de acceso/lock operativo.

## 4. Reglas de red

Configurar ambas capas:

1. **Oracle VCN Security List/NSG:** TCP 80 y 443 desde Internet; TCP 22 solo desde CIDRs de administración.
2. **Ubuntu UFW:** 80, 443 y SSH.

La base de datos, FastAPI, Nginx del panel y Nginx de choferes no publican puertos al host. Solo Caddy publica 80/443.

## 5. Preparar la VM

Cloud-init del Terraform instala Docker. Confirmar al conectarse:

```bash
ssh ubuntu@<ip-publica>
docker --version
docker compose version
```

Crear directorios y archivos de secretos, uno por ambiente:

```bash
sudo install -d -m 700 /opt/recycling-logistics/test
sudo install -d -m 700 /opt/recycling-logistics/production
sudo cp /ruta/.env.production.example /opt/recycling-logistics/test/.env
sudo cp /ruta/.env.production.example /opt/recycling-logistics/production/.env
sudo chmod 600 /opt/recycling-logistics/{test,production}/.env
```

Completar en cada `.env`:

- `APP_DOMAIN` y `ACME_EMAIL`.
- `IMAGE_REPOSITORY=dungeoneaters/recycling-logistics`.
- `POSTGRES_USER`, `POSTGRES_DB`, `POSTGRES_PASSWORD`.
- `JWT_SECRET`.
- `GOOGLE_MAPS_API_KEY` si se utiliza.

Usar contraseñas y secretos generados criptográficamente. No reutilizar los de test en producción.

## 6. Publicación de imágenes y despliegue de test

La infraestructura no contiene secretos de aplicación: Terraform crea recursos y
cloud-init prepara Docker, pero no ejecuta `remote-exec` ni administra `.env`.

Flujo de test:

1. **CI and image build:** los repositorios de aplicación publican una imagen
   inmutable con tag `test-*` en GHCR.
2. Desde una estación administrativa incluida en `ADMIN_CIDRS`, ejecutar
   `make deploy-test-check` y luego
   `make deploy-test-api IMAGE_TAG=<tag>` en `recycling-infra`.
3. El script pide un PAT clásico limitado a `read:packages`, lo transmite por stdin
   para `docker login` y genera secretos de prueba dentro de la VM con permisos `0600`.
4. El script comprueba `http://<ip>/health`. Repetirlo con un tag anterior realiza un
   rollback de aplicación.

Un runner self-hosted aislado podrá reutilizar el script una vez validado el flujo.
No se permite SSH desde GitHub-hosted runners: sus IPs no son una allowlist estable.

## 7. Operación y recuperación

- Configurar backups diarios del volumen PostgreSQL a Object Storage antes de datos reales.
- Configurar alertas de disco, RAM y disponibilidad HTTP.
- Probar restauración de base de datos antes del lanzamiento.
- Actualizar imágenes por tag inmutable `release-*` o `sha-*`, nunca por `latest`.
- Una VM y una réplica por servicio no garantizan cero downtime. El reinicio durante `docker compose up` es una limitación aceptada del MVP.
- Para escalar: separar bases de datos, usar segunda VM/orquestador y un balanceador.

## Checklist de primer despliegue

- [ ] Terraform aplicado y `public_ip` disponible.
- [ ] DNS de los seis hosts propagado.
- [ ] Puertos 80/443 abiertos en OCI y UFW.
- [ ] Docker/Compose verificados en VM.
- [ ] `.env` test y producción creados con secretos distintos.
- [ ] GitHub Environments, secrets y `DEPLOY_PATH` configurados.
- [ ] Imagen `test-*` publicada en GHCR.
- [ ] Deploy manual a `test` validado.
- [ ] Imagen `release-*` publicada en GHCR.
- [ ] Aprobación manual y deploy a producción.
- [ ] Certificados Caddy y `/health` verificados.
