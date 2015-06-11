# k8s-CentOS
## Instalación de k8s en centOS
Detalle del proceso de instalación de Kubernetes en un entorno cluster con dos minions.

## Componentes de K8s
K8s funciona en modo cliente servidor con un master que gobierna los minions (nodos worker)
Pero K8s tiene varios componentes:
  * etcd - almacenamiento distribuido en modo key-value store para configuración y descubrimiento de servicios 
  * flannel - SDN fabric que se apoyua en etcd para contenedores 
  * kube-apiserver - proporciona el api de K8s 
  * kube-controller-manager - Gestiona los servicios de k8s
  * kube-scheduler - Encargado de balancear contenedores en los distintos workers 
  * kubelet - Implementa un manifest de un contenedor para que sean lanzados de acuerdo a como se han descrito 
  * kube-proxy - Proporciona servicios de haproxy 
	
## Componentes de K8s
Pasos a llevar a cabo:

1.- Deshabilitar iptables en cada nodo para evitar conflictos con las reglas de iptables de Docker.

		`sudo systemctl stop firewalld`
		`sudo systemctl disable firewalld`
		
2.- Instalar NTP y asegurar que esta activo y corriendo.
  `sudo yum -y install ntp`
  `sudo systemctl start ntpd`
  `sudo systemctl enable ntpd`
  
3.- Instalar k8s en el nodo MASTER
	`yum -y install etcd kubernetes`

3.1- Configura etcd en MASTER para que escuche de todas las ips. revisar el fichero etcd.conf con objeto de que las siguientes lineas estén descomentadas.

	* ETCD_NAME=default
	* ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
	* ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:4001"


3.2 Configura el api de Kubernetes en MASTER /etc/kubernetes/apiserver para garantizar que tiene los siguientes valores
	* KUBE_API_ADDRESS="--address=0.0.0.0"
	* KUBE_API_PORT="--port=8080"
	* KUBELET_PORT="--kubelet_port=10250"
	* KUBE_ETCD_SERVERS="--etcd_servers=http://127.0.0.1:4001"
	* KUBE_SERVICE_ADDRESSES="--portal_net=10.254.0.0/16"
	* KUBE_ADMISSION_CONTROL="--admission_control=NamespaceAutoProvision,LimitRanger,ResourceQuota"
	* KUBE_API_ARGS=""

3.3 Configura el controller manager en MASTER /etc/kubernetes/controller-manager
	* KUBELET_ADDRESSES="--machines=192.168.132.142,192.168.132.143"

3.4 Reiniciando servicios para que cojan los cambios  MASTER

for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
    sudo systemctl restart $SERVICES
    sudo systemctl enable $SERVICES
    sudo systemctl status $SERVICES 
done

3.5 Definiendo la configuración de  flannel network en etcd. esta configuracion será descargada por el servicio flannel de los minions:
	`etcdctl mk /coreos.com/network/config '{"Network":"172.17.0.0/16"}`

4 Configurando los minions - los siguientes pasos deben ser ejecutados en todos los minions que forman parte del cluster
4.1 Instalando flannel y kubernetes usando yum
		`sudo yum -y install flannel kubernetes`

4.2 Configurando el servicio etcd para el servicio de flannel. Hay que actualizar la siguiente linea en el fichero /etc/sysconfig/flanneld para conectar con master:
	`FLANNEL_ETCD="http://192.168.132.141:4001"`

4.3 configurando el servicio de configuración /etc/kubernetes/config de kubernetes para actualizar el valor del KUBE_MASTER
	`KUBE_MASTER="--master=http://192.168.132.141:8080"`

4.4 Configurando el servicio de kubelet en los minions de etc/kubernetes/kubelet
minion 1:
	`KUBELET_ADDRESS="--address=0.0.0.0"`
	`KUBELET_PORT="--port=10250"`
	`# change the hostname to this host’s IP address`
	`KUBELET_HOSTNAME="--hostname_override=192.168.132.142"`
	`KUBELET_API_SERVER="--api_servers=http://192.168.132.141:8080"`
	`KUBELET_ARGS=""`

minion 2:
	`KUBELET_ADDRESS="--address=0.0.0.0"`
	`KUBELET_PORT="--port=10250"`
	`# change the hostname to this host’s IP address`
	`KUBELET_HOSTNAME="--hostname_override=192.168.132.143"`
	`KUBELET_API_SERVER="--api_servers=http://192.168.132.141:8080"`
	`KUBELET_ARGS=""`

En cada nodo hay que eliminar el interfaz de red docker0  

`sudo /sbin/ifconfig docker0 down`
`sudo brctl delbr docker0`

Arrancando los servicios en cada uno de los minions
	 for SERVICES in kube-proxy kubelet docker flanneld; do 
	    sudo systemctl restart $SERVICES
	    sudo systemctl enable $SERVICES
	    sudo systemctl status $SERVICES 
	done

como resultado de esto, en cada minion deberia existir un par de interfaces de red docker0 y flannel0. deberia existir unos interfaces de red distintos para flannel en cada
uno de los minions
	`ip a | grep flannel | grep inet`


Como utilidad para el copy-paste... 
---------------------------------

parando los servicios de un nodo worker
 for SERVICES in kube-proxy kubelet docker flanneld; do 
    sudo systemctl stop $SERVICES
done


parando los servicios del master
for SERVICES in etcd kube-apiserver kube-controller-manager kube-scheduler; do 
    sudo systemctl stop  $SERVICES
 done

 for SERVICES in kube-proxy kubelet docker flanneld; do 
    sudo systemctl status $SERVICES 
done

for SERVICES in flanneld; do 
    sudo systemctl restart $SERVICES 
done


Para probar que todo esta corriendo
---------------------------------

kubectl create -f mysql.yaml
kubectl create -f mysql-service.yaml
