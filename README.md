# Taller de Docker — Crear y compartir imágenes

Documentación del taller práctico de Docker donde aprendimos a crear imágenes propias, orquestar servicios con Docker Compose y publicar imágenes en Docker Hub.

---

## Requisitos previos

- Docker Desktop instalado y corriendo
- PowerShell
- Cuenta en [Docker Hub](https://hub.docker.com)

---

## Parte 1 — Mi primer Dockerfile (helloapp)

El objetivo de esta parte es entender la estructura básica de un Dockerfile y cómo se construye una imagen desde cero.

### Creación del build context

```powershell
mkdir "$HOME\Sites\hello-world"
cd "$HOME\Sites\hello-world"
"hello" | Out-File -Encoding utf8 hello
```

### Dockerfile

```dockerfile
FROM busybox
COPY /hello /
RUN cat /hello
```

| Instrucción | Qué hace |
|-------------|----------|
| `FROM busybox` | Parte de una imagen base ultraligera |
| `COPY /hello /` | Copia el archivo `hello` dentro de la imagen |
| `RUN cat /hello` | Durante la construcción imprime el contenido del archivo |

### Construcción de la imagen

```powershell
docker build -t helloapp:v1 .
docker images
```

**Lista de imágenes con helloapp:v1**

<img width="1600" height="772" alt="image" src="https://github.com/user-attachments/assets/a0085068-0fe1-44d1-addf-c0c548ecad76" />


---

## Parte 2 — Aplicación Python con Flask (friendlyhello)

Creamos una imagen de una aplicación web real. La app muestra un mensaje de bienvenida y un contador de visitas que usa Redis.

### Estructura del proyecto

```
friendlyhello/
├── app.py
├── requirements.txt
└── Dockerfile
```

### app.py

```python
from flask import Flask
from redis import Redis, RedisError
import os
import socket

redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)
app = Flask(__name__)

@app.route("/")
def hello():
    try:
        visits = redis.incr("counter")
    except RedisError:
        visits = "<i>cannot connect to Redis, counter disabled</i>"

    html = "<h3>Hello {name}!</h3><b>Hostname:</b> {hostname}<br/><b>Visits:</b> {visits}"
    return html.format(name=os.getenv("NAME", "world"), hostname=socket.gethostname(), visits=visits)

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=80)
```

### requirements.txt

```
Flask
Redis
```

### Dockerfile

```dockerfile
FROM python:3-slim
WORKDIR /app
COPY . /app
RUN pip install --trusted-host pypi.python.org -r requirements.txt
EXPOSE 80
ENV NAME World
CMD ["python", "app.py"]
```

| Instrucción | Qué hace |
|-------------|----------|
| `FROM python:3-slim` | Imagen base oficial de Python |
| `WORKDIR /app` | Establece la carpeta de trabajo dentro del contenedor |
| `COPY . /app` | Copia todos los archivos del proyecto al contenedor |
| `RUN pip install ...` | Instala las dependencias de Python |
| `EXPOSE 80` | Documenta que la app escucha en el puerto 80 |
| `CMD` | Comando que se ejecuta al arrancar el contenedor |

### Construcción y verificación

```powershell
docker build -t friendlyhello .
docker images
```

**Lista de imágenes con friendlyhello**

<img width="1600" height="847" alt="image" src="https://github.com/user-attachments/assets/ede5e643-9bb6-4df2-ba72-cba276279336" />


### Prueba del contenedor

```powershell
docker run --rm -p 4000:80 friendlyhello
```

Se abre `http://localhost:4000` en el navegador. En este punto el contador aparece deshabilitado porque no hay Redis disponible.

**App corriendo sin Redis**

<img width="1600" height="635" alt="image" src="https://github.com/user-attachments/assets/249663f7-7911-4d73-85d7-c7920ded0c1f" />



---

## Parte 3 — Docker Compose con Redis

Docker Compose permite orquestar varios contenedores como si fueran un solo servicio. En esta parte levantamos la app junto a una base de datos Redis.

### docker-compose.yaml

```yaml
services:
    web:
        build: .
        ports:
            - "4000:80"
    redis:
        image: redis
        ports:
            - "6379:6379"
        volumes:
            - "./data:/data"
        command: redis-server --appendonly yes
```

La clave aquí es la diferencia entre `build: .` (construye la imagen desde el Dockerfile local) e `image: redis` (descarga la imagen oficial de Docker Hub).

### Levantar la aplicación

```powershell
docker compose up
```

Al abrir `http://localhost:4000` el contador de visitas ya funciona y sube con cada recarga.

**App con contador funcionando**

<img width="1600" height="691" alt="image" src="https://github.com/user-attachments/assets/ecc693bc-e2a4-49ac-a59c-3bbe7911924b" />


```powershell
docker compose down
```

---

## Parte 4 — Balanceo de carga con Traefik

Escalamos el servicio web a 5 instancias y añadimos Traefik como balanceador de carga. Traefik reparte las peticiones entre los contenedores automáticamente.

### docker-compose.yaml con Traefik

```yaml
services:
    web:
        build: .
        labels:
        - "traefik.enable=true"
        - "traefik.http.routers.web.rule=Host(`localhost`)"
        - "traefik.http.routers.web.entrypoints=web"
        - "traefik.http.services.web.loadbalancer.server.port=80"
    redis:
        image: redis
        volumes:
            - "./data:/data"
        command: redis-server --appendonly yes
        labels:
        - "traefik.enable=false"
    traefik:
        image: traefik:v2.3
        command:
        - "--log.level=DEBUG"
        - "--api.insecure=true"
        - "--providers.docker=true"
        - "--providers.docker.exposedByDefault=false"
        - "--entrypoints.web.address=:4000"
        ports:
        - "4000:4000"
        - "8080:8080"
        volumes:
        - "/var/run/docker.sock:/var/run/docker.sock"
        labels:
        - "traefik.enable=true"
```

### Escalar a 5 instancias

```powershell
docker compose up -d --scale web=5
docker ps
```

**5 réplicas del servicio web**

<img width="1600" height="471" alt="image" src="https://github.com/user-attachments/assets/c43669bb-91ad-40f5-842a-ebf1617a40e5" />


```powershell
docker compose down
```

---

## Parte 5 — Publicar la imagen en Docker Hub

Para compartir una imagen con otros es necesario subirla a un registro. Usamos Docker Hub, el registro público oficial de Docker.

### Pasos

```powershell
# Iniciar sesión en Docker Hub
docker login

# Construir la imagen con el namespace del usuario
docker build -t TUUSUARIO/friendlyhello -t TUUSUARIO/friendlyhello:1.0.0 .

# Subir la imagen
docker push TUUSUARIO/friendlyhello
```

La imagen queda disponible públicamente en `hub.docker.com/r/TUUSUARIO/friendlyhello`.

**Imagen publicada en Docker Hub**

<img width="1600" height="760" alt="image" src="https://github.com/user-attachments/assets/cb89daae-37a8-49ba-94ea-167468b43161" />
<img width="1600" height="732" alt="image" src="https://github.com/user-attachments/assets/296a5ae2-bcf5-400f-b399-766c20ffd999" />



---

## Parte 6 — Ejercicios finales

### Ejercicio 1 — Usar la imagen publicada en Compose

En lugar de hacer `build` desde el código local, el `docker-compose.yaml` ahora apunta directamente a la imagen publicada en Docker Hub.

```yaml
services:
    web:
        image: TUUSUARIO/friendlyhello
        ports:
            - "4000:80"
    redis:
        image: redis
        ports:
            - "6379:6379"
        volumes:
            - "./data:/data"
        command: redis-server --appendonly yes
```

```powershell
docker compose up
```

**docker-compose.yaml con imagen propia**

<img width="1600" height="551" alt="image" src="https://github.com/user-attachments/assets/81c57efa-dd33-4c8b-ad36-77a9af662b85" />


**App corriendo con imagen publicada**

<img width="1600" height="302" alt="image" src="https://github.com/user-attachments/assets/f8788093-7bab-4790-9443-5666610e07b3" />


---

### Ejercicio 2 — Usar la imagen de un compañero

Cambiamos la imagen en el `docker-compose.yaml` para usar la imagen publicada por otro compañero.

```yaml
services:
    web:
        image: USUARIOCOMPANERO/friendlyhello
        ports:
            - "4000:80"
    redis:
        image: redis
        ports:
            - "6379:6379"
        volumes:
            - "./data:/data"
        command: redis-server --appendonly yes
```

```powershell
docker compose down
docker compose up
```

Docker descarga automáticamente la imagen del compañero desde Docker Hub. Al abrir `http://localhost:4000` el Hostname es diferente al de nuestra propia imagen, lo que confirma que estamos corriendo el contenedor del compañero.

**App corriendo con imagen del compañero**

<img width="1600" height="465" alt="image" src="https://github.com/user-attachments/assets/f4bb883b-9a1c-426b-8992-239dbe25045c" />


