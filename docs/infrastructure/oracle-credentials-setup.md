# Guía paso a paso — Credenciales Oracle Cloud Always Free

Complementa la tabla de secrets/variables de [recycling-infra/README.md](https://github.com/DungeonEaters/recycling-infra#readme). Aquí se explica **cómo obtener cada valor** en la consola de Oracle Cloud (OCI) y **dónde configurarlo** en GitHub. Alcance: solo recursos elegibles **Always Free**.

## 0. Requisitos previos
- Cuenta Oracle Cloud creada (tier Always Free activado).
- Acceso a la consola OCI con el usuario que usará Terraform (puede ser el usuario administrador de la cuenta para el MVP).
- `gh` CLI autenticado contra `DungeonEaters/recycling-infra` con permisos de administrador del repo (para crear Environment y secrets).

## 1. Tenancy OCID y User OCID
1. Consola OCI → ícono de perfil (arriba derecha) → **Tenancy: <nombre>**. Copiar **OCID** → este es `OCI_TENANCY_OCID`.
2. Perfil → **User settings** → copiar **OCID** del usuario → este es `OCI_USER_OCID`.

## 2. Par de llaves API signing (fingerprint + private key)
1. En **User settings** → **API Keys** → **Add API Key**.
2. Elegir "Generate API Key Pair", descargar la clave privada (`.pem`) — **no subirla a ningún repo**.
3. Click "Add" — la consola muestra un bloque de configuración con el **Fingerprint**: copiarlo → `OCI_API_KEY_FINGERPRINT`.
4. El contenido completo del archivo `.pem` descargado (incluyendo `-----BEGIN PRIVATE KEY-----`/`END`) → `OCI_API_PRIVATE_KEY`.

## 3. Región, compartment, availability domain
1. **Región**: la que se eligió al crear la cuenta (ej. `sa-santiago-1`, `us-ashburn-1`). Visible en la barra superior de la consola → `OCI_REGION`.
2. **Compartment OCID**: Identity & Security → Compartments. Para el MVP puede usarse el compartment raíz (root) de la tenancy o uno dedicado creado a mano (`recycling-logistics`) → copiar su OCID → `OCI_COMPARTMENT_OCID`.
3. **Availability Domain**: Compute → Instances → "Create Instance" muestra el nombre completo exacto (ej. `AbCd:SA-SANTIAGO-1-AD-1`). Copiarlo literal → `OCI_AVAILABILITY_DOMAIN`.

## 4. Imagen Ubuntu ARM64
1. Compute → Images, o directamente en el asistente "Create Instance" → Image → "Change Image" → pestaña **Canonical Ubuntu** → marcar **"See all images in this compartment"** si no aparece primero → elegir la versión LTS más reciente compatible con `VM.Standard.A1.Flex` (ARM64).
2. Click en la imagen → copiar su **OCID** → `OCI_UBUNTU_IMAGE_OCID`.
   - Este OCID es específico por región; si se cambia de región hay que repetir el paso.

## 5. Llave SSH dedicada de despliegue
No reutilizar la llave personal. Generar una exclusiva para CI/CD:
```bash
ssh-keygen -t ed25519 -C "deploy@recycling-logistics" -f ./recycling-deploy-key -N ""
```
- `recycling-deploy-key.pub` (contenido completo) → variable `DEPLOY_SSH_PUBLIC_KEY` (Terraform la inyecta en la VM vía cloud-init).
- `recycling-deploy-key` (privada) → secret `SSH_PRIVATE_KEY` del **Environment de despliegue** (`test`/`production`) usado por `deploy.yml` — no es un secret de Terraform, es para el workflow de deploy que hace SSH a la VM ya creada.

## 6. CIDRs de administración (SSH)
- Obtener la IP pública propia: `curl -4 ifconfig.me`.
- Formato `IP/32` (o rango de oficina/VPN si aplica) → variable `ADMIN_CIDRS` (lista, ej. `["203.0.113.10/32"]`). **Nunca usar `0.0.0.0/0`.**

## 7. Bucket + Customer Secret Key para el estado remoto de Terraform
El bucket es un recurso "bootstrap" creado a mano una sola vez (no lo gestiona Terraform, para evitar el problema del huevo y la gallina).
1. Storage → Object Storage → Buckets → **Create Bucket** (Standard, privado, sin versión pública). Nombre libre, ej. `recycling-tfstate` → `OCI_STATE_BUCKET`.
2. El **namespace** de Object Storage aparece en la misma página de buckets (arriba, "Namespace") o en Administration → Tenancy details → `OCI_STATE_NAMESPACE`.
3. User settings → **Customer Secret Keys** → **Generate Secret Key**. Copiar **Access Key** → `OCI_STATE_S3_ACCESS_KEY` y **Secret Key** (solo se muestra una vez) → `OCI_STATE_S3_SECRET_KEY`.
   - Estas credenciales usan la API S3-compatible de Object Storage; el backend de Terraform se configura con endpoint `https://<namespace>.compat.objectstorage.<region>.oraclecloud.com`.

## 8. Reglas de red (ingress) — ya cubiertas por Terraform
Terraform (`environments/production/main.tf`) crea la VCN y las reglas de entrada para 22 (restringido a `ADMIN_CIDRS`), 80 y 443 (abiertos). No requiere configuración manual adicional si el `apply` corre exitosamente; solo verificar en consola tras el primer apply.

## 9. Configurar todo en GitHub
En `DungeonEaters/recycling-infra` → **Settings → Environments → production** (crear el Environment si no existe; agregar revisores requeridos para producción):

**Secrets:**
```
OCI_TENANCY_OCID
OCI_USER_OCID
OCI_API_KEY_FINGERPRINT
OCI_API_PRIVATE_KEY
OCI_STATE_S3_ACCESS_KEY
OCI_STATE_S3_SECRET_KEY
SSH_PRIVATE_KEY           # para deploy.yml, llave privada del paso 5
```

**Variables:**
```
OCI_REGION
OCI_COMPARTMENT_OCID
OCI_AVAILABILITY_DOMAIN
OCI_UBUNTU_IMAGE_OCID
DEPLOY_SSH_PUBLIC_KEY
ADMIN_CIDRS
OCI_STATE_BUCKET
OCI_STATE_NAMESPACE
SERVER_HOST                # IP pública de la VM, se conoce tras el primer terraform apply
SERVER_USER                # ubuntu
```

Repetir el Environment `test` si se decide tener un ambiente de prueba separado en la misma VM (puertos/dominios distintos, ver ADR-015 y oracle-cloud-provisioning.md).

## 10. Orden de ejecución recomendado
1. Completar pasos 1–7 y cargar todos los secrets/vars.
2. Ejecutar manualmente `Terraform plan` (workflow_dispatch) y revisar el plan.
3. Si el plan es correcto, ejecutar `Terraform apply` (input `APPLY`, requiere aprobación del Environment).
4. Tomar la IP pública de la salida de Terraform → cargarla en `SERVER_HOST`.
5. Seguir [oracle-cloud-provisioning.md](oracle-cloud-provisioning.md) para la configuración inicial de la VM (Docker, `.env`, DNS) antes del primer `deploy.yml`.

## Notas de costo (Always Free)
Todos los recursos de esta guía están dentro de cuotas Always Free: 1 VM Ampere A1 (hasta 4 OCPU/24GB), Object Storage (hasta 20GB), VCN/Load Balancer básico y egress de red (10TB/mes) no tienen cargo mientras no se exceda la cuota. Revisar el panel **Governance → Cost Management** periódicamente para confirmar que no haya cargos inesperados.
