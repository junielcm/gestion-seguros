# Ejecución con Docker

Guía para levantar el proyecto completo (backend Django + frontend React) con un
solo comando, sin instalar Python ni Node en tu máquina.

---

## Inicio rápido

```bash
# Desde la raíz del proyecto (la carpeta que contiene docker-compose.yml)
cp .env.example .env      # opcional: todo tiene valores por defecto

docker compose up -d --build
```

La primera vez tarda unos minutos (descarga imágenes y compila el frontend).
Las siguientes aprovechan la caché.

Cuando termine:

| URL | Qué es |
|---|---|
| http://localhost:8080/ | Aplicación React |
| http://localhost:8080/admin/ | Panel de administración de Django |
| http://localhost:8080/api/v1/ | API REST |

### Credenciales de prueba

Se cargan solas en el primer arranque.

| Rol | Usuario | Contraseña |
|---|---|---|
| Panel Admin | `admin` | `admin` |
| Gerente | `gerente1` | `123456` |
| Analista de Seguros | `analista1` | `123456` |
| Médico | `medico1` | `123456` |
| Recepción de Clínica | `recepcion1` | `123456` |
| Cliente | `cliente1` | `123456` |

### Ver los logs

```bash
docker compose logs -f            # ambos servicios
docker compose logs -f backend    # solo el backend
```

---

## Comandos habituales

```bash
# Estado de los contenedores
docker compose ps

# Reiniciar
docker compose restart

# Parar (conserva los datos)
docker compose down

# Parar y BORRAR los volúmenes (se pierde la base de datos y los archivos subidos)
docker compose down -v

# Reconstruir tras cambiar código
docker compose up -d --build

# Abrir una shell de Python dentro del backend
docker compose exec backend python manage.py shell

# Ejecutar un comando de gestión de Django
docker compose exec backend python manage.py createsuperuser
```

> **Nota sobre los cambios en el código:** el backend se monta con `COPY` en la
> imagen, no con un bind-mount. Por eso, si editas un archivo `.py`, necesitas
> `docker compose up -d --build` para que el cambio se aplique. El frontend sí
> se reconstruye rápido porque la imagen de build solo usa `npm ci` + `vite build`.

---

## Arquitectura

```
   navegador
      │
      │  http://localhost:8080
      ▼
┌─────────────────────────────────┐
│ frontend  (nginx:alpine)       │   Sirve la SPA compilada
│                                 │   + proxy inverso
└──────┬──────────────┬───────────┘
       │              │
  /api/│/admin/       │/static/  /media/
       │              │             │
       ▼              ▼             │
┌──────────────┐  ┌──────────────────────┐
│ backend      │  │ volúmenes (compart.)│
│ (gunicorn)   │  │ backend_static       │
│ Django 6.1   │  │ backend_media        │
└──────┬───────┘  └──────────────────────┘
       │
       ▼
┌──────────────┐
│ backend_data │  SQLite (db.sqlite3)
└──────────────┘
```

Solo el **frontend** publica puertos en el host. El backend se expone
únicamente dentro de la red interna de Compose: no es accesible desde fuera
del equipo, lo que reduce la superficie de ataque.

### Rutas

| Ruta | Servida por | Va a |
|---|---|---|
| `/` | nginx | `dist/index.html` (la SPA) |
| `/assets/*` | nginx | Archivos compilados por Vite (cache 1 año) |
| `/api/v1/*` | nginx → proxy | backend:8000 |
| `/admin/*` | nginx → proxy | backend:8000 (Django admin) |
| `/static/*` | nginx | Volumen `backend_static` |
| `/media/*` | nginx | Volumen `backend_media` |

Como la API se sirve desde el **mismo origen** que el frontend, el navegador
no hace peticiones cross-origin y **no hace falta CORS**.

---

## Qué hace el arranque del backend

`backend/entrypoint.sh` se ejecuta en cada `docker compose up`:

1. **Espera a la base de datos** (solo si hay `DB_HOST`; con SQLite no espera).
2. **`python manage.py migrate`** — aplica las migraciones pendientes.
3. **`python manage.py collectstatic`** — genera los estáticos del admin en el
   volumen que nginx lee.
4. **`python setup_demo.py`** — carga clínicas, servicios, usuarios y la póliza
   de prueba. Se controla con `LOAD_DEMO_DATA`.
5. **`gunicorn`** — sirve la API en `0.0.0.0:8000` con varios workers.

`setup_demo.py` es **idempotente**: usa `get_or_create` en todos los modelos, así
que puedes dejarlo en `true` sin miedo a duplicar datos en cada arranque.

El healthcheck del backend golpea `/admin/login/`. El contenedor `frontend` no
arranca hasta que el backend está sano, para no enrutar tráfico hacia un
gunicorn que todavía está aplicando migraciones.

---

## Configuración

Todo se ajusta en el archivo `.env` de la raíz (plantilla en `.env.example`).

| Variable | Por defecto | Para qué sirve |
|---|---|---|
| `FRONTEND_PORT` | `8080` | Puerto en el host donde se publica la app |
| `DEBUG` | `False` | Django en modo depuración. **No lo actives en producción** |
| `SECRET_KEY` | insegura | **Cámbiala antes de desplegar** |
| `ALLOWED_HOSTS` | `*` | Dominios permitidos, separados por comas |
| `LOAD_DEMO_DATA` | `true` | Cargar los datos de prueba en cada arranque |
| `GUNICORN_WORKERS` | `3` | Nº de workers. Guía: `(núcleos × 2) + 1` |
| `VITE_API_URL` | `/api/v1/` | URL de la API incrustada en el bundle |
| `DB_ENGINE` | sqlite3 | Motor de base de datos |
| `DB_NAME` | `/app/data/db.sqlite3` | Ruta de la base de datos |
| `CORS_ALLOW_ALL_ORIGINS` | `True` | Orígenes permitidos (innecesario en este setup) |

### Generar una SECRET_KEY segura

```bash
docker compose run --rm backend python -c \
  "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

---

## La URL de la API en el frontend

El endpoint **nunca está escrito a mano en el código**. Vive en una única
variable, definida en `frontend/src/services/config.js`:

```js
export const API_BASE_URL =
  import.meta.env.VITE_API_URL || 'http://127.0.0.1:8000/api/v1/';
```

Todos los archivos que llaman a la API (incluidos los que usan `fetch` directo,
como `UserManagement.jsx` y `ClinicReceptionDashboard.jsx`) importan esa
constante, así que hay un solo sitio que cambiar.

| Escenario | `VITE_API_URL` | Por qué |
|---|---|---|
| Docker | `/api/v1/` | nginx hace de proxy: ruta relativa, mismo origen |
| Desarrollo local con `npm run dev` | `http://127.0.0.1:8000/api/v1/` | Vite (5173) y Django (8000) son servidores distintos |
| Frontend y API en dominios separados | `https://api.midominio.com/api/v1/` | Requiere `CORS_ALLOWED_ORIGINS` en el backend |

> **Importante:** Vite incrusta las variables `VITE_*` en el bundle **en tiempo
> de compilación**, no cuando se abre el navegador. Si cambias `VITE_API_URL`
> tienes que reconstruir: `docker compose up -d --build` (y `npm run dev` si
> desarrollas fuera de Docker).

---

## Volúmenes y persistencia

| Volumen | Contenido | Se pierde con `down -v` |
|---|---|---|
| `backend_data` | `db.sqlite3` | Sí |
| `backend_static` | Estáticos del admin | Sí (se regeneran solos) |
| `backend_media` | Comprobantes, órdenes e informes médicos | Sí |

Los datos sobreviven a `docker compose down`, a reinicios de Docker y a
`docker compose up --build`. Solo `docker compose down -v` los elimina.

### Copiar la base de datos

```bash
# Respaldo
docker compose exec backend python manage.py dumpdata > respaldo.json

# Cargar un respaldo
docker compose exec backend python manage.py loaddata respaldo.json
```

---

## Cambiar a PostgreSQL

El compose viene con SQLite porque así funciona sin configurar nada. Para
usar PostgreSQL, añade un servicio `db` al `docker-compose.yml`:

```yaml
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${DB_NAME:-gestion_seguros}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secreto}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - gestion-seguros
```

Y en el `.env`:

```bash
DB_ENGINE=django.db.backends.postgresql
DB_NAME=gestion_seguros
DB_USER=postgres
DB_PASSWORD=secreto
DB_HOST=db
DB_PORT=5432
```

Añade `postgres_data` al apartado `volumes:` y `backend` al `depends_on` de
ambos servicios. El `psycopg2-binary` ya está en `requirements.txt` y el
entrypoint espera a la base de datos automáticamente.

---

## Desarrollo local sin Docker

El proyecto sigue funcionando igual, con las URLs por defecto.

**Backend** (http://127.0.0.1:8000):

```bash
cd backend
python -m venv venv
venv\Scripts\activate        # Windows
source venv/bin/activate     # Linux/Mac
pip install -r requirements.txt
python setup_demo.py
python manage.py runserver
```

**Frontend** (http://localhost:5173):

```bash
cd frontend
npm install
npm run dev
```

Si necesitas cambiar la URL de la API en el entorno local, crea
`frontend/.env` a partir de `.env.example`.

---

## Resolución de problemas

**El puerto 8080 está ocupado**

```bash
FRONTEND_PORT=3000 docker compose up -d
```

o cambia `FRONTEND_PORT` en el `.env`.

**`ports: 0.0.0.0:8080->80: bind: address already in use`**

Ya hay un stack anterior corriendo:

```bash
docker compose down
```

**El admin se ve sin estilos**

Casi siempre es `collectstatic` sin ejecutar. Revisa los logs del backend:

```bash
docker compose logs backend | Select-String "collectstatic"
```

**`Permission denied` sobre `/app/data` o `/app/media`**

Los volúmenes se inicializan con los permisos de la imagen, y el backend
escribe como `appuser` (uid 1000). Si el volumen se creó antes de esta
configuración, recrea solo ese volumen:

```bash
docker compose down -v      # OJO: borra la base de datos
docker compose up -d
```

**El backend reinicia en bucle**

Casi siempre es un error de configuración. Mira el log:

```bash
docker compose logs backend
```

**Cambié un archivo del frontend pero la web no cambia**

La imagen contiene el bundle compilado, no un watcher. Reconstruye:

```bash
docker compose up -d --build frontend
```

**Empezar de cero**

```bash
docker compose down -v --rmi local
docker compose up -d --build
```

---

## Archivos de Docker en el proyecto

```
gestion-seguros/
├── docker-compose.yml        # Orquestación de los dos servicios
├── .env.example              # Plantilla de configuración
├── .gitignore
├── DOCKER.md                 # Este archivo
│
├── backend/
│   ├── Dockerfile            # Imagen del backend (gunicorn)
│   ├── .dockerignore
│   ├── entrypoint.sh         # migrate + collectstatic + demo + gunicorn
│   ├── requirements.txt
│   └── config/settings.py    # Lee DEBUG, ALLOWED_HOSTS, DB_* del entorno
│
└── frontend/
    ├── Dockerfile            # Build de Vite + nginx
    ├── .dockerignore
    ├── nginx.conf            # SPA + proxy + /static + /media
    ├── .env.example
    └── src/services/config.js  # Única fuente de la URL de la API
```
