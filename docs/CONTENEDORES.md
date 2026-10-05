# Despliegue manual con Docker

## Arquitectura

Se crean tres contenedores manualmente:

1. `frontend`: imagen propia construida desde `frontend/Dockerfile`. Su etapa final ejecuta Nginx para servir los archivos compilados de React y reenviar `/api` al backend.
2. `backend`: imagen propia construida desde `backend/Dockerfile`. Ejecuta Express con Node.js y usa Mongoose para conectarse a MongoDB.
3. `mongo`: contenedor basado en la imagen oficial `mongo:7.0`. Guarda la base en un volumen Docker persistente.

Los tres se conectan a una red Docker creada explícitamente. Solo se publica el puerto 80 del frontend; el backend y MongoDB permanecen en la red interna.

## Para qué sirve cada archivo

### `backend/Dockerfile`

- Parte de `node:22-alpine` e instala las dependencias de producción con `npm ci --omit=dev`.
- Copia el código, ejecuta como el usuario sin privilegios `node` y arranca `index.js`.
- `EXPOSE 5000` documenta el puerto de Express, pero no lo publica en el host. No necesitamos publicarlo porque Nginx y el backend se comunican por la red Docker.

### `frontend/Dockerfile`

- En la etapa `build`, usa Node.js y npm para instalar dependencias y compilar React/Vite a `dist`.
- En la etapa final, copia solo `dist` a una imagen Nginx. El contenedor frontend ya incluye Nginx; **no hay que crear un cuarto contenedor de Nginx**.
- `frontend/nginx.conf` entrega la aplicación y enruta las solicitudes `/api/...` a `backend:5000`.

### `frontend/src/App.jsx` y `frontend/vite.config.js`

El código del navegador solicita `/api/tasks`, no `http://localhost:5000/api/tasks`. Esa ruta relativa conserva el mismo origen que la página. Nginx recibe la solicitud y la reenvía por la red privada al contenedor llamado `backend`.

`localhost` en una URL de JavaScript significa el equipo que está ejecutando el navegador del usuario, no automáticamente la VM AWS ni el contenedor backend. Por eso `http://localhost:5000` funcionaría en desarrollo local, pero no al abrir la aplicación desde la IP pública de AWS. El proxy Vite de `vite.config.js` mantiene el desarrollo local funcionando en `localhost:5173`.

## Probar primero en Windows

Abre PowerShell en la carpeta raíz del proyecto, donde está `README.md`. Comprueba que Docker Desktop esté iniciado y usando contenedores Linux:

```powershell
docker version
docker info
```

Prepara las variables locales. `.env` está excluido de Git; `.env.example` contiene solo valores de ejemplo. Crea `.env`, reemplaza la contraseña por una clave larga alfanumérica y cárgala en la sesión actual de PowerShell:

```powershell
Copy-Item .env.example .env
code .env

Get-Content .env | ForEach-Object {
		if ($_ -match '^\s*([^#][^=]+)=(.*)$') {
				Set-Item -Path "Env:$($matches[1].Trim())" -Value $matches[2].Trim()
		}
}
```

Construye ambas imágenes. Node.js y npm se ejecutan dentro de Docker, así que no necesitas instalarlos en Windows para esta prueba:

```powershell
docker build -t task-manager-backend:1.0 ./backend
docker build -t task-manager-frontend:1.0 ./frontend
docker pull mongo:7.0
```

Crea la red privada y el volumen, y arranca MongoDB. Estos nombres deben estar disponibles; si ya existen de una ejecución anterior, usa la sección de limpieza más abajo antes de repetir:

```powershell
docker network create task-manager-net
docker volume create task-manager-mongo-data

docker run -d --name mongo `
	--network task-manager-net `
	--mount source=task-manager-mongo-data,target=/data/db `
	-e "MONGO_INITDB_ROOT_USERNAME=$env:MONGO_ROOT_USERNAME" `
	-e "MONGO_INITDB_ROOT_PASSWORD=$env:MONGO_ROOT_PASSWORD" `
	--restart unless-stopped `
	mongo:7.0
```

Espera unos segundos y verifica MongoDB. El resultado exitoso es `1`:

```powershell
docker exec mongo mongosh --quiet `
	--username $env:MONGO_ROOT_USERNAME `
	--password $env:MONGO_ROOT_PASSWORD `
	--authenticationDatabase admin `
	--eval "db.adminCommand('ping').ok"
```

Arranca el backend y después el frontend. Se publica el puerto `8080` del equipo como puerto `80` del contenedor para evitar conflictos con servicios locales que ya usen el puerto 80:

```powershell
docker run -d --name backend `
	--network task-manager-net `
	-e PORT=5000 `
	-e "MONGO_URI=mongodb://$($env:MONGO_ROOT_USERNAME):$($env:MONGO_ROOT_PASSWORD)@mongo:27017/mern_kanban?authSource=admin" `
	--restart unless-stopped `
	task-manager-backend:1.0

docker run -d --name frontend `
	--network task-manager-net `
	-p 8080:80 `
	--restart unless-stopped `
	task-manager-frontend:1.0
```

Abre `http://localhost:8080` en el navegador. Crea una tarea, cambia su estado y elimínala. Para verificar los contenedores y diagnosticar problemas:

```powershell
docker ps
docker logs backend
docker logs frontend
docker logs mongo
```

Si el navegador no carga la aplicación, confirma que Docker Desktop está activo y revisa que el puerto `8080` no esté ocupado. Si la página carga pero la API falla, revisa `docker logs backend` y `docker logs frontend`; el proxy espera que el contenedor backend se llame exactamente `backend`.

Cuando termines, detén y elimina los contenedores, pero conserva el volumen para mantener las tareas:

```powershell
docker rm -f frontend backend mongo
docker network rm task-manager-net
```

Al repetir la prueba puedes reutilizar `task-manager-mongo-data`. Para borrar también las tareas de prueba, elimina explícitamente el volumen con `docker volume rm task-manager-mongo-data`.

## Subir el proyecto a GitHub

La carpeta aún no está inicializada como repositorio Git. Crea primero un repositorio vacío en GitHub: entra en GitHub, selecciona **New repository**, elige un nombre (por ejemplo `task-manager-containers`) y no marques la creación de README, `.gitignore` ni licencia, porque ya tienes archivos locales. Copia la URL HTTPS del repositorio.

En PowerShell, desde la carpeta raíz del proyecto, verifica que el archivo de credenciales local no se vaya a agregar y crea el primer commit:

```powershell
git init
git status --short
git check-ignore .env
git add .
git status
git commit -m "Add manual Docker deployment"
git branch -M main
```

Antes del `commit`, en `git status` confirma que `.env` **no** aparezca entre los archivos preparados. `.env.example` sí se debe subir porque contiene valores de ejemplo, no credenciales reales.

Conecta el repositorio local con el de GitHub y publica la rama `main`; sustituye la URL de ejemplo por la URL HTTPS copiada de GitHub:

```powershell
git remote add origin https://github.com/TU_USUARIO/TU_REPOSITORIO.git
git push -u origin main
```

GitHub puede pedir iniciar sesión en el navegador o usar Git Credential Manager. Para cambios futuros:

```powershell
git add .
git commit -m "Describe el cambio"
git push
```

No publiques `.env`, contraseñas reales, tokens de Docker Hub ni credenciales de AWS. GitHub aloja el código y los Dockerfiles; Docker Hub aloja las imágenes publicadas en la sección correspondiente de esta guía.

## Construir las imágenes en la VM

En la VM Linux instala Docker Engine, clona el repositorio y entra en su carpeta raíz. No es necesario instalar Node.js ni npm en la VM para construir o ejecutar estas imágenes: Docker usa Node/npm dentro de las etapas definidas en los Dockerfiles. Sí necesitas npm en tu equipo si vas a desarrollar o ejecutar la aplicación directamente fuera de Docker.

Construye las imágenes propias:

```bash
docker build -t task-manager-backend:1.0 ./backend
docker build -t task-manager-frontend:1.0 ./frontend
docker image ls
```

## Crear y ejecutar los contenedores manualmente

Ejecuta estos comandos en Bash en la VM Linux. El ejemplo usa una contraseña alfanumérica para evitar caracteres que requieran codificación en la URI de conexión; reemplázala por una contraseña larga y única. No guardes credenciales reales en GitHub.

```bash
cp .env.example .env
# Edita .env y cambia la contraseña de ejemplo antes de continuar.
set -a
source .env
set +a

docker network create task-manager-net
docker volume create task-manager-mongo-data

docker run -d \
	--name mongo \
	--network task-manager-net \
	--mount source=task-manager-mongo-data,target=/data/db \
	-e MONGO_INITDB_ROOT_USERNAME="$MONGO_ROOT_USERNAME" \
	-e MONGO_INITDB_ROOT_PASSWORD="$MONGO_ROOT_PASSWORD" \
	--restart unless-stopped \
	mongo:7.0
```

Antes de iniciar el backend, verifica que MongoDB haya terminado de iniciar. Repite esta comprobación hasta que devuelva `1`:

```bash
docker exec mongo mongosh --quiet \
	--username "$MONGO_ROOT_USERNAME" \
	--password "$MONGO_ROOT_PASSWORD" \
	--authenticationDatabase admin \
	--eval "db.adminCommand('ping').ok"
```

Inicia los otros dos contenedores. El nombre `backend` debe coincidir con el host configurado en `frontend/nginx.conf`:

```bash
docker run -d \
	--name backend \
	--network task-manager-net \
	-e PORT=5000 \
	-e MONGO_URI="mongodb://${MONGO_ROOT_USERNAME}:${MONGO_ROOT_PASSWORD}@mongo:27017/mern_kanban?authSource=admin" \
	--restart unless-stopped \
	task-manager-backend:1.0

docker run -d \
	--name frontend \
	--network task-manager-net \
	-p 80:80 \
	--restart unless-stopped \
	task-manager-frontend:1.0
```

En el grupo de seguridad de AWS permite TCP 80 desde los orígenes requeridos. Abre `http://IP_PUBLICA_DE_LA_VM` y prueba crear, mover y eliminar una tarea. MongoDB no publica el puerto 27017 y Express no publica el 5000.

## Inspección y mantenimiento manual

```bash
docker ps
docker logs backend
docker logs frontend
docker logs mongo
docker network inspect task-manager-net
docker volume inspect task-manager-mongo-data
```

Para detener y eliminar los contenedores sin borrar las imágenes, red ni datos:

```bash
docker stop frontend backend mongo
docker rm frontend backend mongo
```

El volumen `task-manager-mongo-data` mantiene los datos y se vuelve a montar al crear otro contenedor MongoDB con el mismo `--mount`. No ejecutes `docker volume rm task-manager-mongo-data` si necesitas conservarlos.

## Publicar las imágenes propias en Docker Hub

MongoDB usa la imagen oficial `mongo:7.0`. Publica las imágenes propias del frontend y del backend. Cambia `USUARIO_DOCKERHUB` por tu usuario:

```bash
docker login
docker tag task-manager-frontend:1.0 USUARIO_DOCKERHUB/task-manager-frontend:1.0
docker tag task-manager-backend:1.0 USUARIO_DOCKERHUB/task-manager-backend:1.0
docker push USUARIO_DOCKERHUB/task-manager-frontend:1.0
docker push USUARIO_DOCKERHUB/task-manager-backend:1.0
```

En una VM que consuma Docker Hub, ejecuta `docker pull USUARIO_DOCKERHUB/task-manager-frontend:1.0` y `docker pull USUARIO_DOCKERHUB/task-manager-backend:1.0`, y usa esos nombres de imagen en los dos comandos `docker run` correspondientes. Crea igualmente la red y el volumen, y ejecuta MongoDB con su imagen oficial.

## Justificación de MongoDB y el volumen

El proyecto ya define las tareas con Mongoose y se conecta a MongoDB, por lo que este motor no exige adaptar la persistencia. MongoDB es el tercer contenedor. El volumen `task-manager-mongo-data` es almacenamiento persistente asociado al contenedor, no un contenedor adicional ni una base de datos. Evita perder las tareas al eliminar y volver a crear el contenedor.

## Criterios de la actividad

- **Creación:** dos imágenes propias construidas con Dockerfiles y un contenedor MongoDB oficial.
- **Ejecución:** creación manual de red y volumen, y ejecución individual de cada contenedor con `docker run`.
- **Repositorio:** publicación en Docker Hub de las imágenes propias; código y documentación en GitHub.
- **Documentación:** arquitectura, instrucciones, persistencia y propósito de los Dockerfiles descritos aquí.