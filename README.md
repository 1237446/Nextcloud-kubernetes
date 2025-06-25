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
- :japanese_ogre: **clamAV** Servira como antivirus para los archivos alojados y escaneara los archivos que se quierean subir
- :page_with_curl: **collabora** Servira como suite de ofimatica para los documentos via web 
---
## Requisitos previos
> [!TIP]
> Si no tienes instalado Kuberntes puedes usar esta [guia](https://nginxproxymanager.com/guide/)

> [!NOTE]
> El server de NPM debes intarlo en otra instancia fuera de kubernetes. Puedes instalarlo siguiendo esta [guia](https://pabpereza.dev/docs/cursos/kubernetes/instalacion_de_kubernetes_cluster_completo_ubuntu_server_con_kubeadm)

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
Verificamos que ingress este ejecutandoce correctamente
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
Posteriormente instalamos los manifiestos de ingress 
```
kubectl apply -f ingress/
```
Verificamos que ingress este ejecutandoce correctamente
``` 
kubectl get pods
```
```
NAME                CLASS   HOSTS                       ADDRESS           PORTS     AGE
collabora-ingress   nginx   collabora.local.test        192.168.1.100     80, 443   5d6h
nextcloud-ingress   nginx   collabora.local.test        192.168.1.101     80, 443   5d6h
Una vez revizado y edita los archivos procederemos a la ejecutar los manifiestos
```

Ahora podemos instalar las aplicaciones mediante los mifiestos
```
kubectl apply -f apps/
```
Verificamos que estan ejecutandoce correctamente y Esperemos a que los pods esten en estado Runnig para poder continuar
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
Este manifiesto establece una pequeña ventana de manteniento, en el cual:
- Corrige las tablas de mariadb
- Añade tablas faltantes (si es el caso)
- Establece el manteniento automatico
- Borra la cache de redis

Para ello ejecutamos el manifiesto set-db.yaml
``` 
kubectl apply -f maintenance/set-db.yaml
```

### Establecer la region de predeterminada para teléfonos
Este manifiesto añade la region para telefonos

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

## Configuracion adicional de Nextcloud
Las configuraciones que se va a aplicar es para optimizar el rendimiento y añadir una capa de seguridad mas

### Configuracion de cron
Configuraremos los trabajos en segundo plano, para ello aplicamos el manifiesto cron
``` 
kubectl apply -f maintenance/cron-nextcloud.yaml
```

Ingresamos al panel de administracion, seleccionamos el icono ubicado en la parte superior derecha > Configuraciones de administracion > Ajustes basicos y seleccionamos cron. 
Ahora cada 5 minutos se ejecutara un job el cual ejecuta el archivo cron.php y podremos verlo en los pods para su monitoreo

![guia](/images/imagen-7.png)
``` 
NAME                                               READY   STATUS      RESTARTS      AGE
clamav-66d94fb94b-lj287                            1/1     Running     1             30m
collabora-84ffc497f4-jmd4d                         1/1     Running     0             30m
mariadb-b4b4c9949-m88cf                            1/1     Running     0             30m
nextcloud-8767f454f-9zl7x                          1/1     Running     0             30m
nextcloud-cron-29180975-fdvkr                      0/1     Completed   0             2m40s
nfs-subdir-external-provisioner-5d8784c45d-764xk   1/1     Running     1             41m
redis-6b468c499f-4jj7t                             1/1     Running     0             30m
```

### Seguridad de dominio web
Si tenemos publicado en internet nuestro servicio, aplicamos este manifiesto el cual añade una regla extra para que el servicio pase las pruebas de seguridad de nextcloud
``` 
kubectl apply -f maintenance/set-cookie.yaml
```

Aplicado el manifiesto ingresamos a [scan.nextcloud.com](https://scan.nextcloud.com/) y comprobamos el nivel de seguridad

![guia](/images/imagen-6.png)

## Integracion de Collabora y ClamAV a Nextcloud
Ahora realizaremos la integracion de clamav para el escaneo de los archivos y de collabora para el manejo de los documentos

### ClamAV
Para la instalacion de ClamAV seleccionamos el icono ubicado en la parte superior derecha > Aplicaciones > Seguridad e instalamos el aplicativo **Antifirus for files**

![guia](/images/imagen-8.png)

Regresamos a Configuraciones de administracion y nos dirigimos a **seguridad**, hasta la parte inferior donde estara el apartado de **Antivirus para archivos** en la cual configuramos de esta manera

- **modo:** Dominio de clamAV
- **dominio:** clamav
- **puerto:** 3310

![guia](/images/imagen-9.png)

Ahora nos dirigimos a **Archivos** e intentamos subir un archivo eicar, para probar el correcto funcionamiento de clamAV

> [!NOTE]
> Puedes descargar los archivos eicar de prueba [aqui](https://www.eicar.org/download-anti-malware-testfile/)

![guia](/images/imagen-10.png)

### Collabora
Para la instalacion de Collabora seleccionamos el icono ubicado en la parte superior derecha > Aplicaciones > Oficina y texto e instalamos el aplicativo **Nextcloud Office**

![guia](/images/imagen-11.png)

Regresamos a Configuraciones de administracion y nos dirigimos a **Nextcloud Office**, seleccionamos **Use su propio servidor** e ingresamos la url del dominio de Collabora

![guia](/images/imagen-12.png)

Ahora nos dirigimos a **Archivos** e ingresamos a Documentes y abrimos el documento **Welcome to Nextcloud Hub.docx**

![guia](/images/imagen-13.png)

![guia](/images/imagen-14.png)


