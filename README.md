# Nextcloud en kubernetes
Integracion de nextcloud en la plataforma de kubernetes

![guia](/images/imagen-0.png)

Antes de realizar la instalacion, verificar los archivos y modificar los valores a los que usara
- :key: **secrets** Este manifiesto estan las contraseñas de las demas aplicaciones
> [!IMPORTANT]
> en el archivo publicado estan credenciales de ejemplo, se recomienta cambiarlas al momento de pase a produccion
- :floppy_disk: **mariadb** Servira como base de datos para almacenar la informacion de configuracion y metadatos
- :dvd: **redis** Servira como almacenamiento cache de datos de navegacion dando mayor velocidad
- :open_file_folder: **nextcloud** Es el aplicativo principal, el cual creara nuestra nube privada
- :japanese_ogre: **clamav** Servira como antivirus para los archivos alojados y escaneara los archivos que se quierean subir
- :page_with_curl: **collabora** Servira como suite de ofimatica para los documentos via web 
---
## Requisitos previos
> [!TIP]
> Si no tienes instalado Kuberntes puedes usar esta [guia](https://pabpereza.dev/docs/cursos/kubernetes/instalacion_de_kubernetes_cluster_completo_ubuntu_server_con_kubeadm)
### :bookmark:Ingress Controller
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

### :pencil:Cert Manager
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

### :computer:MetalLB LoadBalancer
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

### :open_file_folder:NFS Provisioner
En este caso usaremos helm para crear el manifiesto para ello añadimos los repositorio 
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
```
creamos el manifiesto
> [!WARNING]
> Utilice sus propios datos
```
helm template nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=10.124.0.9 \
--set nfs.path=/data/nfs \
--set storageClass.onDelete=true > dynamic-nfs.yaml
```
Instalamos mediante manifiesto
```
kubectl apply -f dynamic-nfs.yaml
```
verificamos que ingress este ejecutandoce correctamente
``` 
kubectl get pods
```
```
NAME                                               READY   STATUS              RESTARTS   AGE
nfs-subdir-external-provisioner-5d8784c45d-764xk   1/1     Running             0          60s
```

## Instalacion de aplicaciones
En primera instancia instalamos los certificados
```
kubectl apply -f certificados/
```
posteriormente instalamos los manifiestos de ingress 
```
kubectl apply -f ingress/
```
verificamos que ingress este ejecutandoce correctamente
``` 
kubectl get pods
```
```
NAME                CLASS   HOSTS                       ADDRESS       PORTS     AGE
collabora-ingress   nginx   apu.pitvirtual.uni.edu.pe   172.16.8.88   80, 443   5d6h
nextcloud-ingress   nginx   apu.uni.edu.pe              172.16.8.88   80, 443   5d6h
Una vez revizado y edita los archivos procederemos a la ejecutar los manifiestos
```

Ahora podemos instalar las aplicaciones mediante los mifiestos
```
kubectl apply -f apps/
```
verificamos que estan ejecutandoce correctamente y Esperemos a que los pods esten en estado Runnig para poder continuar
``` 
kubectl get pods
```
``` 
NAME                                               READY   STATUS              RESTARTS   AGE
clamav-8667bfc7fc-6ds25                            0/1     ContainerCreating   0          6s
collabora-745b5cdc-nmr8d                           0/1     ContainerCreating   0          6s
mariadb-c6d7854df-m7kv5                            0/1     ContainerCreating   0          5s
nextcloud-5cc9dc666f-n9pm2                         0/1     Init:0/2            0          5s
nfs-subdir-external-provisioner-5d8784c45d-764xk   1/1     Running             0          3m
redis-686556cf6d-rddd5                             0/1     ContainerCreating   0          5s
```
Ingresamos a nextcloud desde nuestro navegador con el dominio configurado y configuramos las credenciales de **Admnistrador**
> [!IMPORTANT]
> La contraseña usada debe ser robusta para mayor seguridad.

![guia](/images/imagen-1.png)


Instalamos las aplicaciones que usaremos

![guia](/images/imagen-2.png)


Una vez instalado, nos redireccionara al dashboard de Nextcloud

![guia](/images/imagen-3.png)


Ingresamos al panel de administracion, seleccionamos el icono ubicado en la parte superior derecha > Configuraciones de administracion > Vista general
> [!NOTE]
> Dentro podemos verificar los errores que presnta Nextcloud, los cuales seran corregidos por los manifiesto de mantenimiento

![guia](/images/imagen-4.png)

## Correccion de errores
Ahora corregiremos la mayoria de errores que aparecen en Nextcloud
> [!NOTE]
> Puedes revisar los logs del pod en caso de algun error

### Manteniento de base de datos 
este manifiesto establece una pequeña ventana de manteniento, en el cual:
- Corrige las tablas de mariadb
- Añade tablas faltantes (si es el caso)
- Establece el manteniento automatico
- Borra la cache de redis

Para ello ejecutamos el manifiesto set-db.yaml
``` 
kubectl apply -f maintenance/set-db.yaml
```

### Establecer la region de predeterminada para teléfonos
este manifiesto añade la region para telefonos

Para ello ejecutamos el manifiesto set-db.yaml
``` 
kubectl apply -f maintenance/set-phone.yaml
```

### Establecer la configuracion de Proxy
Este manifiesto añade la configuracion necesaria para que Nextcloud funione correctamente con un proxy (ingress no es un proxy pero actua como tal)
``` 
kubectl apply -f maintenance/set-proxy.yaml
```

Aplicado la primara parte de los manifiestos de correccion, refrescamos la pagina y verificamos nuevamente los errores

![guia](/images/imagen-5.png)

## Configuracion de Nextcloud
las configuraciones que se va a aplicar es para optimizar el rendimiento y añadir una capa de seguridad mas

### Configuracion de cron

### Seguridad de dominio web
si tenemos publicado en internet nuestro servicio, aplicamos este manifiesto el cual añade una regla extra para que el servicio pase las pruebas de seguridad de nextcloud
``` 
kubectl apply -f maintenance/set-cookie.yaml
```

aplicado el manifiesto ingresamos a [scan.nextcloud.com](https://scan.nextcloud.com/) y comprobamos el nivel de seguridad

![guia](/images/imagen-6.png)
