
## ¿Qué es Docker?

* El mundo antes de Docker:

  * sin virtualización
  * virtualización basada en hipervisores
  * virtualización basada en contenedores

### Arquitectura

* Docker usa una arquitectura cliente-servidor

![](architecture.svg "architecture.svg"){width="" height="400"}

### Elementos principales de Docker {#main\_docker\_elements}

* **daemon**: proceso que se ejecuta en la máquina anfitriona (servidor)
* **cliente**: interfaz principal de Docker
* **imagen**: plantilla de solo lectura (componente de construcción)
* **registro**: repositorios públicos o privados de imágenes (distribución, componente de envío)
* **contenedor**: creado a partir de una imagen, contiene todo lo necesario para ejecutar una aplicación (componente de ejecución)


### Beneficios de Docker {#benefits\_of\_docker}

* separación de roles y responsabilidades

  * los desarrolladores se enfocan en construir aplicaciones
  * los administradores del sistema se enfocan en el despliegue
* portabilidad: se construye en un entorno, se distribuye y ejecuta en muchos otros
* desarrollo, prueba y despliegue más rápidos
* escalabilidad: fácil creación de nuevos contenedores o migración a hosts más potentes
* mejor utilización de recursos: más aplicaciones en un solo host


### La tecnología subyacente {#the\_underlying\_technology}

* **namespaces (espacios de nombres)**

  * **pid**: aislamiento de procesos (ID de proceso)
  * **net**: manejo de interfaces de red
  * **mnt**: manejo de puntos de montaje
  * **ipc**: manejo del acceso a recursos de comunicación entre procesos
  * **uts**: aislamiento de identificadores del kernel y la versión (Unix Timesharing System)
* **control groups (cgroups)**

  * comparten recursos de hardware disponibles
  * establecen límites y restricciones
* **sistema de archivos en capas (UnionFS)**

  * opera creando capas
  * muchas capas se fusionan y son visibles como un sistema de archivos consistente
  * ejemplos: **AUFS**, btrfs, vfs, DeviceMapper
* **formato de contenedor**

  * dos formatos soportados: **libcontainer**, LXC


## Comenzando {#getting\_started}

### Instalación del motor y cliente de Docker {#installation\_of\_docker\_engine\_and\_client}

```bash
# forma más fácil de instalar Docker
$ wget -qO- https://get.docker.com/ | sh
# para usar docker como usuario ubuntu sin sudo (opcional)
$ sudo usermod -aG docker ubuntu
# para verificar la instalación
$ docker --version
```

* [Pasos de instalación manual, más detalles e instrucciones para otros sistemas operativos](https://docs.docker.com/engine/installation/ubuntulinux/)


### Ejemplo "Hola Mundo" {#hello\_world\_example}

```bash
$ docker run hello-world
$ docker images
$ docker ps -a
```


### Terminal bash dockerizada {#dockerized\_bash\_terminal}

```bash
$ docker run -it ubuntu
$ docker run -it ubuntu:latest
$ docker run -it ubuntu:14.04 bash
$ docker run -it ubuntu ps -aux
```

* docker run **-t**: asigna un pseudo-tty
* docker run **-i** (--interactive): mantiene STDIN abierto incluso si no está conectado
* usar **CTRL + p + q** para desconectarse del contenedor sin detenerlo
* usar el comando **attach** para reconectarse

```bash
$ docker attach nombre_del_contenedor
```

* la importancia del PID 1
* PID en el contenedor y en el host de Docker:

```bash
$ ps -fe | grep $(pidof docker)
```

* instalación de paquetes: mc, vim


### Investigación de contenedores e imágenes {#investigating\_containers\_and\_images}

* docker **inspect** muestra información de bajo nivel sobre un contenedor o imagen

```bash
$ docker inspect webapp
$ docker inspect --format='{{.NetworkSettings.IPAddress}}' webapp
$ docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' webapp
```

* docker **diff** muestra cambios en el sistema de archivos de un contenedor

```bash
$ docker diff webapp
$ docker diff webapp | grep ^A
```

* docker **history** muestra el historial de una imagen (capas)

```bash
$ docker history ubuntu
$ docker history --no-trunc ubuntu
```

* docker **logs** obtiene los registros de un contenedor

```bash
$ docker logs webapp
$ docker logs --tail 15 webapp
$ docker logs -f webapp
$ docker logs -t webapp
$ docker logs --since 1h webapp
```

* docker **top** muestra los procesos en ejecución dentro de un contenedor

```bash
$ docker top webapp
$ docker exec webapp ps aux
```


### Limpieza y mantenimiento {#cleaning\_up\_and\_keeping\_clean}

```bash
$ docker images
$ docker ps -a
```

* docker run **--rm** elimina automáticamente el contenedor al salir

```bash
$ docker run --rm -it ubuntu bash
```

* docker run **--name** asigna un nombre personalizado al contenedor

```bash
$ docker run --rm -it --name=test ubuntu bash
```

* el nombre del contenedor debe ser único y se puede cambiar

```bash
$ docker rename test nombre_significativo
```

* docker **rm** elimina un contenedor
* docker **rmi** elimina una imagen

```bash
$ docker rm nombre_significativo
```

```bash
# elimina todos los contenedores que no están en ejecución
$ docker rm $(docker ps -a | grep 'Exited' | awk '{print $1}')
# elimina todas las imágenes sin etiqueta
$ docker rmi $(docker images | grep '^<none>' | awk '{print $3}')
```

*Ejercicios*

## Almacenamiento y persistencia de datos {#storage\_and\_data\_persistence}

### Dentro del contenedor {#within\_the\_container}

* los datos son visibles solo dentro del contenedor
* no se persisten fuera del contenedor
* se pierden si se elimina el contenedor


### Directamente en el host de Docker {#directly\_on\_docker\_host}

```bash
$ docker run -v /host/dir:/container/dir:rw ...
$ docker run -v /home/ubuntu/docker/training_httpd1/html:/var/www/html/ -d --name www --net host training:httpd1
$ docker run -v $PWD/html:/var/www/html/ -d --name www --net host training:httpd1
$ docker run -v $PWD/html:/var/www/html/:ro -d --name www --net host training:httpd1
```

* los datos son visibles dentro del contenedor, en el host y pueden compartirse entre contenedores
* se persisten incluso si se elimina el contenedor
* ofrece un rendimiento casi como en metal desnudo
* el directorio del host puede ser una unidad NFS, dispositivo formateado o cualquier recurso montable


### Fuera de UnionFS de Docker {#outside\_dockers\_unionfs}

```bash
$ docker run -itd -v /data --name data1 ubuntu
$ docker inspect data1
$ docker exec data1 touch /data/file1
$ docker exec data1 ls -l /data/
$ docker run -itd --volumes-from data1 --name data2 ubuntu
$ docker inspect data2
$ docker rm -fv data1 data2
```

```bash
$ docker volume create --name kb_volume
$ docker run --rm -it -v kb_volume:/data ubuntu touch /data/kb
$ docker run --rm -it -v kb_volume:/data ubuntu ls -l /data
$ docker volume rm kb_volume
$ docker volume ls
```

* los datos son visibles y compartibles entre contenedores
* persisten incluso si el contenedor se elimina

  * usa `docker run --rm -v` para eliminar contenedor y volúmenes (a menos que otro contenedor los use)
* ofrece rendimiento casi de nivel bare-metal
* resuelve el problema de permisos (IDs diferentes entre host y contenedor)

**Crear grupos y usuarios con ID personalizado**

```bash
$ groupadd -r -g 27017  mongodb
$ useradd -r -u 27017 -g mongodb mongodb
```

#### Respaldar y restaurar datos de volúmenes {#backup\_and\_restore\_data\_from\_volumes}

```bash
# respaldo
$ docker run -itd -v /data --name data1 ubuntu
$ docker exec data1 touch /data/file1
$ docker exec data1 chown www-data:www-data /data/file1
$ docker run --rm --volumes-from data1
```


#### Respaldar y restaurar datos de volúmenes {#backup\_and\_restore\_data\_from\_volumes}

```bash
# respaldar datos desde un contenedor
$ docker run -itd -v /data --name data1 ubuntu
$ docker exec data1 touch /data/file1
$ docker exec data1 chown www-data:www-data /data/file1
$ docker run --rm --volumes-from data1 ubuntu ls -l /data
$ docker run --rm --volumes-from data1 -v $PWD:/backup ubuntu tar -cvpf /backup/backup.tar /data
$ docker rm -fv data1
# restaurar datos en un contenedor nuevo
$ docker run -itd -v /data --name data1 ubuntu
$ docker run --rm --volumes-from data1 -v $PWD:/backup ubuntu tar -xvpf /backup/backup.tar
$ docker run --rm --volumes-from data1 -v $PWD:/backup ubuntu bash -c "cd /data && tar -xvf /backup/backup.tar --strip 1"
$ docker run --rm --volumes-from data1 ubuntu ls -l /data
```


### Fuera del host de Docker {#outside\_docker\_host}


## Redes (Networking)

```bash
$ docker network --help
$ docker network ls
```


### Red del host de Docker {#docker\_host\_network}

* la **red host** agrega un contenedor directamente al stack de red del host
* la configuración de red dentro del contenedor es idéntica a la del host

```bash
$ docker run --name db1 -d --net host training:mongod
$ docker run --name www -d --net host training:httpd1
$ docker inspect www
$ docker network inspect host
```


### Sin interfaz de red {#without\_network\_interface}

* la red **none** aísla completamente el contenedor en su propio stack de red
* usa el comando *docker exec* para acceder al contenedor

```bash
$ docker run --name networkless --net none -it --rm ubuntu bash
```


### Red por defecto (bridge) {#default\_network\_bridge}

```bash
$ docker run --name db2 -d --volumes-from db1 training:mongod
$ docker run --name www -d training:httpd1
$ docker run --name www -d -P training:httpd1
$ docker run --name www -d -p 80:80 training:httpd1
$ docker run --name www -d -p 127.0.0.1:88:80 training:httpd1
$ docker run --name www -d -p 80:80 --link db2 training:httpd1
$ docker run --name www -d -p 80:80 --link db2:db training:httpd1
$ docker inspect www
$ docker network inspect host
```

\* esta es la red por defecto para todos los contenedores

* los contenedores pueden comunicarse entre sí usando direcciones IP
* Docker **no** admite descubrimiento automático de servicios en la red bridge por defecto
* para comunicarte por nombre en esta red, debes usar la opción heredada **docker run --link**

### Redes personalizadas del usuario (bridge) {#custom\_user\_networks\_bridge}

```bash
$ docker network create --driver bridge --subnet 10.1.2.0/24 net1
$ docker network create --subnet 10.1.2.0/24 --gateway=10.1.2.1 net1
$ docker network ls
$ docker network inspect net1
$ docker network disconnect bridge www
$ docker network connect net1 www
$ docker run -itd --net net1 --name mongo --net-alias db training:mongod
$ docker run -itd --net net1 --name apache --net-alias www --hostname httpd -p 80:80 --env=db_host=db training:httpd1
```

## Docker y Kubernates recursos
[Docker website](https://www.docker.com)

[Kubernetes website](https://kubernetes.io)

[Docker y Kubernates Capacitación | NobleProg](https://www.nobleprog.mx/cursos-docker)

[Video Capacitación | DaDesktop](https://www.dadesktop.com/videos-catalog/list/4e7b7e9c-2ffa-4cfa-b6a0-ccd2f3a51eaa)

