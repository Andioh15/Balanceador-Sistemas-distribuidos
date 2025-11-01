PASO 1: Crear la carpeta del proyecto

Abre una terminal y ejecuta:

mkdir mongo-load-balance
cd mongo-load-balance


Esto crea y entra a tu carpeta de práctica.

PASO 2: Crear el archivo docker-compose.yml

Dentro de esa carpeta, crea el archivo:

notepad docker-compose.yml

Pega esto dentro:

version: '3.8'

services:
  mongo1:
    image: mongo:latest
    container_name: mongo1
    ports:
      - "27018:27017"
    networks:
      - mongo_net

  mongo2:
    image: mongo:latest
    container_name: mongo2
    ports:
      - "27019:27017"
    networks:
      - mongo_net

  mongo3:
    image: mongo:latest
    container_name: mongo3
    ports:
      - "27020:27017"
    networks:
      - mongo_net

  nginx:
    image: nginx:latest
    container_name: nginx_lb
    ports:
      - "27017:27017"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - mongo1
      - mongo2
      - mongo3
    networks:
      - mongo_net

networks:
  mongo_net:
    driver: bridge


Guarda y cierra el archivo.

PASO 3: Crear la carpeta y archivo de configuración de Nginx

Crea la carpeta nginx:

mkdir nginx

Y dentro, crea el archivo nginx.conf:

notepad nginx/nginx.conf


Pega este contenido:

worker_processes 1;
events { worker_connections 1024; }

stream {
    upstream mongo_cluster {
        server mongo1:27017;
        server mongo2:27017;
        server mongo3:27017;
    }
    server {
        listen 27017; # Nginx escucha en 27017
        proxy_pass mongo_cluster; # Redirige el tráfico TCP a los servidores
        proxy_connect_timeout 10s;
        proxy_timeout 90s; # Usamos proxy_timeout en lugar de proxy_read_timeout para el bloque stream
    }
}

PASO 4: Levantar los contenedores

Asegúrate de estar en la carpeta donde está el docker-compose.yml
Luego ejecuta:

docker-compose up -d

Esto hará lo siguiente:

Descargará las imágenes de Mongo y Nginx.

Creará 4 contenedores: mongo1, mongo2, mongo3, nginx_lb.

PASO 5: Verificar que estén corriendo

Ejecuta:

docker ps

Deberías ver algo así:

CONTAINER ID   NAMES       IMAGE         PORTS
abc123         nginx_lb    nginx:latest  0.0.0.0:27017->27017/tcp
def456         mongo1      mongo:latest  0.0.0.0:27018->27017/tcp
ghi789         mongo2      mongo:latest  0.0.0.0:27019->27017/tcp
jkl012         mongo3      mongo:latest  0.0.0.0:27020->27017/tcp

Para verificar el balanceo:
Conectarse directamente a mongo1 (a través de su puerto mapeado):

Revisa tu docker ps inicial para ver el puerto de host de mongo1. Era el puerto 27018.
Bash

mongosh --port 27018
use server_id_mongo1
db.metadata.insertOne({server: "mongo1"})
exit

Conectarse directamente a mongo2 (puerto 27019):
Bash

mongosh --port 27019
use server_id_mongo2
db.metadata.insertOne({server: "mongo2"})
exit

Conectarse directamente a mongo3 (puerto 27020):
Bash

mongosh --port 27020
use server_id_mongo3
db.metadata.insertOne({server: "mongo3"})
exit

Paso 2: Ejecutar la Prueba a Través de Nginx
Ahora, conéctate repetidamente al balanceador de carga Nginx (localhost:27017) y lista las bases de datos para ver cuál aparece.

En la terminal de tu pc, no de docker (debes tener instalado mongo).
# Repite este comando varias veces (ej. 5 o 6 veces)
mongosh --port 27017 --eval "db.getMongo().getDBNames()"
