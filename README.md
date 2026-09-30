# Gestión de Seguros - Docker

Orquestación del sistema de gestión de seguros médicos. Este repositorio no
contiene código de la aplicación: lo levanta con **Docker Compose** combinando
los dos repositorios del proyecto como submódulos.

```
┌──────────────────────────────────────────────────┐
│  gestion-seguros (este repo)                     │
│  docker-compose.yml + submódulos                 │
└───────────────┬──────────────────┬───────────────┘
                │                  │
        ┌───────▼──────┐   ┌───────▼────────┐
        │   backend    │   │   frontend     │
        │ Django + DRF │   │ React + Vite   │
        │  gunicorn    │   │     nginx      │
        └──────────────┘   └────────────────┘
```

- **Aplicación:** http://localhost:8080
- **Panel de administración:** http://localhost:8080/admin/
- **API REST:** http://localhost:8080/api/v1/

---

## Prerrequisitos

Solo necesitas **Docker** y **Docker Compose**. No hace falta instalar Python,
Node ni PostgreSQL en tu equipo.

| Requisito | Versión mínima | Comprobar |
|---|---|---|
| Docker | 20.10+ | `docker --version` |
| Docker Compose | v2+ | `docker compose version` |

---

## Clonar el repositorio

El proyecto usa **submódulos de Git**: `backend/` y `frontend/` son
repositorios independientes. Por eso el `clone` necesita el flag
`--recurse-submodules`, que descarga también el código de ambos.

```bash
git clone --recurse-submodules https://github.com/junielcm/gestion-seguros.git
cd gestion-seguros
```

> **Importante:** sin ese flag, las carpetas `backend/` y `frontend/` quedan
> vacías y `docker compose up` falla con un error de build.

### Si ya clonaste sin el flag

No hace falta volver a clonar. Ejecuta:

```bash
git submodule update --init --recursive
```

### Verificar que todo quedó bien

```bash
git submodule status
```

Deberías ver dos líneas con un hash y `(heads/main)`, sin el prefijo `-` que
indica que falta descargar:

```
 3a455ad8ede66d018a3d2b7b5bb0fe17440cce6c backend (heads/main)
 5ff342ecbffde1d6c5507e110069ce21b1e011c1 frontend (heads/main)
```

---

## Levantar la aplicación

```bash
docker compose up -d --build
```

La primera vez tarda unos minutos (descarga las imágenes base y compila el
frontend). Las siguientes utilizan la caché de Docker.

En cuanto termine:

| URL | Qué es |
|---|---|
| http://localhost:8080/ | Aplicación React |
| http://localhost:8080/admin/ | Panel de administración de Django |
| http://localhost:8080/api/v1/ | API REST |

### Credenciales de prueba

Se crean solas en el primer arranque.

| Rol | Usuario | Contraseña |
|---|---|---|
| Panel Admin | `admin` | `admin` |
| Gerente | `gerente1` | `123456` |
| Analista de Seguros | `analista1` | `123456` |
| Médico | `medico1` | `123456` |
| Recepción de Clínica | `recepcion1` | `123456` |
| Cliente | `cliente1` | `123456` |

### Configuración opcional

Todo tiene valores por defecto, así que el arranque funciona sin configurar
nada. Para cambiarlos:

```bash
cp .env.example .env
```

Las variables más útiles son `FRONTEND_PORT` (por defecto `8080`),
`SECRET_KEY` y `LOAD_DEMO_DATA`. Ver [`.env.example`](.env.example) para la
lista completa.

### Ver los logs

```bash
docker compose logs -f            # ambos servicios
docker compose logs -f backend    # solo el backend
```

---

## Comandos habituales

```bash
docker compose ps                 # estado de los contenedores
docker compose restart            # reiniciar
docker compose down               # parar (conserva los datos)
docker compose up -d --build      # reconstruir tras cambiar código
docker compose down -v            # parar y BORRAR la base de datos y los archivos subidos
```

Los datos persisten en volúmenes con nombre, así que sobreviven a
`docker compose down` y a los reinicios de Docker. Solo `down -v` los elimina.

---

## Documentación

| Documento | Contenido |
|---|---|
| [DOCKER.md](DOCKER.md) | Guía completa: arquitectura, volúmenes, PostgreSQL, resolución de problemas |
| [backend/README.md](backend/README.md) | API REST: modelos, endpoints, permisos por rol |
| [frontend/README.md](frontend/README.md) | Aplicación React: módulos, estado, configuración |
| [.env.example](.env.example) | Todas las variables disponibles |

---

## Repositorios del proyecto

Cada parte tiene su propio repositorio, con su historial y su documentación.

| Repositorio | Contenido |
|---|---|
| [junielcm/gestion-seguros-backend](https://github.com/junielcm/gestion-seguros-backend) | API Django REST Framework, modelos, RBAC |
| [junielcm/gestion-seguros-frontend](https://github.com/junielcm/gestion-seguros-frontend) | Aplicación React + Vite + Tailwind |
| [junielcm/gestion-seguros](https://github.com/junielcm/gestion-seguros) | Este repositorio: orquestación y guía de despliegue |

### Trabajar sobre una parte

Los submódulos son clones normales, así que puedes hacer commits en cada uno
por separado:

```bash
cd backend
git checkout -b mi-mejora
# ... hacer cambios ...
git add -A && git commit -m "Descripción del cambio"
git push -u origin mi-mejora

cd ../frontend
# trabajar de la misma forma
```

Los archivos de este repositorio (compose, guía, `.env.example`) son
independientes de ambos.

---

## Desarrollo local sin Docker

Cada parte también se puede ejecutar por separado, sin contenedores.

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

La URL de la API se toma de `VITE_API_URL`, que en local apunta a
`http://127.0.0.1:8000/api/v1/`. Ver
[frontend/README.md](frontend/README.md#configuración-de-la-api).

> Si modificas código, recuerda reconstruir: `docker compose up -d --build`.
> El backend se copia dentro de la imagen y el frontend se compila, así que no
> hay recarga en caliente.

---

## Licencia

Este proyecto está bajo la licencia MIT. Ver [LICENSE.md](backend/LICENSE.md)
para más detalles.

Autor: **Juniel Cabrices** - 2026
