# GEMA — Gestión Estratégica de Mantenimiento de Activos

> **Repositorio de Infraestructura** · Orquesta el stack completo con Docker Compose

[![Deploy Staging](https://github.com/tu-org/gema-infraestructura/actions/workflows/deploy-staging.yml/badge.svg)](https://github.com/tu-org/gema-infraestructura/actions/workflows/deploy-staging.yml)
[![Deploy Production](https://github.com/tu-org/gema-infraestructura/actions/workflows/deploy-production.yml/badge.svg)](https://github.com/tu-org/gema-infraestructura/actions/workflows/deploy-production.yml)

---

## 📐 Arquitectura

GEMA usa una arquitectura **Polyrepo** con tres repositorios independientes:

```
/proyectos/
  ├── gema-backend/          # FastAPI + Python + Poetry + Alembic
  ├── gema-frontend/         # Next.js + React
  └── gema-infraestructura/  # Docker Compose + Nginx (este repo)
```

### Servicios del Stack

| Servicio   | Imagen / Build              | Puerto interno | Puerto externo | Descripción                    |
|------------|-----------------------------|:--------------:|:--------------:|--------------------------------|
| `db`       | `postgres:15-alpine`        | 5432           | ❌ No expuesto | Base de datos PostgreSQL       |
| `backend`  | Build desde `../gema-backend` | 8000         | 8000           | API REST con FastAPI           |
| `frontend` | Build desde `../gema-frontend`| 3000         | 3000           | Interfaz web con Next.js       |
| `nginx`    | `nginx:alpine`              | 80, 443        | 80, 443        | Reverse proxy y punto de entrada |

> **Seguridad**: La base de datos **no está expuesta** a internet. Solo es accesible internamente a través de `gema-network`.

---

## ✅ Requisitos Previos

Asegúrate de tener instalado en tu máquina (local o servidor):

| Herramienta    | Versión mínima | Verificar con         |
|----------------|:--------------:|-----------------------|
| Docker         | 24.x           | `docker --version`    |
| Docker Compose | 2.x (plugin)   | `docker compose version` |
| Git            | 2.x            | `git --version`       |

---

## 🚀 Inicio Rápido

### 1. Clonar los tres repositorios en paralelo

```bash
# Crea la carpeta del proyecto y entra en ella
mkdir proyectos && cd proyectos

# Clona los tres repos (ajusta la URL de tu organización/usuario)
git clone https://github.com/tu-org/gema-backend.git
git clone https://github.com/tu-org/gema-frontend.git
git clone https://github.com/tu-org/gema-infraestructura.git

# Entra al repo de infraestructura (punto de control del stack)
cd gema-infraestructura
```

### 2. Configurar las variables de entorno

```bash
# Copia la plantilla de ejemplo
cp .env.example .env

# Edita el archivo .env con tus credenciales reales
# En Linux/Mac:
nano .env
# En Windows:
notepad .env
```

**Variables que DEBES cambiar antes de arrancar:**

| Variable           | Descripción                                  | Ejemplo seguro                         |
|--------------------|----------------------------------------------|----------------------------------------|
| `POSTGRES_PASSWORD`| Contraseña de la base de datos               | Usa `openssl rand -base64 24`          |
| `SECRET_KEY`       | Clave para firmar tokens JWT                 | Usa `openssl rand -hex 32`             |
| `DATABASE_URL`     | URL de conexión (debe coincidir con las vars anteriores) | Ver `.env.example`        |

### 3. Levantar el stack

```bash
# Construye las imágenes y levanta todos los servicios en segundo plano
docker compose up --build -d

# Para ver los logs en tiempo real durante el primer arranque:
docker compose up --build
```

El primer arranque puede tardar **3-5 minutos** mientras construye las imágenes de backend y frontend.

---

## 🌐 URLs de Acceso

Una vez levantado el stack, accede a:

| Servicio              | URL                                   |
|-----------------------|---------------------------------------|
| 🖥️  Frontend (App)    | http://localhost:3000                 |
| ⚙️  Backend (API)     | http://localhost:8000                 |
| 📄  API Docs (Swagger)| http://localhost:8000/docs            |
| 📘  API Docs (ReDoc)  | http://localhost:8000/redoc           |

---

## 🛠️ Comandos Útiles

### Gestión del Stack

```bash
# Ver el estado de todos los servicios
docker compose ps

# Ver logs de todos los servicios
docker compose logs -f

# Ver logs de un servicio específico
docker compose logs -f backend
docker compose logs -f frontend
docker compose logs -f db

# Reiniciar un servicio específico sin re-build
docker compose restart backend

# Rebuild y reinicio de un solo servicio
docker compose up -d --build backend

# Detener el stack (los datos de la DB se conservan)
docker compose down

# Detener el stack Y eliminar los volúmenes (⚠️ BORRA LA BASE DE DATOS)
docker compose down -v
```

### Migraciones de Base de Datos (Alembic)

```bash
# Ejecutar migraciones pendientes
docker compose exec backend alembic upgrade head

# Crear una nueva migración
docker compose exec backend alembic revision --autogenerate -m "descripcion_del_cambio"

# Ver historial de migraciones
docker compose exec backend alembic history
```

### Acceso Directo a los Contenedores

```bash
# Abrir una shell en el backend
docker compose exec backend bash

# Conectarse a PostgreSQL (desde dentro del stack)
docker compose exec db psql -U ${POSTGRES_USER} -d ${POSTGRES_DB}
```

---

## 📦 CI/CD con GitHub Actions

Este repo incluye dos workflows automáticos:

| Workflow           | Rama trigger | Servidor    | Secrets necesarios                                    |
|--------------------|:------------:|-------------|-------------------------------------------------------|
| Deploy to Staging  | `develop`    | Staging     | `SERVER_IP`, `SSH_USER`, `SSH_PRIVATE_KEY`            |
| Deploy to Production | `main`     | Producción  | `PROD_SERVER_IP`, `PROD_SSH_USER`, `PROD_SSH_PRIVATE_KEY` |

### Configurar los Secrets en GitHub

1. Ve a tu repositorio en GitHub
2. **Settings** → **Secrets and variables** → **Actions**
3. Crea cada secret con el valor correspondiente

### Configurar el servidor para CI/CD

```bash
# En el servidor (staging o producción), clona los tres repos
mkdir ~/proyectos && cd ~/proyectos
git clone https://github.com/tu-org/gema-backend.git
git clone https://github.com/tu-org/gema-frontend.git
git clone https://github.com/tu-org/gema-infraestructura.git

# Configura el .env en el servidor
cd gema-infraestructura
cp .env.example .env
nano .env  # Configura con los valores de producción/staging
```

---

## 🔒 Configurar HTTPS con Let's Encrypt (Producción)

```bash
# 1. Instala Certbot en el servidor
sudo apt install certbot

# 2. Detén Nginx temporalmente
docker compose stop nginx

# 3. Obtén el certificado
sudo certbot certonly --standalone -d tu-dominio.com

# 4. Habilita el bloque HTTPS en nginx/nginx.conf (ver comentarios en el archivo)
nano nginx/nginx.conf

# 5. Reinicia el stack
docker compose up -d nginx
```

---

## 🗂️ Estructura del Repositorio

```
gema-infraestructura/
├── .github/
│   └── workflows/
│       ├── deploy-staging.yml      # CI/CD → staging (rama develop)
│       └── deploy-production.yml  # CI/CD → producción (rama main)
├── nginx/
│   └── nginx.conf                 # Configuración del reverse proxy
├── .env.example                   # Plantilla de variables de entorno
├── .gitignore                     # Archivos excluidos del repo
├── docker-compose.yml             # Orquestación de servicios
└── README.md                      # Este archivo
```

---

## 🤝 Flujo de Trabajo de Desarrollo

```
feature/* → develop → main
              ↓            ↓
           Staging    Producción
```

1. Desarrolla en ramas `feature/*`
2. Abre un Pull Request hacia `develop`
3. El merge a `develop` dispara el deploy a **Staging**
4. Después de validar en staging, abre un PR de `develop` → `main`
5. El merge a `main` dispara el deploy a **Producción**

---

## 📝 Licencia

Propiedad de [Tu Organización]. Todos los derechos reservados.
