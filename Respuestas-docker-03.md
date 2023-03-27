---
marp: true
title: Apuntes IISS
description: Apuntes de Implementación e Implantación de Sistemas Software
---

<!-- size: 16:9 -->
<!-- theme: default -->

<!-- paginate: false -->

<!-- headingDivider: 1 -->

<style>
h1 {
  text-align: center;
}
img[alt~="center"] {
  display: block;
  margin: 0 auto;
}
</style>

# DOCKER - Parte 3

![width:480 center](img/docker-010.png)

---

## Recordatorio...


`docker ps` Contenedores creados en ejecución

`docker ps -a` Contenedores creados (incluye los que están parados)

`docker container ls`  (con o sin -a)

`docker run NOMBRE_IMAGEN` Crear un contenedor (si no tiene la imagen en local, la descarga)

`docker image ls` Ver las imágenes disponibles en local 


Docker da un nombre aleatorio a los contenedores si no lo elegimos nosotros. Todos los contenedores tienen un ID

```console
$ docker ps -a
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS                      PORTS     NAMES
c8bb700e5f39   hello-world              "/hello"                 5 seconds ago   Exited (0) 4 seconds ago              busy_haibt
efe18fd91e7b   dodero/myserver:latest   "/bin/sh -c 'apache2…"   20 hours ago    Exited (130) 20 hours ago             myserver
```

---

`docker rm busy_haibt` Para borrar contenedores

`docker rm c8b` (no hace falta el ID completo)

Elegir el nombre del contenedor (debe ser único en local):

`docker run --name=hola hello-world`

Crear un contenedor asignando un puerto random en local:

`docker run -d -P --name=apache-server bitnami/apache`

`docker ps -a` (para ver puerto asignado)

Ejecutar terminal en el contenedor creado:

`docker exec -it apache-server /bin/bash`

---

Construir una imagen nueva

```bash
mkdir mi-apache
nano index.html
mv index.html mi-apache
nano Dockerfile
```

Dockerfile:

 ```dockerfile
FROM bitnami/apache
COPY index.html /opt/bitnami/apache/htdocs/index.html
```

```bash
docker build -t usuario/mi-apache .
docker run -d -P --name=mi-apache-1 usuario/mi-apache
```

Subir nueva imagen a Docker Hub:

`docker push usuario/mi-apache`

---

Contenedor que usa directorio enlazado (bind):

```bash
docker run -d -P --name=apache-bind-1 \
   --mount type=bind,source=/Users/Usuario/directorio-compartido,target=/app bitnami/apache
```

Contenedor que usa volumen:

```bash
docker run -d -P --name=apache-volume-1 \
   --mount type=volume,source=vol-apache,target=/app bitnami/apache
```

Borrando volumen (se deben borrar también los contenedores que lo usen):

`docker rm apache-volume-1`

---

Crear una nueva red `opencart-network`:

`docker network create opencart-network`

Contenedor que usa una red específica:

```bash
docker run -d --name=mariadb \ 
  -e ALLOW_EMPTY_PASSWORD=yes \
  -e MARIADB_USER=bn_opencart \
  -e MARIADB_DATABASE=bitnami_opencart \
  --net=opencart-network bitnami/mariadb
```

Conectar un contenedor a una red:

`docker network connect opencart-network mariadb`

---

## Docker compose

Configurar múltiples contenedores como servicios cuando necesitamos:

- Arquitectura que involucra más de un componente/servicio funcionando al mismo tiempo
- Pre-configurar para que estos contenedores trabajen juntos

### Características

- Permite desplegar los contenedores en un único host (limitación).
- En el caso de que se quiera desplegar en un cluster hay que utilizar [Docker Swarm](https://www.ionos.es/digitalguide/servidores/know-how/docker-compose-y-swarm-gestion-multicontenedor/)

Nosotros solo trabajaremos con Docker Compose...

---

## Fichero `docker-compose.yml`

Secciones principales del fichero Docker Compose que vamos a utilizar:

- **Version**: especifica la versión del fichero Docker Compose, si no se establece por defecto es la 1.
- **Services**: especifica los servicios de su aplicación, es decir, los contenedores que van a ejecutarse.
- **Networks**: especifica la configuración de red que van a utilizar los servicios.
- **Volumes**: especifica los volúmenes que van a utilizar los servicios para guardar/leer la información.

---

### Ejemplo inicial con Docker compose

Vamos a desplegar una aplicación web hecha con Python/Flask que devuelva en JSON el nombre del usuario

1. Creamos una carpeta donde añadir los siguientes ficheros:

- `requirements.txt`
- `app.py`
- `Dockerfile`
- `docker-compose.yml`

```console
$ mkdir docker-compose-ejemplo1
```

---

2. Creamos el fichero “requirements.txt”

```txt
Flask
Redis<3.0.0
```

3. Creamos el código de la aplicación Python `app.py`:

```python
from flask import Flask, request, jsonify
from redis import Redis
app = Flask(__name__)
redis = Redis(host="redis", db=0, socket_timeout=5, charset="utf-8", decode_responses=True)
@app.route('/', methods=['POST', 'GET'])
def index():
    if request.method == 'POST':
        name = request.json['name']
        redis.rpush('students', {'name': name})
        return jsonify({'name': name})
    if request.method == 'GET':
        return jsonify(redis.lrange('students', 0, -1))
```

---

4. Creamos el fichero `Dockerfile`

```dockerfile
FROM python:3.7.0-alpine3.8
WORKDIR /usr/src/app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
ENV FLASK_APP=app.py
CMD flask run --host=0.0.0.0
```

---

5. Creamos el fichero `docker-compose.yml`

```yml
version: '3'
services:
  app:
    build: .
    image: dodero/flask-redis:1.0
    environment:
      - FLASK_ENV=development
    ports:
      - 5000:5000
  redis:
    image: redis:4.0.11-alpine
```

---

Ficheros necesarios en la carpeta:

```console
$ ls -l docker-compose-ejemplo1/
total 32
-rw-r--r--  1 dodero  staff  184 11 mar 09:19 Dockerfile
-rw-r--r--  1 dodero  staff  481 11 mar 09:19 app.py
-rw-r--r--  1 dodero  staff  196 11 mar 09:19 docker-compose.yml
-rw-r--r--  1 dodero  staff   17 11 mar 09:15 requirements.txt
```

Cambiarse a la carpeta: `cd ~/docker-compose-ejemplo1`

Ejecutar Docker Compose:

`docker-compose up`  (añadir `-d` para ejecutar en background)

**NOTA**: docker-compose está instalado en Windows y MacOS por defecto. En linux hay que instalarlo manualmente.

---

Con lo anterior hemos arrancado un contenedor (servicio) `app` con la aplicación en Python y otro contenedor (servicio) `redis` con una base de datos _redis_.

```console
$ docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS         PORTS                    NAMES
ff4292697a4c   dodero/flask-redis:1.0   "/bin/sh -c 'flask r…"   5 minutes ago   Up 5 minutes   0.0.0.0:5000->5000/tcp   docker-compose-ejemplo1-app-1
dc5a0762ce04   redis:4.0.11-alpine      "docker-entrypoint.s…"   5 minutes ago   Up 5 minutes   6379/tcp                 docker-compose-ejemplo1-redis-1
```

---

Probando el ejemplo (añadiendo nuevo estudiante):

```console
$ curl --header "Content-Type: application/json" \
   --request POST \
   --data '{"name":"Usuario"}' \
   localhost:5000
```

Probando el ejemplo (obteniendo los estudiantes creados): Acceder a `localhost:5000` en el navegador o ejecutando `curl localhost:5000`

```json
[
    "{'name': 'Usuario'}"
]
```

---

## Gestionando aplicaciones con múltiples contenedores

Parar el servicio

`docker-compose down`

Visualizar la configuración de la red creada

`docker network ls`

`docker network inspect docker-compose-ejemplo1_default`

Procesos que están ejecutándose:

`docker-compose ps`

`docker-compose top`

---

Ver los logs generados por estos procesos:

`docker-compose logs`

Ejecutar un comando en un contenedor iniciado:

`docker-compose exec app ps`

Subir las imágenes de los servicios a DockerHub:

`docker-compose push app`

`docker-compose push redis`

Actualizar las imágenes de los servicios:

`docker-compose pull app`

`docker-compose pull redis`

---

Actualización del fichero `docker-compose.yml`:

```yml
version: '3'
services:
  app:
    build: 
      context: .
      args:
        - IMAGE_VERSION=3.7.0-alpine3.8
    image: dodero/flask-redis:1.0
    environment:
      - FLASK_ENV=development
    ports:
      - 80:5000
  redis:
    image: redis:4.0.11-alpine
```

Ejecutamos `docker-compose up -d` y sólo se regenera lo necesario (docker-compose mantiene los estados)

---

### Escalar servicios

Se pueden arrancar varias instancias de un mismo servicio para así escalarlo

Escalar un servicio con Docker Compose:

`docker-compose up -d --scale app=3` Falla, ¿por qué?

```console
$ docker-compose up -d --scale app=3
[+] Running 2/4
 ⠿ Container docker-compose-ejemplo1-redis-1  Running                                                                 0.0s
 ⠿ Container docker-compose-ejemplo1-app-3    Running                                                                 0.0s
 ⠹ Container docker-compose-ejemplo1-app-1    Starting                                                                0.2s
 ⠹ Container docker-compose-ejemplo1-app-2    Starting                                                                0.2s
Error response from daemon: driver failed programming external connectivity on endpoint docker-compose-ejemplo1-app-2
(43f832ac334345e85b60a65851c9ac76d250e3498e0699469e066dddd8655ba6): Bind for 0.0.0.0:80 failed: port is already allocated
```

---

Actualizamos el fichero `docker-compose.yml` eliminando la redirección de puertos:

```yml
version: '3'
services:
    app:
      build:
        context: .
        args:
          - IMAGE_VERSION=3.7.0-alpine3.8
      image: dodero/flask-redis:1.0
      environment:
        - FLASK_ENV=development
      ports:
        - “80-83:5000”
    redis:
      image: redis:4.0.11-alpine
```

Intentamos escalar el servicio de nuevo:

`docker-compose up -d --scale app=3`

---

Había un conflicto de puertos porque todas las réplicas querían usar el mismo.

Si persisten los conflictos de puertos, comprobar qué puertos están ocupados y cerrar todos los procesos que ocupen los puertos 80-83:

`sudo lsof -i -P | grep LISTEN`

Abrir en el navegador `localhost:80`, `localhost:81`, etc.

Probar a ejecutar varias veces el siguiente comando con distintos datos (`'{"name":"Ana"}'`, `'{"name":"Belén"}'`, etc.) y distintos endpoints (`localhost:80`, `localhost:81`, `localhost:82`). ¿Se comparten o no los valores almacenados en `redis`? ¿Y si lanzáramos varias instancias del contenedor `redis`?

```bash
curl --header "Content-Type: application/json" \
   --request POST \
   --data '{"name":"Usuario"}' \
   localhost:80
```

---

## Usando volúmenes con Docker Compose

Usando memoria persistente en los contenedores

---

### Ejemplo de uso de volumen con Docker Compose

Configuramos el volumen en el archivo `docker-compose.yml`

```yml
version: '3'
services:
  app:
    build: .
    image: dodero/flask-redis:1.0
    environment:
      - FLASK_ENV=development
    ports:
      - 80:5000
  redis:
    image: redis:4.0.11-alpine
    volumes:
      - mydata:/data
volumes:
  mydata:
```

---

Ejecutamos docker-compose

`docker-compose down`

`docker-compose up -d`

Consultamos el volumen creado

`docker volume ls`

---

## Configuraciones de red con Docker Compose

Conceptos básicos de red con Docker Compose

---

### Ejemplo de configuración de red con Docker Compose

Configuramos la red en el archivo `docker-compose.yml`

```yml
version: '3'
services:
  app:
    build: .
    image: dodero/flask-redis:1.0
    environment:
      - FLASK_ENV=development
    ports:
      - 80:5000
    networks:
      - mynet
  redis:
    image: redis:4.0.11-alpine
    networks:
      - mynet
networks:
  mynet:
```

---

Ejecutamos docker-compose

`docker-compose down`

`docker-compose up -d`

Ver la configuración de la red creada

`docker network inspect docker-compose-ejemplo1_mynet`

---

## Práctica 3

_(Tarea disponible en el Campus Virtual)_

Subir el fichero `docker-compose.yml` creado y todo lo necesario, y además, un documento PDF con la explicación de la configuración realizada en el fichero. Incluir también las capturas donde se requiera mostrar en el navegador el correcto funcionamiento de los contenedores.

---

### Parte 1

1) Crear un fichero `docker-compose.yml` con dos servicios: drupal + mysql.

El contenido del fichero `docker-compose.yml` será este:
```
version: '3.9'

services:
  db:
    image: mysql:latest
    volumes:
      - volumenDocker:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: example

  drupal:
    image: drupal:latest
    ports:
      - "81:80"
    volumes:
      - volumenDocker:/var/www/html
    depends_on:
      - db

volumes:
  volumenDocker:
```

2) Hacer que el servicio drupal utilice el puerto 81.
Ya está especificado arriba. 
```
ports:
  - "81:80"
```
3) Hacer que ambos contenedores usen un volumen `volumenDocker`.
Ya hemmos hecho que ambos contenedores usen el volumen `volumenDocker`.
```
volumes:
  volumenDocker:

```
4) Comprobar que puede acceder a `localhost:81` y puede visualizar la página de configuración de drupal.

Antes de comprobarlo, tenemos que poner este comando en la terminal para poner los contenedores en ejecución:
```
docker-compose up -d
```

Comprobamos que funciona:

![imagen](.\Docker-03-Drupal.png)

Para parar los contenedores, usamos el siguiente comando:
```
docker-compose down
```


---

### Parte 2

1) Crear un fichero `docker-compose.yml` con dos servicios: wordpress +  mariadb.

El contenido del fichero `docker-compose.yml` será este:

```
version: '3'

services:
  db:
    image: mariadb
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress
      MYSQL_ROOT_PASSWORD: root
    networks:
      - redDocker
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    depends_on:
      - db
    image: wordpress
    restart: always
    ports:
      - "82:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress
      WORDPRESS_DB_NAME: wordpress
    networks:
      - redDocker
    volumes:
      - wp_data:/var/www/html

networks:
  redDocker:

volumes:
  db_data:
  wp_data:
```

2) Hacer que el servicio wordpress utilice el puerto 82.

Ya lo hemos especificado en el apartado anterior:

```
   ports:
      - "82:80"
```
3) Hacer que ambos contenedores usen la red `redDocker`.

Ya se especifica en el apartado 1:
```
networks:
  redDocker:
```
4) Comprobar que puede acceder a `localhost:82` y puede visualizar la página de configuración de wordpress.
Antes de comprobarlo, tenemos que poner este comando en la terminal para poner los contenedores en ejecución:
```
docker-compose up -d
```
Comprobamos que funciona:
![imagen](.\Docker-03-Wordpress.png)

Paramos los contenedores con el siguiente comando:
```
docker-compose down
```


---

## Task 3

_(Task available in the Virtual Campus)_

Upload the file `docker-compose.yml` created and everything needed, and also a PDF document with the explanation of the configuration made in the file. Include also the screenshots where it is required to show in the browser the correct functioning of the containers.

---

### Part 1

1) Create a `docker-compose.yml` file with two services: drupal + mysql.
2) Make the drupal service use port 81.
3) Make both containers use a `volumeDocker`.
4) Check that you can access `localhost:81` and you can see the drupal configuration page.

---

### Part 2

1) Create a `docker-compose.yml` file with two services: wordpress + mariadb.
2) Make the wordpress service use port 82.
3) Make both containers use the `networkDocker` network.
4) Check that you can access `localhost:82` and can see the wordpress configuration page.

