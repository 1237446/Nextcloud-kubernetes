# Nextcloud-kubernetes
Integracion de nextcloud en la plataforma de kubernetes

---
## Prerequisites for on-premise cluster
> [!NOTE]
> Si no tienes instalado Kuberntes puedes usar esta guia https://pabpereza.dev/docs/cursos/kubernetes/instalacion_de_kubernetes_cluster_completo_ubuntu_server_con_kubeadm

### Ingress Controller
Instalamos NGINX Ingress Controller. Puedes cambiar la version 
``` 
https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
``` 
### Cert Manager
Required to change the 'email' field for create Let's Encrypt account:
cert-manager/prod_issuer.yaml cert-manager/staging_issuer.yaml

Apply/install cert-manager: kubectl apply -f ./cert-manager/
## MetalLB LoadBalancer
Editamos la configuración de kube-proxy en el clúster actual
``` 
kubectl edit configmap -n kube-system kube-proxy
```
Instalamos MetalLB mediante manifiesto
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```
> [!WARNING]
> Utilice sus propias direcciones IP cambiando el parámetro "addresses:" en el archivo metallb/ipaddresspool.yaml
```
kubectl apply -f metallb/ipaddresspool.yaml
```
En este caso usaremos Layer2 
```
kubectl apply -f metallb/layer2advertisement.yaml
```
### NFS Provisioner
In nfs-provisioner/nfs-deployment.yaml two different pods are deployed, one representing nfs for HDD and the other for SSD. The NFS server address can be changed in the following snippet:
env:
    - name: PROVISIONER_NAME
      value: ff1.dev/ssd 
    - name: NFS_SERVER
      value: 10.0.0.2 #Server IP
    - name: NFS_PATH
      value: /ssd01 #NFS server path


Antes de realizar la instalacion, verificar los archivos y modificar los valores a los que usara
## Instalacion de aplicaciones
- :dark_sunglasses: **secrets** Este manifiesto estan las contraseñas de las demas aplicaciones
> [!WARNING]
> en el archivo publicado estan credenciales de ejemplo, se recomienta cambiarlas al momento de pase a produccion
- :floppy_disk: **mariadb** Servira como base de datos para almacenar la informacion de configuracion y metadatos
- :dvd: **redis** Servira como almacenamiento cache de datos de navegacion dando mayor velocidad
- :open_file_folder: **nextcloud** Es el aplicativo principal, el cual creara nuestra nube privada
- :japanese_ogre: **clamav** Servira como antivirus para los archivos alojados y escaneara los archivos que se quierean subir
- :page_with_curl: **collabora** Servira como suite de ofimatica para los documentos via web 

Una vez revizado y edita los archivos procederemos a la ejecutar los manifiestos
Podemos ejecutar todos de una sola pasada
```
kubectl apply -f apps/
```
o individualmente cada uno
```
kubectl apply -f apps/mariadb.yaml
...
```
verificamos que estan ejecutandoce correctamente
``` 
kubectl get pods
```
Esperemaos a que los pods esten en estado Runnig para poder continuar
``` 
NAME                                               READY   STATUS      RESTARTS      AGE
clamav-66d94fb94b-lj287                            1/1     Running     0             5m
collabora-84ffc497f4-jmd4d                         1/1     Running     0             5m
mariadb-b4b4c9949-m88cf                            1/1     Running     0             5m
nextcloud-8767f454f-9zl7x                          1/1     Running     0             5m
nfs-subdir-external-provisioner-5d8784c45d-764xk   1/1     Running     0             5m
redis-6b468c499f-4jj7t                             1/1     Running     0             5m
```

