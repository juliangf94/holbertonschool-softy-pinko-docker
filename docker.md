# Docker - Conceptos del Proyecto Softy Pinko

---

## 1. Imagen, contenedor y volumen

### ¿Qué es cada uno?

**Imagen:** Una plantilla de solo lectura que describe el entorno completo: sistema operativo, dependencias, codigo, y el comando que se ejecuta. No corre sola. Es estatica, como una clase en POO.

**Contenedor:** Una instancia en ejecucion de una imagen. Cuando ejecutas `docker run`, Docker toma la imagen y crea un proceso aislado. Puedes tener muchos contenedores corriendo desde la misma imagen al mismo tiempo. Cuando se detiene y elimina, todo lo que escribio en su sistema de archivos desaparece.

**Volumen:** Almacenamiento persistente que existe fuera del contenedor. Sobrevive cuando el contenedor se elimina.

### ¿Por qué importa la distincion?

El problema que Docker resuelve es "funciona en mi maquina pero no en el servidor". La imagen encapsula todo el entorno. El contenedor es descartable y reemplazable. El volumen guarda los datos que no puedes perder.

| Concepto   | Analogo en POO    | Persiste?          |
|------------|-------------------|--------------------|
| Imagen     | Clase             | Si                 |
| Contenedor | Instancia/objeto  | No (por defecto)   |
| Volumen    | Base de datos     | Si                 |

---

## 2. Instrucciones del Dockerfile

Un Dockerfile es un archivo de texto que Docker lee linea por linea para construir una imagen. Cada instruccion crea una capa nueva.

### FROM

Define la imagen base. Siempre es la primera instruccion.

```dockerfile
FROM ubuntu:latest
```

Sin `FROM` no hay imagen. Todo lo que haces despues se construye encima de esta base.

### RUN

Ejecuta un comando durante el build. Se usa para instalar paquetes, compilar codigo, crear directorios, etc. Cada `RUN` crea una nueva capa.

```dockerfile
RUN apt-get update
RUN apt-get upgrade -y
```

La flag `-y` responde "yes" automaticamente a confirmaciones interactivas. Sin ella, el build se queda esperando input y falla.

### WORKDIR

Establece el directorio de trabajo dentro del contenedor para todas las instrucciones que siguen (`RUN`, `COPY`, `CMD`). Lo crea si no existe.

```dockerfile
WORKDIR /app
```

Es mas seguro que hacer `RUN cd /app` porque `cd` en un `RUN` no persiste para la siguiente instruccion.

### COPY

Copia archivos desde el host (tu maquina) al sistema de archivos de la imagen.

```dockerfile
COPY requirements.txt .
COPY . .
```

El `.` final es la ruta destino dentro de la imagen, relativa al `WORKDIR` actual.

### ADD

Hace lo mismo que `COPY` pero ademas puede descomprimir archivos `.tar` y descargar desde URLs. La regla es: usa `COPY` siempre. Usa `ADD` solo si necesitas extraer un tarball directamente.

```dockerfile
# Use this only if you need to extract a tarball
ADD archive.tar.gz /app/
```

### EXPOSE

Documenta que puerto usa la aplicacion. No publica el puerto ni lo hace accesible desde fuera. Es solo metadata.

```dockerfile
EXPOSE 5000
```

Para que el puerto sea accesible necesitas `-p` en `docker run` o la seccion `ports` en docker-compose.

### ENV

Define una variable de entorno que estara disponible en el contenedor durante runtime.

```dockerfile
ENV APP_PORT=5000
ENV FLASK_ENV=production
```

### ARG

Define una variable que solo existe durante el build. No esta disponible cuando el contenedor corre.

```dockerfile
ARG DEBIAN_FRONTEND=noninteractive
```

Se usa para parametrizar el build sin contaminar el entorno del contenedor.

### VOLUME

Declara un punto de montaje para datos persistentes.

```dockerfile
VOLUME /app/data
```

### CMD

Define el comando por defecto que se ejecuta cuando el contenedor arranca. Puede ser sobreescrito completamente desde `docker run`.

```dockerfile
CMD ["echo", "Hello, World!"]
CMD ["python3", "app.py"]
```

---

## 3. CMD vs ENTRYPOINT

Esta distincion confunde mucho. La diferencia clave:

**CMD:** El comando por defecto. Si pasas argumentos en `docker run`, CMD se ignora completamente y lo que pasaste lo reemplaza.

**ENTRYPOINT:** El ejecutable fijo. No se reemplaza con `docker run`. Los argumentos que pases en `docker run` se suman como argumentos del entrypoint.

**Juntos (patron recomendado):** ENTRYPOINT es el ejecutable, CMD son los argumentos por defecto. El usuario puede cambiar los argumentos sin cambiar el ejecutable.

```dockerfile
# Only CMD: docker run my-image ls -la replaces echo completely
CMD ["echo", "Hello"]

# Only ENTRYPOINT: docker run my-image app.py runs python3 app.py
ENTRYPOINT ["python3"]

# Both: docker run my-image other.py runs python3 other.py (CMD replaced, ENTRYPOINT fixed)
ENTRYPOINT ["python3"]
CMD ["app.py"]
```

| Situacion                         | Que se ejecuta              |
|-----------------------------------|-----------------------------|
| Solo CMD, sin args                | CMD completo                |
| Solo CMD, con args en docker run  | Los args (CMD ignorado)     |
| Solo ENTRYPOINT, sin args         | ENTRYPOINT solo             |
| Solo ENTRYPOINT, con args         | ENTRYPOINT + args           |
| ENTRYPOINT + CMD, sin args        | ENTRYPOINT + CMD            |
| ENTRYPOINT + CMD, con args        | ENTRYPOINT + args (CMD ignorado) |

---

## 4. Build cache y capas

Cada instruccion en el Dockerfile crea una capa. Docker cachea cada capa. En el proximo build, si la instruccion y todo lo que la precede no cambiaron, Docker reutiliza la capa del cache en lugar de reejecutarla.

**Regla critica:** Si una capa cambia, todas las capas que siguen se invalidan y se reconstruyen aunque no hayan cambiado.

```dockerfile
# Bad order: if app.py changes, apt-get runs again (slow)
FROM ubuntu:latest
COPY . .
RUN apt-get update && apt-get install -y python3

# Good order: apt-get is cached unless FROM changes
FROM ubuntu:latest
RUN apt-get update && apt-get install -y python3
COPY . .
```

La regla practica: pon primero lo que cambia menos (instalar paquetes del sistema), deja para el final lo que cambia mas (tu codigo fuente).

---

## 5. Comandos utiles para operar contenedores

```bash
# Build an image from Dockerfile in current directory
docker build -t image-name:tag .

# Build specifying a Dockerfile path
docker build -f ./task0/Dockerfile -t softy-pinko:task0 .

# Build ignoring the cache (forces every layer to rebuild)
docker build -t image-name:tag . --no-cache

# Run a container (removes itself when stopped with --rm)
docker run -it --rm --name my-container image-name:tag

# Run in background (detached)
docker run -d --name my-container image-name:tag

# Run and map port 8080 on host to 5000 in container
docker run -p 8080:5000 image-name:tag

# Start or stop an existing (already created) container
docker start my-container
docker stop my-container

# List running containers
docker ps

# List all containers including stopped
docker ps --all

# List all local images
docker images

# Remove a stopped container
docker rm my-container

# Remove an image
docker rmi image-name:tag

# Remove all unused images (not referenced by any container)
docker image prune

# Show real-time CPU, memory, and network usage of containers
docker container stats

# Display system-wide Docker information (version, containers, images, etc.)
docker info
```

---

## 6. Depurar un contenedor que no arranca

Cuando un contenedor falla o no hace lo que esperas, estos son los pasos en orden:

```bash
# 1. See what the container printed before dying
docker logs container-name

# 2. Follow logs in real time
docker logs -f container-name

# 3. Open a shell inside a running container
docker exec -it container-name bash

# 4. If the container won't start, override CMD to open a shell instead
docker run -it --rm --entrypoint bash image-name:tag

# 5. See full container metadata (network, mounts, env vars, exit code)
docker inspect container-name

# 6. See resource usage
docker stats container-name
```

El flujo habitual: primero `docker logs` para ver el error, luego `docker exec -it bash` para explorar el sistema de archivos del contenedor si el problema no es obvio.

---

## 7. Docker Compose

Docker Compose permite definir y correr multiples contenedores juntos usando un archivo `docker-compose.yml`. En lugar de ejecutar varios `docker run` con flags largos, describes todo en YAML.

### Estructura basica

```yaml
services:
  backend:
    build: ./task1          # build from Dockerfile in this directory
    ports:
      - "5000:5000"         # host:container
    environment:
      - FLASK_ENV=production
    networks:
      - app-network

  frontend:
    image: nginx:latest     # use existing image instead of building
    ports:
      - "80:80"
    depends_on:
      - backend             # wait for backend to start first
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### Conceptos clave de Compose

**services:** Cada servicio es un contenedor. Puede construirse desde un Dockerfile (`build`) o usar una imagen existente (`image`).

**ports:** Mapeo `host:contenedor`. El puerto de la izquierda es el que abres en tu maquina. El de la derecha es el que escucha la aplicacion dentro del contenedor.

**networks:** Red virtual que conecta contenedores. Los contenedores en la misma red se pueden alcanzar por su nombre de servicio (no por IP). Por ejemplo, el frontend puede llamar a `http://backend:5000`.

**depends_on:** Le dice a Compose que espere a que el servicio listado este corriendo antes de arrancar este. No garantiza que la aplicacion dentro del contenedor este lista, solo que el contenedor inicio.

**environment:** Variables de entorno disponibles en runtime dentro del contenedor.

```bash
# Start all services (build if needed)
docker compose up

# Start in background
docker compose up -d

# Stop all services
docker compose down

# Rebuild images before starting
docker compose up --build

# Watch for file changes and sync them into running containers automatically
docker compose watch

# See logs of all services
docker compose logs

# See logs of one service
docker compose logs backend
```

### docker compose watch

Es el comando de desarrollo mas util. En lugar de reconstruir la imagen cada vez que cambias un archivo, `watch` detecta cambios en tu codigo y los sincroniza directamente al contenedor en ejecucion. El resultado es hot reload sin reiniciar el contenedor.

Para que funcione, cada servicio necesita una seccion `develop` en el `docker-compose.yml`:

```yaml
services:
  backend:
    build: ./backend
    develop:
      watch:
        - action: sync          # sync file changes into the container
          path: ./backend       # watch this local directory
          target: /app          # map it to this path inside the container
        - action: rebuild       # rebuild the image if these files change
          path: ./backend/requirements.txt
```

La diferencia con un volumen normal: `watch` es unidireccional (host hacia contenedor) y permite acciones distintas segun el archivo que cambie (sync para codigo, rebuild para dependencias).

---

## 8. Docker Hub

Docker Hub es el registro publico de imagenes de Docker. Es como GitHub pero para imagenes. Desde ahi se descargan las imagenes base (`ubuntu:latest`, `nginx:latest`, `python:3.11`) y tambien puedes publicar las tuyas.

### ¿Por qué existe un registro?

Sin un registro, cada imagen solo existe en tu maquina local. El registro permite compartir imagenes entre maquinas, equipos, y entornos de produccion. Cuando ejecutas `docker pull ubuntu:latest`, Docker busca esa imagen en Docker Hub y la descarga.

### Tagging de imagenes

Una imagen puede tener multiples tags. El tag es la version o variante de la imagen.

```bash
# Format: username/repository:tag
# If no tag is specified, Docker uses "latest" by default
docker build -t myusername/my-app:1.0 .
docker build -t myusername/my-app:latest .
```

### Flujo completo para publicar una imagen

```bash
# 1. Log in to Docker Hub
docker login -u your-username
# Docker will prompt for your password

# 2. Build and tag the image with your Docker Hub username
docker build -t your-username/image-name:tag .

# 3. Verify the image exists locally
docker images

# 4. Push to Docker Hub
docker push your-username/image-name:tag

# 5. Search for images on Docker Hub from the CLI
docker search image-name

# 6. Pull an image from Docker Hub
docker pull image-name:tag
```

### Imagenes oficiales vs de usuario

| Tipo | Formato | Ejemplo | Quien la mantiene |
|------|---------|---------|-------------------|
| Oficial | `nombre:tag` | `ubuntu:latest` | Docker / comunidad verificada |
| De usuario | `usuario/nombre:tag` | `myuser/my-app:1.0` | Tu cuenta |

Las imagenes oficiales no tienen prefijo de usuario. Son auditadas y mantenidas por Docker o por los proyectos oficiales.

---

## 9. Proxy directo vs proxy inverso

Son dos tipos de proxy y se confunden porque el nombre "proxy inverso" no es intuitivo.

**Proxy directo (forward proxy):** Se coloca del lado del cliente. El cliente le pide al proxy que haga la peticion en su nombre. El servidor no sabe quien es el cliente real. Se usa para anonimato, filtrado de contenido, o acceder a sitios bloqueados.

```
CLIENT --> [FORWARD PROXY] --> INTERNET / SERVER
(server sees the proxy, not the client)
```

**Proxy inverso (reverse proxy):** Se coloca del lado del servidor. El cliente le habla al proxy sin saber que hay servidores detras. El proxy redirige la peticion al servidor correcto. Se usa para seguridad, load balancing, y enrutar trafico entre multiples servicios.

```
CLIENT --> [REVERSE PROXY] --> SERVER(S)
(client sees the proxy, not the real servers)
```

| Caracteristica     | Proxy directo             | Proxy inverso              |
|--------------------|---------------------------|----------------------------|
| Protege a          | El cliente                | Los servidores             |
| Lo conoce          | El cliente (lo configura) | El servidor (lo instala)   |
| Caso de uso tipico | VPN, filtros de red       | Nginx, load balancer, CDN  |

En este proyecto nginx es un **proxy inverso**: el cliente hace peticiones a nginx en el puerto 80, y nginx decide internamente a que servicio enviar cada peticion (frontend o backend).

---

## 10. Nginx como reverse proxy y load balancer

En este proyecto, nginx actua como el punto de entrada unico. Recibe todas las peticiones y decide a donde enviarlas.

**Load balancer:** Cuando hay multiples instancias del mismo servidor, nginx distribuye las peticiones entre ellas usando un algoritmo.

**Round Robin:** El algoritmo por defecto de nginx. Peticion 1 va al servidor A, peticion 2 al servidor B, peticion 3 al servidor A, y asi sucesivamente. Cada servidor recibe la misma cantidad de peticiones.

```nginx
# Define a group of backend servers (load balancing pool)
upstream api_servers {
    server backend1:5000;   # service name from docker-compose
    server backend2:5000;
    # round robin is the default algorithm, no keyword needed
}

server {
    listen 80;

    # Route /api requests to the backend pool
    location /api {
        proxy_pass http://api_servers;
    }

    # Route everything else to the frontend
    location / {
        proxy_pass http://frontend:80;
    }
}
```

La directiva `proxy_pass` le dice a nginx a donde reenviar la peticion. El bloque `upstream` define el pool de servidores para el load balancing. Como no se especifica algoritmo, nginx usa Round Robin por defecto.

---

## 11. Redes en Docker

Cuando un contenedor necesita comunicarse con otro (por ejemplo, el backend hablando con la base de datos), necesitan estar en la misma red. Docker maneja esto con redes virtuales.

### Driver bridge

El driver por defecto para redes locales. Crea una red privada virtual entre los contenedores que la usan. Los contenedores en la misma red bridge se pueden ver entre si; los que no estan en la misma red no se pueden ver.

### DNS interno de Docker Compose

Esta es la parte mas importante para este proyecto: dentro de una red de Compose, cada contenedor es accesible por su **nombre de servicio**. Docker tiene un servidor DNS interno que resuelve esos nombres.

```yaml
services:
  backend:
    build: ./backend
    networks:
      - app-network

  proxy:
    build: ./proxy
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

Dentro del contenedor `proxy`, puedes hacer una peticion a `http://backend:5000` y Docker resuelve `backend` a la IP interna del contenedor backend. No necesitas saber la IP, solo el nombre del servicio.

**Por que esto importa:** En la configuracion de nginx, vas a escribir `proxy_pass http://backend:5000` usando el nombre del servicio, no una IP. Eso funciona porque nginx y el backend estan en la misma red de Compose.

### localhost dentro de un contenedor

`localhost` dentro de un contenedor apunta al propio contenedor, no a tu maquina host. Si el backend corre en el contenedor `backend` y quieres acceder desde el contenedor `proxy`, no usas `localhost:5000` -- usas `backend:5000`.

```
HOST MACHINE (localhost:8080)
        |
        | port mapping -p 8080:80
        |
   [proxy container] (localhost = this container only)
        |
        | http://backend:5000  <-- Docker DNS resolves "backend"
        |
   [backend container]
```

### Tipos de redes en Docker

| Driver  | Uso                                      |
|---------|------------------------------------------|
| bridge  | Red local entre contenedores (default)   |
| host    | El contenedor usa la red del host directamente |
| none    | Sin red                                  |

En este proyecto usamos `bridge` en todos los casos.

---

## Arquitectura final del proyecto

```
                    ┌─────────────────────────────────┐
                    │           CLIENT                │
                    └─────────────┬───────────────────┘
                                  │ HTTP :80
                    ┌─────────────▼───────────────────┐
                    │     NGINX (reverse proxy         │
                    │          + load balancer)        │
                    └──────┬──────────────┬────────────┘
                           │              │
              /api/*        │              │  /*
                    ┌───────▼──────┐  ┌───▼──────────┐
                    │  Round Robin │  │   FRONTEND   │
                    └──┬───────┬───┘  │  (static)    │
                       │       │      └──────────────┘
              ┌────────▼─┐ ┌───▼──────┐
              │ BACKEND 1│ │ BACKEND 2│
              │ (Flask)  │ │ (Flask)  │
              └──────────┘ └──────────┘
```


---

## 12. Herramientas de desarrollo: Docker Extension vs Container Tools

### Extension Docker (vieja) vs Container Tools (nueva)

La extension **Docker** de Microsoft en su version 2.0 se convirtio en un paquete que instala automaticamente **Container Tools**. Container Tools es la que realmente hace el trabajo hoy en dia.

| | Docker Extension (vieja) | Container Tools (nueva) |
|---|---|---|
| Soporte de motor | Solo Docker | Docker y Podman |
| Comandos en VSCode | `Docker: ...` | `Containers: ...` |
| Panel lateral | DOCKER | CONTAINERS |
| Estado | Reemplazada | Activa |

Si instalas la extension "Docker" de Microsoft hoy, obtienes Container Tools automaticamente. No son dos herramientas separadas que debas elegir -- una instala a la otra.

### Que puedes hacer desde el panel CONTAINERS

| Accion en el panel | Equivalente en terminal |
|---|---|
| Click derecho imagen > Run | `docker run image-name` |
| Click derecho contenedor > View Logs | `docker logs -f container-name` |
| Click derecho contenedor > Attach Shell (Exec) | `docker exec -it container-name bash` |
| Click derecho contenedor > Stop | `docker stop container-name` |
| Click derecho contenedor > Remove | `docker rm container-name` |
| Click derecho imagen > Remove | `docker rmi image-name` |

### Que es Podman

Podman es un motor de contenedores alternativo a Docker desarrollado por Red Hat. Hace lo mismo (construir y correr contenedores) pero con dos diferencias:

- **Sin daemon:** Docker necesita un proceso en segundo plano corriendo como root (`dockerd`). Podman no -- cada comando es un proceso independiente.
- **Rootless por defecto:** Podman puede correr contenedores sin privilegios de administrador, lo que es mas seguro en servidores de produccion.

Usa el mismo formato de imagenes (OCI), asi que las imagenes de Docker Hub funcionan en Podman sin cambios. Para este proyecto usamos Docker, pero Container Tools lo soporta por eso aparece mencionado.

---
---

# Ejercicios
## 0. Create Your First Docker Image
To create a Docker image, you will need to utilize a Dockerfile. Create a Dockerfile that:
- Is based on the latest ubuntu
- Update APT using apt-get update
- Upgrade currently installed software through APT using apt-get upgrade -y
- Once built, you can run the Docker image in a container and it will echo “Hello, World!” on the terminal.

### Codigo
#### Dockerfile
```Dockerfile
# Use the latest Ubuntu image as the base for this container
FROM ubuntu:latest
# Update the package index so apt knows about the latest available versions
RUN apt-get update
# Upgrade all installed packages to their latest versions
# -y auto-confirms all prompts so the build does not hang waiting for input
RUN apt-get upgrade -y
# Default command executed when the container starts
# Exec form (JSON array) runs the binary directly without a shell process
CMD ["echo", "Hello, World!"]

```

#### Logica

1. `docker build` lee el Dockerfile de arriba hacia abajo, instruccion por instruccion.
2. `FROM ubuntu:latest` -- Docker descarga la imagen base de Ubuntu desde Docker Hub si no la tiene en cache local. Todo lo que sigue se construye encima de esta capa.
3. `RUN apt-get update` -- Docker ejecuta ese comando dentro de un contenedor temporal y guarda el resultado como una nueva capa de solo lectura. El indice de paquetes de APT queda actualizado en esa capa.
4. `RUN apt-get upgrade -y` -- otro contenedor temporal, otro comando, otra capa. Los paquetes del sistema operativo quedan actualizados. El `-y` evita que el proceso se detenga esperando confirmacion del usuario, lo que romperia el build automatico.
5. `CMD ["echo", "Hello, World!"]` -- esto NO se ejecuta durante el build. Solo registra cual es el comando por defecto cuando el contenedor arranque. No crea una capa de datos.
6. `docker run` crea un contenedor a partir de la imagen construida y ejecuta el CMD. El proceso `echo` imprime `Hello, World!` en stdout y termina. El contenedor se detiene porque su proceso principal ya no existe.

#### Test
```
# 1. Build the image
docker build -f ./task0/Dockerfile -t softy-pinko:task0 .

# 2. Run a container from it
docker run -it --rm --name softy-pinko-task0 softy-pinko:task0
```


#### Output
```Bash
Hello, World!
```
#### Terminal
```bash
Dereks-MacBook-Pro:docker-project derekwebb$ docker build -f ./Dockerfile -t softy-pinko:task0 .
[+] Building 0.7s (7/7) FINISHED                                                                  
 => [internal] load build definition from Dockerfile                                         0.0s
 => => transferring dockerfile: 37B                                                          0.0s
 => [internal] load .dockerignore                                                            0.0s
 => => transferring context: 2B                                                              0.0s
 => [internal] load metadata for docker.io/library/ubuntu:latest                             0.6s
 => [1/3] FROM docker.io/library/ubuntu:latest@sha256:dfd64a3b4296d8c9b62aa3309984f8620b98d  0.0s
 => CACHED [2/3] RUN apt-get update                                                          0.0s
 => CACHED [3/3] RUN apt-get upgrade -y                                                      0.0s
 => exporting to image                                                                       0.0s
 => => exporting layers                                                                      0.0s
 => => writing image sha256:45d461b2a1471047589659e82af46202206be08b5b725d941a0a659b843a402  0.0s
 => => naming to docker.io/library/softy-pinko:task0                                         0.0s
Dereks-MacBook-Pro:docker-project derekwebb$ docker run -it --rm --name softy-pinko-task0 softy-pinko:task0
Hello, World!
Dereks-MacBook-Pro:docker-project derekwebb$
```

---

## 1. Back-end

Copia task0 como base e instala Python3, pip3 y Flask. Crea un servidor Flask con un endpoint `/api/hello` que retorna `Hello, World!` en el puerto 5252.

For this task, start by making a copy of your `task0` directory and name it task1. Next, we want to change the `Dockerfile` to install Python3, pip3, and Flask. You may not have used Flask, yet, but not to worry; for this project, we will give you all of the Flask code you need to get started. We’ll validate that all have been installed correctly by running a Flask server with one endpoint that when called returns “Hello, World!”

- Install `python3`, `python3-pip`, and `flask`
- Note: Make sure to automatically install and skip user input with the `-y` flag for `apt-get`
- Note: flask must be installed with `pip3`, **not through apt-get**

If you get a `This environment is externally managed` error when trying to install Python packages, add the following line before calling `pip` on your `Dockerfile`

`RUN rm /usr/lib/python*/EXTERNALLY-MANAGED`

- Locally, create a Python file named `api.py` and paste the following Python script - it uses Flask to create one endpoint that returns “Hello, World!” when called
- Hosting this Flask app on `0.0.0.0` instead of `127.0.0.1` means that it is reachable outside of the current machine (the current machine being a Docker container which is running inside of your laptop/desktop)
- Host this Flask app on port 5252

### Codigo
#### api.py
```python
from flask import Flask

app = Flask(__name__)

@app.route('/api/hello')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5252)
```

#### Dockerfile
```dockerfile
# Use the latest Ubuntu image as the base
FROM ubuntu:latest

# Update the package index
RUN apt-get update

# Upgrade all installed packages, -y skips confirmation prompts
RUN apt-get upgrade -y

# Install Python3 and pip3 in a single layer
RUN apt-get install -y python3 python3-pip

# Remove the externally-managed marker that blocks pip installs on modern Ubuntu
RUN rm /usr/lib/python*/EXTERNALLY-MANAGED

# Install Flask via pip3 (not apt-get, to get the latest version)
RUN pip3 install Flask

# Set the working directory inside the container
WORKDIR /app

# Copy the Flask app from the build context into the image
COPY ./api.py /app/api.py

# Start the Flask server when the container runs
CMD ["python3", "api.py"]
```

#### Logica

1. Las primeras tres capas son identicas a task0: imagen base, update y upgrade de APT.
2. `RUN apt-get install -y python3 python3-pip` instala el interprete de Python y el gestor de paquetes pip en una sola capa. Se combinan en un solo `apt-get install` porque instalar ambos en el mismo `RUN` evita crear una capa extra innecesaria.
3. `RUN rm /usr/lib/python*/EXTERNALLY-MANAGED` elimina un archivo que Ubuntu moderno usa para bloquear instalaciones de pip fuera de un entorno virtual. Sin este paso, `pip3 install` falla con un error de entorno gestionado externamente.
4. `RUN pip3 install flask` instala Flask usando pip, no apt-get. La razon: la version de Flask en apt-get suele estar desactualizada. pip siempre tiene la version mas reciente.
5. `WORKDIR /app` establece el directorio de trabajo. Si no existe, Docker lo crea.
6. `COPY ./api.py /app/api.py` copia el archivo Python desde el contexto de build al directorio de trabajo dentro de la imagen.
7. `CMD ["python3", "api.py"]` arranca el servidor Flask cuando el contenedor corre. Flask queda escuchando en `0.0.0.0:5252`, lo que significa que acepta conexiones desde cualquier interfaz de red, no solo localhost -- necesario para que el host pueda acceder al contenedor.
8. El flag `-p 5252:5252` en `docker run` mapea el puerto 5252 del host al puerto 5252 del contenedor. Sin esto, el servidor corre pero es inaccesible desde fuera del contenedor.

#### Test

```bash
# 1. Build the image
docker build -f ./task1/Dockerfile -t softy-pinko:task1 ./task1


# 2. Run the container with port mapping
docker run -p 5252:5252 -it --rm --name softy-pinko-task1 softy-pinko:task1

# 3. Test the endpoint from another terminal or browser
curl http://localhost:5252/api/hello
```

#### Output

```bash
Hello, World!
```

#### Terminal

```bash
juliangf94@DESKTOP-3UBL8O2$ docker run -p 5252:5252 -it --rm --name softy-pinko-task1 softy-pinko:task1
 * Serving Flask app 'api'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5252
 * Running on http://172.17.0.2:5252
Press CTRL+C to quit
172.17.0.1 - - [18/May/2026 13:44:35] "GET /api/hello HTTP/1.1" 200 -
 => naming to docker.io/library/softy-pinko:task1
Dereks-MacBook-Pro:task1 derekwebb$ docker run -p 5252:5252 -it --rm --name softy-pinko-task1 softy-pinko:task1
 * Serving Flask app 'api'
 * Debug mode: off
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5252
 * Running on http://172.17.0.2:5252
Press CTRL+C to quit
```

---

## 2. Front-end

En este task corren dos servidores en contenedores separados e independientes:

| Servidor | Puerto | Que sirve |
|---|---|---|
| Flask | 5252 | Datos (API) -- endpoint `/api/hello` |
| Nginx | 9000 | Pagina web estatica (HTML/CSS/JS) |

El navegador accede al frontend en `localhost:9000` para ver la pagina. En tasks posteriores el frontend va a hacer peticiones al backend en `localhost:5252` para obtener datos dinamicos. Por ahora los dos contenedores no se comunican entre si.

Agrega un servidor frontend con Nginx. La estructura del directorio cambia: el backend se mueve a `back-end/` y el frontend va en `front-end/`. Nginx sirve archivos estaticos clonados desde un repositorio externo en el puerto 9000.

```
task2/
├── back-end/
│   ├── Dockerfile
│   └── api.py
└── front-end/
    ├── Dockerfile
    ├── softy-pinko-front-end.conf
    └── softy-pinko-front-end/   (clonado de GitHub)
        ├── index.html
        └── assets/
```

### Codigo

#### softy-pinko-front-end.conf
```nginx
server {
    listen 9000;
    server_name localhost;

    location / {
        root /var/www/html/softy-pinko-front-end;
        index index.html;
    }
}
```

#### Dockerfile (front-end)
```dockerfile
# Use the official Nginx image as the base (already has Nginx installed)
FROM nginx:latest

# Copy the front-end static files into the directory Nginx will serve from
COPY ./softy-pinko-front-end /var/www/html/softy-pinko-front-end

# Replace the default Nginx config with our custom one
COPY ./softy-pinko-front-end.conf /etc/nginx/conf.d/default.conf
```

#### Logica

1. `FROM nginx:latest` usa la imagen oficial de Nginx en lugar de Ubuntu. Esta imagen ya tiene Nginx instalado y configurado para arrancar automaticamente -- no necesitas `RUN apt-get install nginx`.
2. `COPY ./softy-pinko-front-end /var/www/html/softy-pinko-front-end` copia todo el contenido del sitio al directorio donde Nginx va a buscarlo. `/var/www/html/` es la convencion estandar para archivos web en Linux.
3. `COPY ./softy-pinko-front-end.conf /etc/nginx/conf.d/default.conf` reemplaza la configuracion por defecto de Nginx con la nuestra. Nginx carga automaticamente todos los archivos `.conf` que esten en `/etc/nginx/conf.d/`.
4. En el `.conf`, `listen 9000` le dice a Nginx en que puerto escuchar. `root` le dice donde estan los archivos. `index` le dice cual archivo servir cuando se pide el directorio raiz.
5. No hay `CMD` en el Dockerfile porque la imagen de Nginx ya lo define internamente -- arranca el proceso de Nginx automaticamente.
6. `-p 9000:9000` en `docker run` mapea el puerto del host al puerto del contenedor, igual que en task1 con Flask.

#### Test

```bash
# 1. Build the frontend image (build context is ./front-end)
docker build -f ./task2/front-end/Dockerfile -t softy-pinko-front-end:task2 ./task2/front-end

# 2. Run the container
docker run -p 9000:9000 -it --rm --name softy-pinko-front-end-task2 softy-pinko-front-end:task2

# 3. Open in browser (nginx root ya es /var/www/html/softy-pinko-front-end, no repetir la carpeta en la URL)
curl http://localhost:9000/



```

#### Output

```bash
Configuration complete; ready for start up
```

#### Terminal

```bash
Dereks-MacBook-Pro:task2 derekwebb$ docker build -f ./front-end/Dockerfile -t softy-pinko-front-end:task2 ./front-end
[+] Building 0.6s (8/8) FINISHED
 => [1/3] FROM docker.io/library/nginx:latest
 => CACHED [2/3] COPY ./softy-pinko-front-end /var/www/html/softy-pinko-front-end
 => CACHED [3/3] COPY ./softy-pinko-front-end.conf /etc/nginx/conf.d/default.conf
 => naming to docker.io/library/softy-pinko-front-end:task2
Dereks-MacBook-Pro:task2 derekwebb$ docker run -p 9000:9000 -it --rm --name softy-pinko-front-end-task2 softy-pinko-front-end:task2
/docker-entrypoint.sh: Configuration complete; ready for start up
```

---

## 3. Connecting the Front-end and Back-end

Conecta el frontend con el backend. El frontend hace una peticion AJAX al backend para obtener datos dinamicos. Se necesita CORS en el backend para permitir peticiones entre origenes distintos (dos contenedores diferentes). Los dos contenedores siguen corriendo por separado, uno en cada terminal.

**Cambios respecto a task2:**
- `back-end/api.py`: agrega `flask-cors` para permitir peticiones cross-origin
- `back-end/Dockerfile`: instala `flask-cors` con pip3
- `front-end/softy-pinko-front-end/index.html`: agrega el `<h1>` dinamico y el script AJAX

### Que es CORS

CORS (Cross-Origin Resource Sharing) es un mecanismo de seguridad del navegador. Cuando el frontend en `localhost:9000` hace una peticion a `localhost:5252`, el navegador lo bloquea por defecto porque son origenes distintos (distinto puerto = distinto origen). `flask-cors` agrega los headers HTTP necesarios para que el navegador permita esa comunicacion.

### Codigo

#### back-end/api.py
```python
from flask import Flask
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

@app.route('/api/hello')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5252)
```

#### back-end/Dockerfile
```dockerfile
# Use the latest Ubuntu image as the base
FROM ubuntu:latest

# Update the package index
RUN apt-get update

# Upgrade all installed packages
RUN apt-get upgrade -y

# Install Python3 and pip3
RUN apt-get install -y python3 python3-pip

# Remove the externally-managed marker that blocks pip installs on modern Ubuntu
RUN rm /usr/lib/python*/EXTERNALLY-MANAGED

# Install Flask via pip3
RUN pip3 install Flask

# Install flask-cors to allow cross-origin requests from the frontend container
RUN pip3 install flask-cors

# Set the working directory
WORKDIR /app

# Copy the Flask app into the image
COPY ./api.py /app/api.py

# Start the Flask server
CMD ["python3", "api.py"]
```

#### front-end/index.html (cambios)
```html
<!-- Add before the existing h1 -->
<h1 id="dynamic-content"></h1>

<!-- Add as last script before </body> -->
<script>
    $(function() {
        $.ajax({
            type: "GET",
            url: "http://localhost:5252/api/hello",
            success: function(data) {
                console.log(data);
                $('#dynamic-content').text(data);
            }
        });
    });
</script>
```

#### Logica

1. El navegador carga la pagina desde Nginx en `localhost:9000`.
2. El script jQuery hace un GET a `http://localhost:5252/api/hello` (el backend Flask).
3. Sin CORS, el navegador bloquea esa peticion porque el origen de la pagina (`localhost:9000`) es distinto al destino de la peticion (`localhost:5252`).
4. `flask-cors` agrega el header `Access-Control-Allow-Origin: *` a todas las respuestas del backend, lo que le dice al navegador que permita la peticion.
5. Cuando Flask responde con `Hello, World!`, el script lo inserta como texto en el `<h1 id="dynamic-content">`.
6. El resultado es que el texto `Hello, World!` aparece en la pagina cargado dinamicamente desde el backend.

#### Test

```bash
# Terminal 1 -- backend
docker build -f ./task3/back-end/Dockerfile -t softy-pinko-back-end:task3 ./task3/back-end
docker run -p 5252:5252 -it --rm --name softy-pinko-back-end-task3 softy-pinko-back-end:task3

# Terminal 2 -- frontend
docker build -f ./task3/front-end/Dockerfile -t softy-pinko-front-end:task3 ./task3/front-end
docker run -p 9000:9000 -it --rm --name softy-pinko-front-end-task3 softy-pinko-front-end:task3

# Browser
http://localhost:9000/
```

#### Output

```
"Hello, World!" aparece en la pagina como contenido dinamico cargado desde el backend
```

#### Terminal

```bash
# Backend terminal
 * Serving Flask app 'api'
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5252
172.17.0.1 - - [19/May/2026] "GET /api/hello HTTP/1.1" 200 -

# Frontend terminal
Configuration complete; ready for start up
172.17.0.1 - - [19/May/2026] "GET / HTTP/1.1" 200 -
```

---

## 4. Making it Simpler with Docker Compose

En lugar de correr cada contenedor manualmente en terminales separadas, `docker-compose.yml` define todos los servicios en un solo archivo y los levanta con un comando.

**Cambio respecto a task3:** Solo se agrega el archivo `docker-compose.yml`. Los Dockerfiles del backend y frontend no cambian.

### Codigo

#### docker-compose.yml
```yaml
services:
  back-end:
    build:
      context: ./back-end
      dockerfile: Dockerfile
    image: softy-pinko-back-end:task4
    ports:
      - "5252:5252"

  front-end:
    build:
      context: ./front-end
      dockerfile: Dockerfile
    image: softy-pinko-front-end:task4
    ports:
      - "9000:9000"
    depends_on:
      - back-end
```

#### Logica

1. `services` define los contenedores que Compose va a manejar. Cada clave (`back-end`, `front-end`) es el nombre del servicio.
2. `build.context` le dice a Compose desde que directorio tomar el build context -- equivalente al `./task4/back-end` al final del `docker build`.
3. `build.dockerfile` especifica el nombre del Dockerfile dentro de ese context. Como se llama `Dockerfile` (el nombre por defecto), tecnicamente es opcional, pero es buena practica ser explicito.
4. `image` es el nombre y tag que tendra la imagen construida -- equivalente al `-t softy-pinko-back-end:task4` del `docker build`.
5. `ports` mapea puertos en formato `"HOST:CONTAINER"` -- equivalente al `-p 5252:5252` del `docker run`.
6. `depends_on` le dice a Compose que arranque `back-end` antes de `front-end`. No garantiza que Flask este listo para recibir peticiones, solo que el contenedor inicio.
7. Compose crea automaticamente una red compartida para todos los servicios del archivo. Los contenedores pueden comunicarse entre si usando el nombre del servicio como hostname.

#### Test

```bash
# Desde la raiz del repo, usar -f para apuntar al docker-compose.yml de task4
docker compose -f ./task4/docker-compose.yml build

# Levantar ambos servicios en foreground (Ctrl+C para detener)
docker compose -f ./task4/docker-compose.yml up

# En otra terminal, verificar backend (puerto 5252 expuesto en el compose)
curl http://localhost:5252/api/hello
# Esperado: Hello, World!

# Verificar frontend (puerto 9000 expuesto en el compose)
curl http://localhost:9000/
# Esperado: HTML de la pagina

# Detener y eliminar contenedores y red
docker compose -f ./task4/docker-compose.yml down
```

#### Output

```bash
# Los dos contenedores arrancan juntos
task4-back-end-1   |  * Running on all addresses (0.0.0.0)
task4-back-end-1   |  * Running on http://127.0.0.1:5252
task4-front-end-1  | Configuration complete; ready for start up
```

#### Terminal

```bash
Dereks-MacBook-Pro:task4 derekwebb$ docker-compose up
[+] Running 3/3
 => Network task4_default        Created
 => Container task4-back-end-1   Created
 => Container task4-front-end-1  Created
Attaching to task4-back-end-1, task4-front-end-1
task4-back-end-1   |  * Serving Flask app 'api'
task4-back-end-1   |  * Running on all addresses (0.0.0.0)
task4-back-end-1   |  * Running on http://127.0.0.1:5252
task4-front-end-1  | /docker-entrypoint.sh: Configuration complete; ready for start up
```

---

## 5. Proxy Server

Agrega un servidor proxy Nginx que actua como punto de entrada unico. El cliente solo habla con el proxy en el puerto 80. El proxy decide internamente si enrutar al frontend (puerto 9000) o al backend (puerto 5252). Los puertos del frontend y backend dejan de estar expuestos al host.

**Cambios respecto a task4:**
- Nueva carpeta `proxy/` con `Dockerfile` y `proxy.conf`
- `docker-compose.yml`: agrega servicio `proxy`, elimina mapeos de host del frontend y backend
- `index.html`: URL del AJAX cambia de `http://localhost:5252/api/hello` a `/api/hello`

**Por que cambia la URL del AJAX:** Antes el navegador llamaba directamente al backend. Ahora el proxy intercepta todo. Una URL relativa como `/api/hello` se resuelve contra el servidor que sirvio la pagina (el proxy en `localhost:80`), que luego la reenvía al backend.

### Codigo

#### proxy/proxy.conf
```nginx
server {
    listen 80;
    server_name localhost;

    # Route all non-API requests to the front-end container
    location / {
        proxy_pass http://front-end:9000;
    }

    # Route /api requests to the back-end container
    location /api {
        proxy_pass http://back-end:5252;
    }
}
```

#### proxy/Dockerfile
```dockerfile
# Use the official Nginx image as the base
FROM nginx:latest

# Replace the default Nginx config with the proxy configuration
COPY ./proxy.conf /etc/nginx/conf.d/default.conf
```

#### docker-compose.yml
```yaml
services:
  back-end:
    build:
      context: ./back-end
      dockerfile: Dockerfile
    image: softy-pinko-back-end:task5

  front-end:
    build:
      context: ./front-end
      dockerfile: Dockerfile
    image: softy-pinko-front-end:task5
    depends_on:
      - back-end

  proxy:
    build:
      context: ./proxy
      dockerfile: Dockerfile
    image: softy-pinko-proxy:task5
    ports:
      - "80:80"
    depends_on:
      - front-end
      - back-end
```

#### index.html (cambio)
```javascript
// Before (task4): called backend directly
url: "http://localhost:5252/api/hello",

// After (task5): goes through the proxy
url: "/api/hello",
```

#### Logica

1. El cliente abre `http://localhost:80` -- llega al contenedor `proxy`.
2. El proxy recibe la peticion `/` y la reenvía a `http://front-end:9000`. Docker resuelve `front-end` al IP interno del contenedor frontend.
3. Nginx del frontend sirve el `index.html`. El navegador lo recibe y ejecuta el script AJAX.
4. El AJAX hace GET a `/api/hello`. Como es una URL relativa, el navegador la resuelve contra el origen de la pagina: `http://localhost:80/api/hello`.
5. Esa peticion llega al proxy. El bloque `location /api` la coincide y la reenvía a `http://back-end:5252`.
6. Flask responde `Hello, World!`. El proxy lo devuelve al navegador, que lo inserta en el `<h1 id="dynamic-content">`.
7. Los puertos 5252 y 9000 ya no estan mapeados al host -- son inaccesibles directamente desde el navegador. Solo el puerto 80 del proxy esta abierto.

#### Test

```bash
# Desde la raiz del repo
docker compose -f ./task5/docker-compose.yml build

# Levantar los tres servicios
docker compose -f ./task5/docker-compose.yml up

# En otra terminal, verificar que el proxy enruta correctamente
curl http://localhost:80/api/hello
# Esperado: Hello, World!

curl http://localhost:80/
# Esperado: HTML de la pagina (status 200)

# Verificar que los puertos internos no estan expuestos al host (deben timeout)
curl --max-time 2 http://localhost:9000   # should fail
curl --max-time 2 http://localhost:5252   # should fail

# Detener
docker compose -f ./task5/docker-compose.yml down
```

#### Output

```bash
task5-back-end-1   |  * Running on all addresses (0.0.0.0)
task5-front-end-1  | Configuration complete; ready for start up
task5-proxy-1      | Configuration complete; ready for start up
task5-back-end-1   | "GET /api/hello HTTP/1.0" 200 -
```

#### Terminal

```bash
juliangf94@DESKTOP-3UBL8O2:/task5$ docker compose build
 Image softy-pinko-back-end:task5 Built
 Image softy-pinko-front-end:task5 Built
 Image softy-pinko-proxy:task5 Built

juliangf94@DESKTOP-3UBL8O2:/task5$ docker compose up -d
 Network task5_default Created
 Container task5-back-end-1 Started
 Container task5-front-end-1 Started
 Container task5-proxy-1 Started

juliangf94@DESKTOP-3UBL8O2:/task5$ curl http://localhost:80/api/hello
Hello, World!

juliangf94@DESKTOP-3UBL8O2:/task5$ curl -o /dev/null -w "%{http_code}" http://localhost:80/
200

# Verificacion de que los puertos internos no estan expuestos al host
juliangf94@DESKTOP-3UBL8O2:/task5$ curl --max-time 2 http://localhost:9000
curl: (28) Connection timed out
juliangf94@DESKTOP-3UBL8O2:/task5$ curl --max-time 2 http://localhost:5252
curl: (28) Connection timed out
```

---

## 6. Scale Horizontally

En lugar de modificar el codigo, se escala el servicio `back-end` a 2 instancias con un flag en el comando. El proxy nginx ya usa el nombre de servicio `back-end` para enrutar -- cuando hay multiples contenedores con ese nombre, el DNS interno de Docker hace round-robin automaticamente entre ellos.

**Cambio respecto a task5:** Solo el tag de las imagenes cambia a `:task6`. Se agrega el archivo `2-api-servers.txt` con el comando exacto para levantar 2 backends.

**Por que funciona sin cambiar nginx.conf:** Cuando Compose crea 2 instancias de `back-end`, Docker registra ambas IPs bajo el mismo nombre DNS `back-end`. Nginx resuelve ese nombre y obtiene una IP distinta en cada peticion, alternando entre los dos contenedores en Round Robin.

### Codigo

#### docker-compose.yml
```yaml
services:
  back-end:
    build:
      context: ./back-end
      dockerfile: Dockerfile
    image: softy-pinko-back-end:task6

  front-end:
    build:
      context: ./front-end
      dockerfile: Dockerfile
    image: softy-pinko-front-end:task6
    depends_on:
      - back-end

  proxy:
    build:
      context: ./proxy
      dockerfile: Dockerfile
    image: softy-pinko-proxy:task6
    ports:
      - "80:80"
    depends_on:
      - front-end
      - back-end
```

#### 2-api-servers.txt
```
docker compose up --scale back-end=2
```

#### Logica

1. `--scale back-end=2` le dice a Compose que cree 2 contenedores para el servicio `back-end` en lugar de 1.
2. Los contenedores se nombran `task6-back-end-1` y `task6-back-end-2` automaticamente.
3. El DNS interno de Docker registra ambos contenedores bajo el nombre `back-end`. Cada resolucion de ese nombre retorna una IP diferente en rotacion.
4. Nginx en el proxy usa `proxy_pass http://back-end:5252`. Cada vez que nginx resuelve `back-end`, obtiene la IP del siguiente contenedor -- Round Robin implementado por el DNS de Docker, no por nginx.
5. Al recargar la pagina varias veces, los logs muestran que las peticiones alternan entre `task6-back-end-1` y `task6-back-end-2`.

#### Test

```bash
# Desde la raiz del repo
docker compose -f ./task6/docker-compose.yml build

# Levantar con 2 instancias del backend
docker compose -f ./task6/docker-compose.yml up --scale back-end=2

# En otra terminal, enviar varias requests y observar los logs
curl http://localhost:80/api/hello
curl http://localhost:80/api/hello
curl http://localhost:80/api/hello
# Los logs deben mostrar que las requests alternan entre back-end-1 y back-end-2

# Detener
docker compose -f ./task6/docker-compose.yml down
```

#### Output

```bash
task6-back-end-1   | "GET /api/hello HTTP/1.0" 200 -
task6-back-end-2   | "GET /api/hello HTTP/1.0" 200 -
task6-back-end-1   | "GET /api/hello HTTP/1.0" 200 -
task6-back-end-2   | "GET /api/hello HTTP/1.0" 200 -
```

#### Terminal

```bash
[+] Running 4/0
 => Container task6-back-end-2   Created
 => Container task6-back-end-1   Created
 => Container task6-front-end-1  Created
 => Container task6-proxy-1      Created
Attaching to task6-back-end-1, task6-back-end-2, task6-front-end-1, task6-proxy-1
task6-back-end-2   |  * Running on all addresses (0.0.0.0)
task6-back-end-1   |  * Running on all addresses (0.0.0.0)
task6-front-end-1  | Configuration complete; ready for start up
task6-proxy-1      | Configuration complete; ready for start up
```
