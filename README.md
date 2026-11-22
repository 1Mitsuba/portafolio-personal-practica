# Portafolio personal — Wendy Ramírez

Portafolio sencillo con Vue 3, TypeScript, Vite, FastAPI y PostgreSQL de Supabase. Incluye pruebas unitarias, de integración y E2E; Docker Compose; Terraform para OCI y Cloudflare; y Ansible para configurar y desplegar la VM.

## Ejecución local

Requisitos: Docker Desktop y Git.

1. Copiar `.env.example` como `.env` (`Copy-Item .env.example .env` en PowerShell).
2. Ejecutar `docker compose -f docker-compose.yml -f docker-compose.local.yml --profile local up --build`.
3. Abrir <http://localhost:8080>.

El perfil local usa PostgreSQL en Docker. Para usar tu proyecto Supabase, coloca en `DATABASE_URL` la cadena de conexión **Session pooler** de Supabase. No pongas claves de servicio en variables `VITE_*`, porque esas variables se envían al navegador.

## Pruebas

```powershell
cd frontend
  npm install
npm run test:unit
```

```powershell
cd backend
python -m venv .venv
# Windows: .venv\Scripts\Activate.ps1
# Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
pytest -m unit
```

Las pruebas de integración usan la base PostgreSQL de CI o la base local del Compose. Las E2E requieren la aplicación levantada con Compose:

```powershell
cd frontend
npm install
npx playwright install chromium
npm run test:e2e
```

## Flujo de ramas

- Cada push a `dev` instala dependencias y ejecuta unit, integración y E2E.
- Cada push a `main` ejecuta las mismas comprobaciones, publica las imágenes del frontend y API en Docker Hub, aplica Terraform y ejecuta Ansible para desplegar.

## Preparación de servicios

### GitHub

Crear las ramas `dev` y `main`, y cargar estos **Repository secrets** (nunca subir valores al repositorio):

| Secreto | Contenido |
|---|---|
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub, por ejemplo `1mitsuba` |
| `DOCKERHUB_TOKEN` | Access token de Docker Hub con permiso de escritura |
| `TF_API_TOKEN` | Token de HCP Terraform |
| `HCP_TFC_ORGANIZATION` | Nombre exacto de la organización HCP Terraform |
| `OCI_TENANCY_OCID` | OCID de la tenancy |
| `OCI_USER_OCID` | OCID del usuario OCI |
| `OCI_FINGERPRINT` | Fingerprint de la clave API OCI |
| `OCI_COMPARTMENT_OCID` | OCID del compartimento donde se crearán los recursos |
| `OCI_PRIVATE_KEY` | Contenido PEM de la clave privada API OCI |
| `OCI_SSH_PUBLIC_KEY` | Clave pública autorizada para SSH a Ubuntu |
| `SSH_PRIVATE_KEY` | Clave privada correspondiente a `OCI_SSH_PUBLIC_KEY` |
| `CLOUDFLARE_API_TOKEN` | Token restringido a la zona, con Zone:Read y DNS:Edit |
| `CLOUDFLARE_ZONE_ID` | Zone ID del dominio |
| `SUPABASE_DATABASE_URL` | Connection string Session pooler de Supabase |

Agregar estas **Repository variables**:

| Variable | Ejemplo |
|---|---|
| `PORTFOLIO_DOMAIN` | `portfolio.tudominio.com` |
| `OCI_REGION` | Región habilitada en tu tenancy, por ejemplo `sa-santiago-1` |
| `OCI_IMAGE_OCID` | OCID de una imagen Ubuntu 24.04 x86_64 de `sa-santiago-1` |
| `OCI_ADMIN_CIDR` | Tu IP pública con `/32`, para restringir SSH |

El dominio debe estar agregado a Cloudflare y usar sus nameservers antes del primer despliegue. Crear previamente los repositorios públicos `portfolio-frontend` y `portfolio-api` dentro de la cuenta Docker Hub para que el runner pueda publicar.

### HCP Terraform (estado remoto)

Crear una organización y un workspace llamado `portfolio-personal`. En el workspace seleccionar **Local execution mode**: Terraform se ejecuta en el runner de GitHub Actions, pero el estado queda guardado remotamente en HCP Terraform. Crear un token de usuario y guardarlo como `TF_API_TOKEN`. El nombre de organización debe coincidir con `HCP_TFC_ORGANIZATION`.

### OCI

Usaremos inicialmente `VM.Standard.E4.Flex` con **1 OCPU, 2 GB RAM y 50 GB de boot volume**, sujeto a que la shape esté habilitada en tu tenancy de Santiago. La lista pública de Oracle LatAm da una referencia de aproximadamente **US$26.80/mes para cómputo** si permanece activa 730 horas (1 OCPU + 2 GB); faltan almacenamiento, tráfico, IP e impuestos. Confirma el total en OCI Cost Estimator antes del primer `main`. Terraform no crea recursos hasta que se ejecute ese workflow.

Crear una clave de API OCI y guardar su clave privada y fingerprint como secretos. Crear o seleccionar una imagen **Canonical Ubuntu 24.04 x86_64/AMD64** compatible con E4 y guardar su OCID en `OCI_IMAGE_OCID`. La VM necesita salida a Internet para instalar paquetes y descargar imágenes Docker.

Al ejecutar `npm install` por primera vez se generará `frontend/package-lock.json`; guarda ese archivo en Git para fijar versiones de dependencias. `terraform init` generará `terraform/.terraform.lock.hcl`; también debe versionarse.

### Subir el proyecto a tu GitHub

Después de extraer el ZIP, crea un repositorio vacío en tu cuenta y usa su URL al configurar el remoto:

```bash
git init -b main
git add .
git commit -m "Crear portafolio personal"
git remote add origin URL_DE_TU_REPOSITORIO
git push -u origin main
git switch -c dev
git push -u origin dev
```

Los pushes a `dev` ejecutarán CI. Al hacer el primer push a `main`, el pipeline intentará crear recursos de pago en OCI, así que primero completa y revisa todos los secretos y variables indicados.

### Supabase

Crear la tabla ejecutando `database/schema.sql` en el SQL Editor de tu proyecto. Copiar la connection string de **Session pooler** a `SUPABASE_DATABASE_URL`. No hace falta compartir aquí el token ni el URI.

### Cloudflare

Agregar el dominio y completar la delegación de nameservers en el registrador. Crear un token API de zona con permisos mínimos **Zone:Read** y **DNS:Edit**. Terraform creará un A record proxied para el host elegido. El certificado del origen se gestiona con Caddy y HTTPS-01; Cloudflare se configura en modo **Full (strict)** después de que el primer certificado esté emitido.

## Diseño de red y seguridad

- VCN con subnet pública e Internet Gateway.
- Ingress: TCP 80/443 desde Internet; SSH TCP 22 solo desde `OCI_ADMIN_CIDR`.
- API y PostgreSQL no publican puertos en la VM; el backend habla con Supabase por TLS.
- Ansible configura UFW y comprueba las reglas efectivas de `iptables`.
- Docker Hub publica tags `sha-<commit>` e `latest`; el servidor despliega el tag inmutable del commit.
- Terraform conserva su estado en HCP Terraform. No guardar `terraform.tfstate`, claves ni `.env` en Git.

## Estructura

```text
frontend/       Vue, TypeScript, Vitest y Playwright
backend/        FastAPI, pytest y conexión PostgreSQL
database/       Esquema SQL para Supabase/PostgreSQL
docker/         Dockerfiles, Nginx y Caddy
terraform/      OCI, red, VM y DNS Cloudflare
ansible/        Instalación segura y despliegue
.github/        Pipeline de GitHub Actions
```
