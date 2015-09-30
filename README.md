# Consul

Se trata de una solución distribuida que ofrece funcionalidades y utilidades que encajan muy bien en un entorno Dockerizado en lo que se refiere a:

* Descubrimiento de servicios con capacidades para personalizar el health checking de los servicios registrados y excluir aquellas instancias que no estén funcionando adecuadamente.
* Almacen K/V para diversos usos:
    * Gestionar la configuración variable de aplicaciones de forma centralizada.
    * Gestión interna de los servicios registrados en runtime.

[Ofrece una lista amplia de utilidades] (https://www.consul.io/downloads_tools.html) creadas por el proveedor - Hashicorp- y por la comunidad (tanto en github como Docker hub)

## Instanciacion

El plateamiento de despliegue para por la existencia de una instanciación de Consul por entorno (dev/cert/pre y pro)
Para ello, se podría aprovechar las capacidades MultiDatacenter de Consul o bien crear una instancia específica por entorno.

Para disponer de un cluster de Consul, basta con levantar dos tipos de nodos:

* Nodos SERVER son los que son capaces de persistir la información de forma distribuida
* Nodos CLIENTE o AGENTES. Se encargan de hablar directamente con los nodos SERVER además de exponer servicios (UI, DNS) evitando carga directamente sobre los nodos SERVER.

Los siguientes comandos permiten simular un cluster de consul sobre una maquina virtual:

```
docker run -d --name node1 -v /var/tmp/data-node1:/data -h node1 progrium/consul -server -bootstrap-expect 3 

export JOIN_IP="$(docker inspect -f '{{.NetworkSettings.IPAddress}}' node1)"

docker run -d --name node2 -v /var/tmp/data-node2:/data -h node2 progrium/consul -server -join $JOIN_IP

docker run -d --name node3 -v /var/tmp/data-node3:/data -h node3 progrium/consul -server -join $JOIN_IP

docker run -d -v /var/tmp/data-node4:/data -p 8400:8400 -p 8500:8500 -p 53:53/udp --name node4 -h node4 progrium/consul -join $JOIN_IP

```
## Accediendo a consul
* [UI - Consul] (http://oclubunc022.isbcloud.isban.corp:8500/ui/#/dc1)
* [API - Consul] (http://oclubunc022.isbcloud.isban.corp:8500/v1/catalog/nodes) 

        ```
        curl oclubunc022.isbcloud.isban.corp:8500/v1/catalog/nodes
        curl oclubunc022.isbcloud.isban.corp:8500/v1/catalog/services
        curl oclubunc022.isbcloud.isban.corp:8500/v1/health/checks/service
        ```

## Componentes a instanciar en un entorno de ejecución de aplicaciones dockerizadas
 * [Registrator] (https://github.com/gliderlabs/registrator)
    * FUNCION: Posibilitar el registro automático en el directorio de servicios de las aplicaciones/servicios/microservicios dockerizados.
    * CMD:
    ```
      docker run -d --name=registrator --net=host --volume=/var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://oclubunc022.isbcloud.isban.corp:8500
    ```
 
 *  [k-v plugin] (https://github.com/bryanlarsen/docker-plugin-kv-consul)
    * FUNCION: Posibilitar el volcado en runtime de info de servicio
    * CMD:
     ```
     docker run -d -e "KV_CONSUL_PREFIX=services/" -e "KV_CONSUL_URL=oclubunc022.isbcloud.isban.corp:8500/v1/kv/$KV_CONSUL_PREFIX" --hostname="$(hostname)" -v /var/run/docker.sock:/var/run/docker.sock bryanlarsen/kv-consul
     ```
## Demo registro simple y automático en el directorio de servicios

* A modo de ejemplo, al leventar el contenedor de base de datos MYSQL se registrará de forma automática en Consul la existencia de dicho servicio.

``` docker run -d  --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=consul-test mysql:5.5```

Esta modalidad de efectuar el registro es:
0. Automática. El SW/Mware que se ejecuta en el contenedor no conoce la existencia de un directorio de servicios.
0. No intrusiva. No es necesario ni embeber librerías ni desarrollar codigo a medida para efectuar el registro.

**NOTA**: Es posible jugar con variables de entorno específicas para customizar la información a registrar:
* SERVICE_NAME
* SERVICE_ID
* SERVICE_TAGS

Ejemplo:
``` docker run -d -p 3307:3306 -e "SERVICE_NAME=miSQL" -e "SERVICE_ID=BD.1" -e "SERVICE_TAGS=5.5,DEV" -e MYSQL_ROOT_PASSWORD=consul-test mysql:5.5```

¿que ocurre con otro tipo de servicios que no corren sobre dicha infraestructura dockerizada o que corren en otro entorno?
* Es posible registrar estos servicios para que las aplicaciones puedan atarse a dichos servicios de la misma forma que si fueran servicios dockerizados.
* Esto ofrece además capacidades las capacidades de monitorización que, en caso de caída del servicio, se eliminaría del registro de forma automática 

El caso más tipico puede ser el registro de un servicio "search" que ofrece Google.
```curl -X PUT -d '{"Datacenter": "dc1", "Node": "Google External","Address": "www.google.com", "Service": {"Service": "search", "Port": 80}}' http://oclubunc022.isbcloud.isban.corp:8500/v1/catalog/register```

Pero también podría ser una base de datos Oracle que corre sobre infraestructura virtualizada - en este caso, la que el proyecto CRIS Digital utiliza en el entorno DEV
```curl -X PUT -d '{"Datacenter": "dc1", "Node": "Oracle External","Address": "odisnbp3.isban.dev.corp:50010:odisnbp3", "Service": {"Service": "Oracle-DB-odisnbp3", "Port": 50010}}' http://oclubunc022.isbcloud.isban.corp:8500/v1/catalog/register```

## Demo conectividad via DNS a un servicio existente en el directorio de Consul
Tomando como ejemplo una base de datos mysql del ejemplo anterior, se va a mostrar como, desde otro entorno, es posible conectar mediante un cliente mysql a través del nombre tal y como se ha registrado dicho servicio en consul.

```docker run --dns=180.197.78.33 --rm -it shopigniter/mysql-client -h mysql.service.dc1.consul -uroot -pconsul-test```

El contenedor se ejecuta de forma interactiva de modo que se debe visualizar algo como lo siguiente:
```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 1
Server version: 5.5.45 MySQL Community Server (GPL)
Copyright (c) 2000, 2014, Oracle and/or its affiliates. All rights reserved.
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql
```

La parte referente al DNS es opcional siempre y cuando en la configuración del docker daemon del host donde se arranca el cliente mysql tenga algo como esto:
```DOCKER_OPTS="--dns 180.197.190.20 --dns 180.197.78.33 --dns-search service.dc1.consul ..."```

## Mapeo de servicios en runtime y sus dependencias sobre K/V
¿Se necesita más información de los servicios que se encuentran catalogados en el directorio?
Podría ser interesante disponer de información completa de los servicios que se encuentran en ejecución sobre contenedores Docker en varios niveles:
0. A nivel de infraestructura (id y nombre del contenedor docker, puertos internos, host desde el que se ha levantado, etc)
0. A nivel funcional (listado de dependencias de otros servicios e incluso los servicios que expone - API REST?)

Parece interesante y lo es aun más si toda esta información se puede registrar de forma automática. 
Lo que tiene un poco mas de complejidad es:
0. Ofrecer una estructura en formato YML que permita albergar este tipo de información de forma global para las aplicaciones. (YML Extendido)
0. Incluir la información de dependencias y servicios expuestos en el YML
0. Diseñar una estructura jerarquica global que permita mapearla información en runtime de los servicios sobre la estructura k/v de Consul

A modo de ejemplo, este sería el comando final a ejecutar. 
```
docker run -d -e "SERVICE_NAME=demo"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/SERVICE_ID=#<SERVICE_ID>#"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/IMAGE_NAME=#<IMAGE_NAME>#"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/IMAGE_ID=#<IMAGE_ID>#"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/CONTAINER_ID=#<CONTAINER_ID>#"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/IP_ADDRESS=#<IP_ADDRESS>#"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/HOSTNAME=#<HOSTNAME>#"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/SERVICE_PORT_NAME=<SERVICE_<port>_NAME>"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/SERVICE_PORT_ID=<SERVICE_<port>_ID>"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/HOST_PORT_PORT=<HOST_<port>_PORT>"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/dependencies/dep1=mysql.service.dc1.consul"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/dependencies/dep2=bdp.service.dc1.consul"\
 -e "KV_SET:services/#<SERVICE_NAME>#/#<CONTAINER_NAME>#/exposes/exp1=products"\
 -p 80 tutum/hello-world
```

**NOTA**: Ver qué ocurre al instanciar mas contenedores con el mismo nombre de servicio.

## Gestión de la configuración k-v de  las aplicaciones
A título informativo, merece la pena una lectura el siguiente [enlace] (http://12factor.net/) y concretamente [éste] (http://12factor.net/config) en lo que al contexto del apartado se refiere 
Por tanto, cuando hablamos de configuración nos referimos a elementos cuyos valores son distintos en distintos entornos.
Desde nuestro punto de vista, la explotación de variables de entorno se acentúa mucho más si se desarrolla haciendo uso de imágenes Docker en puesto local.

**¿Qué buscamos?**
0. Una forma centralizada para definir la configuración de la aplicación (vía UI, API,etc)
0. Una forma de convertir de forma automática la configuración dinámica de la aplicación en variables de entorno.

Para llevar a cabo el primer punto, es posible ver tanto el UI como ver las operaciones que se exponen en el API K-V de Consul para hacerse una idea.
Para el segundo, basta con analizar una utilidad denominada [envconsul daemon] (https://github.com/hashicorp/envconsul) para hacerse una idea de su funcionamiento.

Ejemplo: Volcado de configuración a env del entorno donde se ejecuta:

```envconsul -consul oclubunc022.isbcloud.isban.corp:8500 -sanitize -upcase  env & ```

¿Y si envconsul daemon corre DENTRO del contenedor donde la aplicación está en ejecución?
Bastaría con modificar/extender la imagen donde va a correr la aplicación para incluir en script de arranque el siguiente comando:

  envconsul -consul demo.consul.io -prefix global /bin/sh -c "env; echo "-----"; sleep 1000"
  
Lo importante aquí es la parte prefix que es la que va a implementar el watch automatico que permite traspasar los cambios del K-V Store a variables de entorno.

Ver https://hub.docker.com/r/mhamrah/envconsul/

## Proximos pasos (a considerar si aplica)

Cobertura de otros aspectos sobre el producto:

0. Encriptación de datos tanto en la comunicación entre nodos agente y servidor como en la persitencia de datos.
0. Capacidades de control de acceso sobre K/V y servicios.
0. Multiples chequeos sobre servicios.
0. Despliegue de Consul para entornos productivos.
0. Alertas.
0. Consul-template + haproxy como solución para balancear peticiones entre distintas instancias de un servicio.
