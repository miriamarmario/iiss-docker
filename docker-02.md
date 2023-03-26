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

# DOCKER - Parte 2

![width:480 center](img/docker-010.png)

---

## Recordatorio...


`docker ps` Contenedores creados en ejecución

`docker ps -a` Contenedores creados (incluye los que están parados)

`docker container ls`  (con o sin -a)

`docker run NOMBRE_IMAGEN` Crear un contenedor (si no tiene la imagen en local, la descarga)

`docker image ls` Ver las imágenes disponibles en local 


Docker da un nombre aleatorio a los contenedores si no lo elegimos nosotros. Todos los contenedores tienen un ID

```bash
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
```docker
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

## Directorios enlazados y volúmenes

Para compartir información con Docker cuando necesitamos:

- Compartir un __directorio__ entre múltiple contenedores.
- Compartir __un directorio__ entre el host y un contenedor.
- Compartir __un fichero__ entre el host y un contenedor.

Opciones:

- __Directorios enlazados__ *(bind):* la información se guarda fuera de Docker en el host local. Esta opción es mejor para datos no generados por los contenedores.
- __Volúmenes__: la información se guarda usando Docker. Mejor para datos generados por los propios contenedores.

---

## Directorios enlazados (bind)

Características:

- Permiten  __persistencia__  de la información del contenedor.
- Se montan en un  __path específico de la máquina local__  (fuera de Docker).
- Permiten  __borrar los contenedores sin perder la información__ .

---

### Ejemplo con directorios enlazados (bind) (1)

Crear directorio local `p02` para compartir:

`mkdir /Users/Usuario/p02`

Ejecutar el contenedor:

```bash
docker run -d -P --name=apache-bind-1 \
  --mount type=bind,source=`pwd`/p02,target=/app bitnami/apache
```

Comprobar los puertos a los que se ha asignado el 8080 (http) y 8443 (https) del Servidor Apache configurado en el contenedor:

```bash
$ docker ps -a
CONTAINER ID   IMAGE           COMMAND                 CREATED        STATUS         PORTS                                              NAMES
...
bcaae8157e45   bitnami/apache  "/opt/bitnami/script…"  3 minutes ago  Up 3 minutes   0.0.0.0:55004->8080/tcp, 0.0.0.0:55003->8443/tcp   apache-bind-1
...
```

---

Editar el fichero con _nano_ (u otro editor)

`nano /Users/Usuario/p02/index.html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="utf-8">
  <title>Apache en Docker</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>
  <h1>Hola Usuario! :)</h1>
</body>
</html>
```


---

Abrir en el host las URL http://localhost:55004/ y https://localhost:55003/

![width:550](img/iiss-docker-021.png) ![width:550](img/iiss-docker-022.png)

Si eliminamos el contenedor, no perdemos el contenido de la web:


```bash
docker stop apache-bind-1
docker rm apache-bind-1
docker run -d -P --name=apache-bind-2 \
  --mount type=bind,source=`pwd`/p02,target=/app bitnami/apache
```

---

## Volúmenes

### Características

- Permiten persistencia de información del contenedor
- Se montan en un _path_ específico del contenedor
- El acceso a la información sólo puede realizarse a través de Docker


### Ejemplo con volúmenes (1)

Creando un volumen para la web

```
docker run -d -P --name=apache-volume-1 \
   --mount type=volume,source=vol-apache,target=/app bitnami/apache
```

Comprobar el puerto asignado con `docker ps -a `

---

Acceso al volumen

`docker volume ls`

`docker exec -it apache-volume-1 /bin/bash`

Comprobar que `/app` está vacía y salir con `exit`

Actualizamos el contenido del volumen

`nano index.html`

`docker cp index.html apache-volume-1:app/`

Acceso de nuevo al volumen y comprobar que está el archivo `index.html`

`docker exec -it apache-volume-1 /bin/bash`


---

Acceso al contenido de la web

![width:550](img/iiss-docker-023.png)

Compartir volumen con otro contenedor

```
docker run -d -P --name=apache-volume-2 \
   --mount type=volume,source=vol-apache,target=/app bitnami/apache
```

---

Borrando volumen

```
docker stop apache-volume-2
docker rm apache-volume-2
```

`docker volume remove vol-apache`

```
Error response from daemon: remove vol-apache: volume is in use - [4a4794f86...3c8]
```

Reintentar borrado:

```
docker stop apache-volume-1
docker rm apache-volume-1
docker rm apache-volume-2
docker volume rm vol-apache
```

Comprobar que se ha borrado con `docker volume ls`

---

##  Configuraciones de red

- Permiten comunicación entre todos los contenedores pertenecientes a una red a través del nombre del contenedor
- Permiten aislamiento con respecto a otros contenedores
- Un contenedor puede pertenecer a varias redes

---

###  Tipos de redes

Ver las redes existentes: `docker network ls`

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
21c353b4b7a5   bridge    bridge    local
7a7abec15748   host      host      local
40157ecf9fcf   none      null      local
```

- **host** representa la red del propio equipo y haría referencia a `eth0`
- **bridge** representa la red `docker0` y a ella se conectan todos los contenedores por defecto 
- **none** significa que el contenedor no se incluye en ninguna red. Si verificamos esto con el comando `ifconfig` dentro del contenedor, veríamos que sólo tiene la interfaz de loopback `lo`

Saber qué contenedores usan un tipo de red: `docker network inspect bridge`

---

###  Ejemplo de configuración de red (1)

WordPress empaquetado en un contenedor:

https://github.com/bitnami/bitnami-docker-wordpress

Configuración manual de la red:

https://github.com/bitnami/bitnami-docker-wordpress#using-the-docker-command-line

Crear una nueva red:

`docker network create wordpress-network`

Crear primer volumen para persistencia de MariaDB:

`docker volume create --name mariadb_data`

Crear segundo volumen para persistencia de WordPress:

`docker volume create --name wordpress_data`

---

Crear primer contenedor que usa la red (base de datos `mariadb`):

```bash
docker run -d --name mariadb --env ALLOW_EMPTY_PASSWORD=yes \
  --env MARIADB_USER=bn_wordpress \
  --env MARIADB_DATABASE=bitnami_wordpress \
  --network wordpress-network \
  --volume mariadb_data:/bitnami/mariadb \
  bitnami/mariadb:latest
```

Crear segundo contenedor que usa la red (servidor `wordpress`):

```bash
docker run -d --name wordpress -p 8080:8080 -p 8443:8443 \
  --env ALLOW_EMPTY_PASSWORD=yes \
  --env WORDPRESS_DATABASE_USER=bn_wordpress \
  --env WORDPRESS_DATABASE_NAME=bitnami_wordpress \
  --network wordpress-network \
  --volume wordpress_data:/bitnami/wordpress \
  bitnami/wordpress:latest
```

---

![width:600 center](img/iiss-docker-024.png)

---

## Tarea: Práctica 2

Escribir un documento markdown con los comandos realizados para resolver los siguientes ejercicios propuestos. Incluir también la respuesta a las preguntas realizadas y capturas donde se requiera mostrar en el navegador el correcto funcionamiento de los contenedores.

---

### Parte 1

1. Crear volumen compartido `volumenDocker`

Creamos el volumen que puede ser usado por uno  varios contenedores usando el siguiente comando:

```
docker volume create volumenDocker
```

2. Crear un contenedor de Nginx que use el volumen `volumenDocker`.
```
docker run -d -p 80:80 --name nginx-container -v volumenDocker: /usr/share/nginx/html nginx
```

3. Modifique el contenido del fichero `index.html` incluyendo un saludo personal en lugar del texto por defecto.

Desde el dockerfile programo el fichero index.html para modificar el saludo. Lo construimos y ejecutamos y comprobamos que funciona.
```
//Contenido del Dockerfile
FROM nginx:latest

RUN echo '<!DOCTYPE html> <html> <head> <meta charset="UTF-8"> <title>Miriam Armario</title> <style> body {background-color: black;} </style> </head> <body> <h1 style="color: white; text-align: center;">Miriam Armario IISS 2023</h1> <img src="https://cdn.pixabay.com/photo/2015/02/24/15/41/wolf-647528_640.jpg" alt="Impulsada por el viento del atardecer"> </body> </html>' > /usr/share/nginx/html/index.html

EXPOSE 80

//Comandos de la terminal
docker build -t nginx1 .
docker run -d 80:80 nginx1

```


4. Cree un segundo contenedor que también use el volumen `volumenDocker`.
```
docker run -d -p 81:80 --name nginx-container2 -v volumenDocker: /usr/share/nginx/html nginx
```
5. Compruebe que puede acceder a `localhost:80` (primer contenedor) y `localhost:81` (segundo contenedor) y ver el contenido de `index.html`.

Las imágenes de comprobación están subidas aparte, con nombre `Docker2-80` (imagen que comprueba el acceso a `localhost:80`) y `Docker2-81` (imagen que comprueba el acceso a `localhost:81`).

---

### Parte 2

1. Crear una nueva red `redDocker`.
2. Crear un contenedor de Ubuntu `Ubuntu1`.
3. Crear un contenedor de Ubuntu `Ubuntu2`.
4. Conectar `Ubuntu1` a la red `redDocker`.
5. Intentar hacer ping a `Ubuntu1` desde `Ubuntu2`. ¿Funciona? ¿Por qué?.
6. Conectar `Ubuntu2` a la red `redDocker`.
7. Intentar de nuevo hacer ping a `Ubuntu1` desde `Ubuntu2`. ¿Funciona? ¿Por qué?.

---

## Task 2

Write a markdown document with the commands to be run to solve the following tasks. Answer also the proposed questions and include screenshots where it is required to check the correct functioning of containers.

---

###  Part 1

1. Create shared volume `DockerVolume`
2. Create a Nginx container that uses the `DockerVolume` volume.
3. Update the `index.html` file contents by including a personal greeting instead of the default message.
4. Create a second container that uses the `DockerVolume` volume.
5. Check that you can access `localhost:80` (first container) and `localhost:81` (second container) and see the content of `index.html`.

---

### Part 2

1. Create a new `DockerNetwork` network.
2. Create an Ubuntu container `Ubuntu1`.
3. Create an Ubuntu container `Ubuntu2`.
4. Connect `Ubuntu1` to the `DockerNetwork` network.
5. Try to _ping_ from `Ubuntu1` to `Ubuntu2`. Does it work? Why?
6. Connect `Ubuntu2` to the `DockerNetwork` network.
7. Try to _ping_ from `Ubuntu1` to `Ubuntu2` again. Does it work? Why?
