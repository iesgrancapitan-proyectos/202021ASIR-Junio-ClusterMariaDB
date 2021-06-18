## 1. Definición de proyecto

Este proyecto integrado tiene como objetivo el montaje de un clúster de MariaDB con el complemento galera, el cual está montado en contenedores Docker gestionados por Kubernetes. Todas las conexiones entre los nodos serán seguras gracias a la implementación de mecanismo de cifrado, así como se estudiarán las diferentes formas de cifrar los datos contenidos en la base de datos, y se decidirá si se implementa o no en función de los recursos necesarios y el valor de los datos almacenados.

Kubernetes nos permite añadir un mecanismo de alta disponibilidad, ya que este se encarga de forma automática de levantar y mantener tantos nodos como sea necesario, permitiendo una rápida escalabilidad del sistema , si en un momento dado se necesita un mayor número de nodos del clúster de MariaDB, podemos implementarlos de forma sencilla, además kubernetes se encargara de forma automática de reponer un nodo en caso de que este deje de funcionar.

Para probar el sistema se levantará en un contenedor un WordPress en apache, el cual se comunicará con el clúster de MariaDB y todas sus conexiones se asegurarán con mecanismos de cifrado.

## 2. Despliegue
El despliegue de este proyecto se lleva a cabo en 5 etapas:

### 2.1. Instalación de Docker y Kubernetes
El primer paso consiste en instalar Docker y Kubernetes en el sistema operativo anfitrión, en este caso Ubuntu Server 20.04 LTS. Docker se instala desde los repositorios de Ubuntu a través del comando apt, para instalar Kubernetes tenemos que añadir a nuestros repositorios el repositorio correspondiente de Google. Una vez instalado se lanza el nodo de Kubernetes. Todo esto se detalla en el entregable e-kmdb-01 de la wiki.

### 2.2. Creación de los contenedores del clúster de MariaDB-Galera
Para crear los contenedores utilizaremos el comando helm el cual se instala a través de los repositorios de snap. Con este comando desplegaremos los contnedores correspondientes de los servicios que nos permita el proyecto de bitnami para kubernetes. Para MariaDB-Galera utilizamos el repositorio bitnami/mariadb-galera. Desde este repostorio montamos un clúster de 3 nodos con una configuración por defecto, pero el despliegue con helm nos permite modificar parámetros para el despliegue del contenedor, para personalizar estos contenderes como se detalla en el entregable e-kmdb-03 de la wiki.

### 2.3. Creacción del conetendor WordPress-Apache
Nuevamente aprovecharemos los repositorios de bitnami para desplegar el contenedor con el uso del comando helm, para este caso utilizaremos el repositorio bitnami/wordpress incluyendo en los parámetros de despliegue el nombre del servicio correspondiente a la base de datos desplegada en el punto anterior. Este proceso se detalla en el entregable e-kmdb-04 de la wiki.

### 2.4. Cifrado de la conexión de los nodos de MariaDB-Galera
El comando helm no solo nos permite desplegar contenedores, sino también actualizar la configuración de un servicio de contenedores ya desplegado, para ello añadimos el parametro update al comando helm y el nombre del servicio que queremos actualizar junto con las variables que queremos actualizar. En este punto, actualizamos los contenedores que conforman el clúster de MariaDB-Galera incluyendo certificados para el cifrado de la conexión de la misma. Estos certificados se pasan al contenedor a través del uso de secret como se detalla en el entregable e-kmdb-05 de la wiki.

### 2.5. Cifrado del contenido de la base de datos
Por último se cifrafrá el contenido de la base de datos utilizando el plugin file_key_managment de MariaDB, para ello tenemos que volver a actualizar la configuración de los contenedores desplegados incluyendo un fichero de configuración adicional a MariaDB, para personalizar por completo el despliegue de los contenedores. Este fichero se incluye en un archivo con extensión .yaml donde añadimos las líneas correspondiente al plugin en el apartado MariaDB. Al realizar el update pasamos como parámetro este fichero .yaml al ejecutar nuevamente el helm update. Podemos verlo en detalle en el entregable e-kmdb-06 de la wiki

## 3. Uso
Una vez el sistema esté completamente desplegado, accedemos desde el navegador a la dirección  http://192.168.12.146:32049/wp-admin para gestionar el contenido de nuestro navegador. Para gestionar la base de datos tenemos varios métodos de acceso:
### 1. Acceso mediante un contenedor temporal 
Levantamos un contenedor temporal en la en kubernetes que lanza un cliente de MaríaDB que se conecta directamente con el servidor ejecutando: 

 $ kubectl run my-release-mariadb-galera-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/mariadb-galera:10.5.9- 
  debian-10-r52 --command -- mysql -h my-release-mariadb-galera -P 3306 -uroot -p$(kubectl get secret --namespace default my-release-mariadb-galera -o 
  jsonpath="{.data.mariadb-root-password}" | base64 --decode) my_database
### 2. Acceso mediante una terminal de bash
Podemos acceder a cualquier contenedor abriendo una terminal de bash, para ello ejecutamos: 

 $ kubectl exec --stdin --tty my-mariadb-galera-0 -- /bin/bash
Donde my-mariadb-galera-0 es el nombre del conetenedor al que queremos acceder. Una vez dentro del contenedor nos conectamos a la base de datos de forma local como en cualquier otra terminal de bash ejecutando el comando mysql y los parámetros -u para el usuario y -p para la contraseña.
### 3. Acceso remoto
Podemos acceder de forma remota con un cliente de mariadb incluyendo el nombre del contenedor con el parámetro -h indicando el nombre del contenedor al que queremos acceder. Kubernetes se encarga de dar acceso a través del puerto 3306 gracias a la gestión de espacio de nombres que incorpora.

## 4. Autor

Autor del proyecto:
Rafael Siles Rodríguez
Alumno de 2º de ASIR del IES Gran Capitán (Córdoba)

Supervisor:
José Ramón Albendín Ramírez
Profesor de Informática del IES Gran Capitán (Córdoba)
