1.	Requisitos previos
Los requisitos mínimos de hardware para instalar kubernetes en entornos Linux son los siguientes:
•	2GB de RAM por máquina anfitrión
•	2CPUs
•	Conexión de red completa entre las máquinas
•	Nombre de host, dirección MAC y product_uuid únicos para cada nodo
•	Swap desactivado para que kubelet funcione correctamente
Todos los comandos ejecutados se realizarán con permisos de sudo o con el usuario root, hay que prestar atención a que estamos ejecutando porque un error puede inutilizar el sistema.
2.	Instalación de Kubernetes
2.1.	Docker
Para instalar Kubernetes, previamente debemos instalar Docker en nuestro sistema operativo, este actuará como controlador de cgroup. Tanto el tiempo de ejecución del contenedor como el kubelet tienen una propiedad llamada "controlador cgroup", la cual es importante para la gestión de cgroups en máquinas Linux.
la instalación se realiza desde los repositorios de Docker ejecutando:

 
$ apt install docker.io apt-transport-https


Una vez instalado, creamos el demonio JSON que actuará de controlador de los contenedores de kubernetes en el directorio /etc/docker con el siguiente contenido:

 
$ cat /etc/docker/daemon.json << EOF
 {
   "exec-opts": ["native.cgroupdriver = systemd"],
   "log-driver": "json-file",
   "log-opts": {
     "max-size": "100m"
  },
   "storage-driver": "overlay2"
 }
EOF


Reiniciamos Docker y lo habilitamos para que este se inicie al inicia el sistema.

 
$ systemctl restart docker
$ systemctl enable Docker


2.2.	Iptables
Debemos asegurarnos que el módulo br_netfilter está cargado, para ello ejecutamos: 

 
$ lsmod | grep br_netfilter 
br_netfilter           28672  0
bridge                176128  1 br_netfilter


	

En el caso que no estuviera cargado, se carga ejecutando:

 
$ modprobe br_netfilter


Como requisito para que iptables vea correctamente el tráfico, debemos asegurarnos que net.bridge.bridge-nf-call-iptables tenga como valor 1, para ello creamos el fichero de configuración k8s.conf en el directorio /etc/sysctl.d con el siguiente contenido y ejecutamos sysctl --system

 
$ cat > /etc/sysctl.d/k8s.conf  <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
$ sysctl –system


Una vez ejecutado tenemos que reiniciar la máquina para que se aplique por completo la instalación.
Por último, desactivamos la swap y comentamos la línea que la activa al iniciar el sistema en el fichero /etc/fstab:

 
$ swapoff -a


Tenemos que asegurarnos que los siguientes puertos están habilitados para la conexión entre los nodos:
a)	Nodos del plano de control
Protocolo	Dirección	Puertos	Servicio	Uso
TCP	Entrante	6443*	Kubernetes API server	Todos
TCP	Entrante	2379-2380	etcd server client API	Kube-APIserver, etcd
TCP	Entrante	10250	Kubelet API	Kubelet API, plano de control
TCP	Entrante	10251	Kube-scheduler	Kube-scheduler
TCP	Entrante	10252	Kube-controller-manager	Kube-controller-manager

b)	Nodos de trabajo
Protocolo	Dirección	Puertos	Servicio	Uso
TCP	Entrante	10250	Kubelet API	Kubelet API, plano de control
TCP	Entrante	30000-32767	Servicio de NodePort	Todas

*El puerto 6443 es el único que puede ser personalizado, los demás puertos son los indicados y tienen que estar habilitados. 
3.	Instalación de kubeadm, kubelet y kubectl
Instalamos ahora los siguientes complementos:

•	Kubeadm: comando para inicializar el clúster.
•	Kubelet: el componente que se ejecuta en todas las máquinas de su clúster y hace cosas como iniciar Pods y contenedores.
•	kubectl: la utilidad de línea de comando para hablar con su clúster.

Para ejecutar contenedores en Pods, Kubernetes usa container runtime. De forma predeterminada, Kubernetes usa la Interfaz de tiempo de ejecución del contenedor (CRI) para interactuar con el tiempo de ejecución de su contenedor elegido.

Si no especifica un tiempo de ejecución, kubeadm automáticamente intenta detectar un tiempo de ejecución de contenedor instalado, escaneando una lista de sockets de dominio Unix conocidos. A continuación, se enumeran los tiempos de ejecución del contenedor y sus rutas de socket asociadas:

Docker	 /var/run/dockershim.sock
Containerd	/run/containerd/containerd.sock
CRI-O	/var/run/crio/crio.sock

Si se detectan tanto Docker como containerd, Docker tiene prioridad. Esto es necesario porque Docker 18.09 se envía con containerd y ambos son detectables incluso si solo instaló Docker. Si se detectan otros dos o más tiempos de ejecución, kubeadm sale con un error.

La instalación de cada uno de ellos se realiza desde los repositorios de kubernetes, para ello, primero añadimos los repositorios a nuestro sistema.
 

$ curl -fsSLo /usr/share/keyrings/kubernetes-archive- keyring.gpg https://packages.cloud.google.com/apt/doc/apt- key.gpg

$ echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list


Una vez añadido el repositorio, ya podemos instalar los paquetes como lo realizamos habitualmente desde los repositorios. Nos interesa fijar la versión para evitar posibles problemas con las actualizaciones, para ello ejecutaremos el comando apt-mark hold.


$ apt update
$ apt install kubelet kubeadm kubectl
$ apt-mark hold kubelet kubeadm kubectl

4.	Configuración del nodo maestro
Para levantar el nodo maestro se utiliza el comando kubeadm init, que se le pueden añadir los siguientes parámetros:

•	--apiserver-advertise-address: con esta opción indicamos la dirección IP del nodo maestro, si no se indica, al iniciarse el nodo maestro se tomará la dirección IP de la tarjeta de red.
•	--control-plane-kubeadm: se usa para añadir un nombre de dominio en lugar de la dirección IP del nodo maestro.
•	--pod-network-cidr: establece la dirección IP del proveedor de Pod.
•	--cri-socket: detecta el tiempo de ejecución del contenedor en Linux mediante la lista de socket conocidas, para usar uno diferente se utiliza este argumento.
 

$ sudo kudeadm init –apserver-advertise-address 192.168.12.144 –pod-network-cidr 10.244.0.0/16


Este comando lanza el nodo maestro. La operación tarda varios minutos, si todo es correcto el mensaje final será similar al siguiente:


Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
https://kubernetes.io/docs/concepts/cluster-administration/addons/


Para poder utilizar kubernetes sin permisos de sudo, debemos de ejecutar las ordenes que nos indica al finalizar el proceso de instalación, las cuales son crear el directorio oculto .kube en el home del usuario, copiar en este el contenido del fichero de configuración y cambiarle los permisos a dicho archivo para que pueda ejecutarlo el usuario.

4.1.	Configuración del Pod Network
El Pod Network es un complemento de red para que los nodos puedan comunicarse entre sí creando redes privadas virtuales de forma automática. Existe un gran número de Pod Network que se pueden implementar en kubernetes los cuales, algunos de ellos se listan a continuación:
 
 
•	ACI
•	Antrea
•	AOS from Apstra
•	AWX VPC CNI
•	Azure CNI
•	Big Cloud Fabric
•	Calico
•	Cilium
•	Flannel
•	Google Compute Engine
•	Kube-OVN
 
 
Se va a utilizar Flannel por la sencillez de la implementación en kubernetes. Para instalarlo ejecutamos el siguiente comando:


$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel
/master/Documentation/kube-flannel.yml


Este crea automáticamente todos los demonios necesarios para su correcto funcionamiento. Una vez creados podemos comprobar el estado del nodo maestro con el comando kubectl get nodes, si la instalación es correcta debemos ver algo similar a esto:


$ kubectl get nodes

NAME            STATUS   ROLES    AGE    VERSION
dlp.srv.world   Ready    master   7m7s   v1.18.8


 
De igual forma podemos comprobar el estado y los servicios que están corriendo en el nodo maestro ejecutando:


$ kubectl get pods -A
NAMESPACE     NAME                                   READY   STATUS   RESTARTS    AGE
kube-system   coredns-66bff467f8-968bw               1/1     Running   0          7m47s
kube-system   coredns-66bff467f8-xd2m7               1/1     Running   0          7m47s
Kube-system   etcd-dlp.srv.world                     1/1     Running   0          7m56s
kube-system   kube-apiserver-dlp.srv.world           1/1     Running   0          7m55s
kube-system   kube-controller-manager-dlp.srv.world  1/1     Running   0          7m56s
kube-system   kube-flannel-ds-amd64-k8fbp            1/1     Running   0          85s
kube-system   kube-proxy-h588r                       1/1     Running   0          7m47s
kube-system   kube-scheduler-dlp.srv.world           1/1     Running   0          7m56s


4.2.	Implementación de un nodo secundario
Una vez en marcha el nodo primario, podemos añadir tantos nodos como deseemos al clúster, estos se comunicarán de forma segura con el nodo primario a través de una conexión cifrada con algoritmo sha256. 

Como nodo secundario utilizaremos un servidor idéntico al anterior, donde se ha instalado Docker, kubeadm, kubectl y kubelet. Para lanzar el nodo ejecutamos el siguiente comando en el nodo primario:


$ kubeadm token create --print-join-command


Este comando nos devolverá el comando que tenemos que ejecutar en el nuevo servidor para levantar el nodo secundario, el cual es similar a este:


$ kubeadm join 192.168.12.144:6443 --token mf7wmw.i6lzmcweb1k5fhan \
--discovery-token-ca-cert-hash sha256: a2fe3ff26b45163db734448366299b00ab111754ac5cd8f0b941ea4a0ad07e11


Este comando lo podemos ejecutar en cuantos nodos queramos para formar el clúster, una vez ejecutado, si volvemos a comprobar el estado del clúster con el comando kubectl get nodes, nos encontraremos una respuesta similar a esta:

$ kubectl get nodes

NAME               STATUS   ROLES    AGE    VERSION
dlp.srv.world      Ready    master   135m   v1.18.8
node01.srv.world   Ready    <none>   132m   v1.18.8

Podemos comprobar los servicios habilitados por kubernetes filtrando por los nodos al ejecutar el comando kubctl get pods -A -o wide.

kubectl get pods -A -o wide | grep node03

NAMESPACE     NAME                       READY STATUS  RESTARTS AGE IP         NODE             NOMINATED   NODE READINESS GATES
kube-system  kube-flannel-ds-amd64-558tf 1/1   Running 0        16m 10.0.0.53  node03.srv.world <none>      <none>
kube-system   kube-proxy-h79c2           1/1   Running 0        16m 10.0.0.53  node03.srv.world <none>      <none>

 
5.	Referencias

Docker Documentation. 2021. Install Docker Engine on Ubuntu. [online] Available at: <https://docs.docker.com/engine/install/ubuntu/>.
Kubernetes. 2021. Installing Kubernetes with deployment tools. [online] Available at: <https://kubernetes.io/docs/setup/production-environment/tools/>.
Server-world.info. 2021. Ubuntu 20.04 LTS : Kubernetes : Install Kubeadm : Server World. [online] Available at: <https://www.server-world.info/en/note?os=Ubuntu_20.04&p=kubernetes&f=1>.
