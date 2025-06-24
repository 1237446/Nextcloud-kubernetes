# Nextcloud en kubernetes
Integracion de nextcloud en la plataforma de kubernetes

Antes de realizar la instalacion, verificar los archivos y modificar los valores a los que usara
- :dark_sunglasses: **secrets** Este manifiesto estan las contraseñas de las demas aplicaciones
> [!WARNING]
> en el archivo publicado estan credenciales de ejemplo, se recomienta cambiarlas al momento de pase a produccion
- :floppy_disk: **mariadb** Servira como base de datos para almacenar la informacion de configuracion y metadatos
- :dvd: **redis** Servira como almacenamiento cache de datos de navegacion dando mayor velocidad
- :open_file_folder: **nextcloud** Es el aplicativo principal, el cual creara nuestra nube privada
- :japanese_ogre: **clamav** Servira como antivirus para los archivos alojados y escaneara los archivos que se quierean subir
- :page_with_curl: **collabora** Servira como suite de ofimatica para los documentos via web 
---
## Requisitos previos
> [!NOTE]
> Si no tienes instalado Kuberntes puedes usar esta [guia](https://pabpereza.dev/docs/cursos/kubernetes/instalacion_de_kubernetes_cluster_completo_ubuntu_server_con_kubeadm)
### Ingress Controller
Instalamos NGINX Ingress Controller. Puedes cambiar la version 
``` 
https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml
```
> [!NOTE]
> Si sale algun error, crea el namespaces ingress-nginx

Verificar los Pods
```
kubectl get pods -n ingress-nginx
```
```
NAME                                        READY   STATUS    RESTARTS        AGE
ingress-nginx-controller-578c564c54-ln8kq   1/1     Running   0)              2m
```

### Cert Manager
Requerido para cambiar el campo 'email' para crear una cuenta Let's Encrypt:
Instalación de los CustomResourceDefinitions (CRDs)
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.1/cert-manager.crds.yaml
```
Instalación de los Componentes Principales de cert-manager
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.1/cert-manager.yaml
```
> [!NOTE]
> Si sale algun error, crea el namespaces cert-manager

Verificar los Pods
```
kubectl get pods -n cert-manager
```
```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6997b69b-l2p7p                1/1     Running   0          2m
cert-manager-cainjector-654b9d7994-fg5kl   1/1     Running   0          2m
cert-manager-webhook-65786f5c8c-vghj9      1/1     Running   0          2m
```

### MetalLB LoadBalancer
Editamos la configuración de kube-proxy en el clúster actual
``` 
kubectl edit configmap -n kube-system kube-proxy
```
Instalamos MetalLB mediante manifiesto
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```
> [!NOTE]
> Si sale algun error, crea el namespaces metallb-system

Verificar los Pods
```
kubectl get pods -n metallb-system
```
```
NAME                          READY   STATUS    RESTARTS   AGE
controller-bb5f47665-nb55f    1/1     Running   0          2m
speaker-bvj9j                 1/1     Running   0          2m
speaker-wxdbl                 1/1     Running   0          2m
```
En este caso usaremos Layer2 
> [!WARNING]
> Utilice sus propias direcciones IP cambiando el parámetro "addresses:" en el archivo metallb/ipaddresspool.yaml
```
kubectl apply -f ipaddresspool.yaml
```

### NFS Provisioner
En este caso usaremos helm para crear el manifiesto para ello añadimos los repositorio 
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
```
creamos el manifiesto
> [!WARNING]
> Utilice sus propios datos
```
helm install nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=10.124.0.9 \
--set nfs.path=/data/nfs \
--set storageClass.onDelete=true
```

## Instalacion

Una vez revizado y edita los archivos procederemos a la ejecutar los manifiestos
Podemos ejecutar todos de una sola pasada
```
kubectl apply -f apps/
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










