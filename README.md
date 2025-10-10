# Despliegue de Nextcloud en Kubernetes

Esta guía detalla el proceso para desplegar una instancia de **Nextcloud** de alta disponibilidad en un clúster de Kubernetes, integrando componentes esenciales como MariaDB, Redis, ClamAV y Collabora Online.

![guia](/imagenes/diagrama.png)

Antes de comenzar, es crucial revisar y modificar los valores en los archivos de manifiesto para que se ajusten a tu entorno.

  * 🔑 **`secrets.yaml`**: Este manifiesto contiene las credenciales para las aplicaciones.

> [\!IMPORTANT]
> Las credenciales proporcionadas en el repositorio son de ejemplo. **Debes cambiarlas** antes de pasar a un entorno de producción.

  * 💾 **MariaDB**: Actúa como la base de datos para almacenar metadatos y configuraciones de Nextcloud.

  * 💨 **Redis**: Funciona como un sistema de caché en memoria para mejorar la velocidad y el rendimiento de la interfaz.

  * 📂 **Nextcloud**: Es la aplicación principal que conforma nuestra nube privada.

  * 👹 **ClamAV**: Proporciona un escaneo antivirus para todos los archivos subidos a la plataforma.

  * 📝 **Collabora Online**: Sirve como una suite ofimática en línea, permitiendo la edición de documentos directamente desde el navegador.

-----

## 1\. Requisitos Previos

> [\!TIP]
> Si no tienes un clúster de Kubernetes, puedes seguir mi [guía de instalación](https://github.com/1237446/Instalacion-de-Kubernetes).

> [\!NOTE]
> Se recomienda instalar Nginx Proxy Manager (NPM) en una instancia separada, fuera del clúster de Kubernetes. Puedes seguir la [guía oficial](https://nginxproxymanager.com/guide/) y usar mi archivo `docker-compose.yml` como referencia.

### 🔗 Ingress Controller (nginx)

El Ingress Controller es necesario para exponer los servicios de Nextcloud y Collabora a la red externa.

  * **Agrega el repositorio de Helm:**
  
      ```bash
      helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
      helm repo update
      ```
  
  * **Instala el controlador:**
      
      ```bash
      helm install ingress-nginx ingress-nginx/ingress-nginx
      ```
  
  * **Verifica los Pods:**
  
      ```bash
      kubectl get pods -n ingress-nginx
      ```
      ```
      NAME                                       READY   STATUS    RESTARTS   AGE
      ingress-nginx-controller-578c564c54-ln8kq   1/1     Running   0          2m
      ```

-----

### 💻 MetalLB (LoadBalancer)

MetalLB asignará una dirección IP externa al servicio del Ingress Controller, haciéndolo accesible desde tu red local.

  * **Instala MetalLB desde su manifiesto oficial:**
  
      ```bash
      kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.3/config/manifests/metallb-native.yaml
      ```
  
  * **Verifica los Pods en el namespace `metallb-system`:**
  
      ```bash
      kubectl get pods -n metallb-system
      ```
      
      ```
      NAME                          READY   STATUS    RESTARTS   AGE
      controller-bb5f47665-nb55f    1/1     Running   0          2m
      speaker-bvj9j                 1/1     Running   0          2m
      ```
  
  * **Configura un Pool de IPs (Layer 2):**
      Crea un archivo `metallb-config.yaml` con el siguiente contenido.
  
      ```yaml
      apiVersion: metallb.io/v1beta1
      kind: IPAddressPool
      metadata:
        name: ingress-nginx-pool
        namespace: metallb-system
      spec:
        addresses:
        - 192.168.1.200-192.168.1.250 # ¡Cambia este rango!
        autoAssign: true
      ---
      apiVersion: metallb.io/v1beta1
      kind: L2Advertisement
      metadata:
        name: default-l2-advertisement
        namespace: metallb-system
      spec:
        ipAddressPools:
        - ingress-nginx-pool
        interfaces:
        - eno1
      ```
> [\!WARNING]
> Asegúrate de cambiar el rango en `addresses` por un rango de IPs que esté **libre** en tu red local.
  
  * **Aplica la configuración:**
  
      ```bash
      kubectl apply -f metallb-config.yaml
      ```
-----

### 🗄️ Almacenamiento Persistente

Necesitas una solución de almacenamiento para guardar los datos de forma persistente. A continuación se usara dos opciones populares.

#### NFS Subdir External Provisioner

Esta opción es ideal si ya tienes un servidor NFS en tu red.

  * **Instala el cliente NFS en todos los nodos del clúster:**
  
      ```bash
      sudo apt-get update
      sudo apt-get install nfs-common -y
      ```
  
  * **Agrega el repositorio de Helm:**
  
      ```bash
      helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
      ```

  * **Crea el manifiesto de configuración (`dynamic-nfs.yaml`):**
  
      ```bash
      helm template nfs-subdir-external-provisioner \
      nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
      --set nfs.server=10.124.0.9 \
      --set nfs.path=/data/nfs \
      --set storageClass.onDelete=true > dynamic-nfs.yaml
      ```

> [\!WARNING]
> Reemplaza `nfs.server` con la IP y `nfs.path` con la ruta de tu servidor NFS.

  * **Instala el provisionador:**
  
      ```bash
      kubectl apply -f dynamic-nfs.yaml
      ```
  
  * **Verifica que el Pod esté en ejecución:**
  
      ```bash
      kubectl get pods
      ```
      ```
      NAME                                       READY   STATUS    RESTARTS   AGE
      nfs-subdir-external-provisioner-5d8784c45d   1/1     Running   0          60s
      ```

#### Rook-Ceph

Rook-Ceph es una solución de almacenamiento distribuido nativa de la nube, ideal para entornos que requieren alta disponibilidad y escalabilidad.

  * **Agrega el repositorio de Helm de Rook:**
  
      ```bash
      helm repo add rook-release https://charts.rook.io/release
      helm repo update
      ```
  
  * **Despliega el operador de Rook-Ceph:**
  
      ```bash
      helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph
      ```
  
  * **Despliega el clúster de Ceph:**
  
      ```bash
      helm install --create-namespace --namespace rook-ceph rook-ceph-cluster \
         --set operatorNamespace=rook-ceph rook-release/rook-ceph-cluster
      ```
  
  * **Verifica los Pods:**
  La verificación de los pods en `rook-ceph` puede tomar varios minutos. Asegúrate de que todos los componentes, como `mgr`, `mon`, y `osd`, estén en estado `Running` o `Completed`.
  
      ```bash
      kubectl get pods -n rook-ceph
      ```
  
      *El resultado será una lista larga de pods correspondientes al clúster de Ceph.*

-----

## 2\. Despliegue de los Componentes de Nextcloud

Ahora, procederemos a instalar las aplicaciones que componen la solución.

  * **Crea el namespace para Nextcloud:**
  
      ```bash
      kubectl create namespace nextcloud
      ```
  
  * **Aplica los secretos con tus credenciales personalizadas:**
      
      ```bash
      kubectl apply -f secrets.yaml -n nextcloud
      ```

### MariaDB Operator

Utilizaremos el operador de MariaDB para gestionar nuestro clúster de base de datos de forma automatizada.

  * **Agrega el repositorio de Helm:**
  
      ```bash
      helm repo add mariadb-operator https://helm.mariadb.com/
      ```
  
  * **Instala el operador y sus CRDs:**
  
      ```bash
      helm install mariadb-operator mariadb-operator/mariadb-operator --create-namespace --namespace mariadb-operator
      ```
  
  * **Despliega el clúster de MariaDB Galera:**
  
      ```bash
      kubectl apply -f mariadb-galera.yaml -n nextcloud
      ```
  
  * **Verifica que los Pods del clúster estén listos:**
  
      ```bash
      kubectl get pods -n nextcloud
      ```
      
      ```
      NAME               READY   STATUS    RESTARTS   AGE
      mariadb-galera-0   2/2     Running   0          4m
      mariadb-galera-1   2/2     Running   0          4m
      mariadb-galera-2   2/2     Running   0          4m
      ```
  
  * **Crea los recursos de la base de datos (DB, usuario y permisos):**
  
      ```bash
      kubectl apply -f mariadb-extra.yaml -n nextcloud
      ```
  
  > [\!WARNING]
  > Aplica este manifiesto solo después de que todos los pods de `mariadb-galera` estén en estado `Running`.
  
  * **Verifica que los recursos se hayan creado correctamente:**
  
      ```bash
      # Verificar la base de datos
      kubectl get database -n nextcloud
      NAME        READY   STATUS    CHARSET   COLLATE           MARIADB          AGE     NAME
      nextcloud   True    Created   utf8      utf8_general_ci   mariadb-galera   7s 
      
      # Verificar el usuario
      kubectl get user -n nextcloud
      NAME                         READY   STATUS    MAXCONNS   MARIADB          AGE
      mariadb-galera-mariadb-sys   True    Created   20         mariadb-galera   5m
      nextcloud                    True    Created   50         mariadb-galera   10s
      
      # Verificar los permisos
      kubectl get grant -n nextcloud
      NAME                                     READY   STATUS    DATABASE    TABLE         USERNAME        GRANTOPT   MARIADB          AGE
      grant                                    True    Created   nextcloud   *             nextcloud     true       mariadb-galera   14s
      mariadb-galera-mariadb-sys-global-priv   True    Created   mysql       global_priv   mariadb.sys   false      mariadb-galera   5m
      ```

-----

### Redis Operator

Redis se utilizará para el almacenamiento en caché y el bloqueo de archivos, mejorando el rendimiento general.

  * **Agrega el repositorio de Helm:**
  
      ```bash
      helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
      ```
  
  * **Instala el operador de Redis:**
  
      ```bash
      helm install redis-operator ot-helm/redis-operator --create-namespace --namespace ot-operators
      ```
  
  * **Despliega el clúster de Redis con Sentinel para alta disponibilidad:**
  
      ```bash
      kubectl apply -f redis-sentinel.yaml -n nextcloud
      ```
  
  * **Verifica los Pods de Redis:**
  
      ```bash
      kubectl get pods -n nextcloud
      ```
      
      ```
      NAME                        READY   STATUS    RESTARTS   AGE
      redis-replication-0         1/1     Running   0          3m
      redis-replication-1         1/1     Running   0          3m
      redis-replication-2         1/1     Running   0          3m
      ```

## 3\. Instalación y Configuración de Nextcloud

### Despliegue de la Aplicación

  * ** Despliega Nextcloud:**
  
      ```bash
      kubectl apply -f nextcloud.yaml -n nextcloud
      ```

  * **Verifica el Pod y espera a que la instalación inicial finalice:**
      Puedes monitorear el progreso revisando los logs del pod.
      
      ```bash
      # Primero, obtén el nombre del pod
      kubectl get pods -n nextcloud
      nextcloud-75df488cff-fg4ll      1/1     Running     0          3s
      
      # Luego, revisa los logs (reemplaza el nombre del pod)
      kubectl logs -f -n nextcloud <nombre-del-pod-de-nextcloud>
      Nextcloud was successfully installed
      => Searching for hook scripts (*.sh) to run, located in the folder
      "/docker-entrypoint-hooks.d/post-installation"
      ==> Skipped: the "post-installation" folder is empty (or does not exist)
      Initializing finished
      => Searching for hook scripts (*.sh) to run, located in the folder
      "/docker-entrypoint-hooks.d/before-starting"
      ==> Skipped: the "before-starting" folder is empty (or does not exist)
      AH00558: apache2: Could not reliably determine the server's fully qualified domain
      name, using 10.244.1.10. Set the 'ServerName' directive globally to suppress this
      message
      ```

      La instalación estará completa cuando veas una línea similar a `Apache/2.4.62 (Debian) PHP/8.3.23 configured -- resuming normal operations

### Configuración del Acceso Externo (Ingress)

* **1. Despliega el Ingress para Nextcloud:**

```bash
kubectl apply -f ingress-nextcloud.yaml -n nextcloud
```

* **2. Verifica el Ingress:**

```bash
kubectl get ingress -n nextcloud
```

```
NAME                CLASS   HOSTS             ADDRESS         PORTS     AGE
nextcloud-ingress   nginx   nextcloud.midominio.com   192.168.1.200   80, 443   3s
```

Apunta el `HOST` configurado en tu Ingress a la `ADDRESS` (IP de MetalLB) en tu servidor DNS o en tu Nginx Proxy Manager.

Ingresamos al pod
```
kubectl exec -it -n nextcloud nextcloud-5dff54784f-pqmqn -- bash
```
```
Defaulted container "nextcloud" out of: nextcloud, wait-for-mariadb-cluster (init),
wait-for-redis-cluster (init)
root@nextcloud-5dff54784f-pqmqn:/var/www/html#
```

Actualizamos el pod e instalamos el editor de texto
```
root@nextcloud-5dff54784f-pqmqn:/var/www/html# apt update
root@nextcloud-5dff54784f-pqmqn:/var/www/html# apt install nano
```

Editamos el archivo config.php y añadimos lo que no está comentado
```
root@nextcloud-5dff54784f-pqmqn:/var/www/html# nano config/config.php
```
```php
//<?php
//$CONFIG = array (
//   'htaccess.RewriteBase' => '/',
//   'memcache.local' => '\\OC\\Memcache\\APCu',
//   'apps_paths' => 
//   array (
//     0 => 
//     array (
//       'path' => '/var/www/html/apps',
//       'url' => '/apps',
//       'writable' => false,
//     ),
//     1 => 
//     array (
//       'path' => '/var/www/html/custom_apps',
//       'url' => '/custom_apps',
//       'writable' => true,
//     ),
//   ),
//   'memcache.distributed' => '\\OC\\Memcache\\Redis',
//   'memcache.locking' => '\\OC\\Memcache\\Redis',
     'filelocking.enabled' => true,
//   'redis' => 
//   array (
//     'host' => 'redis-replication-master',
//     'password' => 'redis',
//     'port' => 6379,
       'timeout' => 1.5,
       'read_timeout' => 1.5,
//   ),
//   'upgrade.disable-web' => true,
//   'passwordsalt' => 'mFYaed0z3v4WOambO7lQS1fFPAoh4w',
//   'secret' => 'yG+NuLVJRs1fOKyMeaCIQuJFZL3YEKHHVTglv+3bsu2gD5RM',
      'trusted_domains' => 
      array (
       0 => 'localhost',
       1 => 'apu.uni.edu.pe',
      ),
//   'datadirectory' => '/var/www/html/data',
//   'dbtype' => 'mysql',
//   'version' => '31.0.7.1',
      'overwrite.cli.url' => 'http://domain.test.com',
      'overwritehost' => 'domain.test.com',
      'overwriteprotocol' => 'https',
      'trusted_proxies' => 
      array (
       0 => '10.0.0.0/8',
       1 => '192.168.1.0/24',
      ),
//   'dbname' => 'nextcloud',
//   'dbhost' => 'mariadb-galera',
//   'dbport' => '',
//   'dbtableprefix' => 'oc_',
//   'mysql.utf8mb4' => true,
//   'dbuser' => 'nextcloud',
//   'dbpassword' => 'redis',
      'asset-pipeline.enabled' => true,
      'asset-pipeline.minify' => true,
      'asset-pipeline.combine' => true,
      'has_internet_connection' => true,
      'check_for_working_htaccess' => true,
      'default_phone_region' => 'PE',
      'cookie_same_site' => 'Lax',
//   'installed' => true,
//   'instanceid' => 'oc3m9a2x0tcn',
      'enable_previews' => true,
      'enabledPreviewProviders' => 
      array (
        0 => 'OC\\Preview\\Image',
        1 => 'OC\\Preview\\MarkDown',
        2 => 'OC\\Preview\\MP3',
        3 => 'OC\\Preview\\TXT',
        4 => 'OC\\Preview\\OpenDocument',
        5 => 'OC\\Preview\\Movie',
        6 => 'OC\\Preview\\Krita',
        7 => 'OC\\Preview\\PNG',
        8 => 'OC\\Preview\\JPEG',
        9 => 'OC\\Preview\\GIF',
        10 => 'OC\\Preview\\HEIC',
        11 => 'OC\\Preview\\BMP',
        12 => 'OC\\Preview\\XBitmap',
        13 => 'OC\\Preview\\MKV',
        14 => 'OC\\Preview\\MP4',
        15 => 'OC\\Preview\\AVI',
        16 => 'OC\\Preview\\PDF',
      ),
      'preview_max_x' => '1024',
      'preview_max_y' => '1024',
      'jpeg_quality' => '60',
      'preview_max_memory' => '2048',
      'preview_max_filesize_image' => '50',
//);
```

Ejecutar job de correccion
> [!NOTE]
> - Corrige las tablas de mariadb
> - Añade tablas faltantes (si es el caso)
> - Establece el manteniento automatico
```
kubectl apply -f set-db.yaml
```

Verificar el job
```
kubectl get jobs -n nextcloud
```
```
NAME                      STATUS     COMPLETIONS   DURATION   AGE
nextcloud-db-repair-job   Complete   1/1           69s        2m23s
```

### Nginx Proxy Manager
> [!TIP]
> Si no tienes instalado NPM puedes usar el dockerfile **docker-compose.yml** de este repositorio
​
Ingresamos y creamos el certificado SSL

![guia](/imagenes/proxy-3.png)

Creamos proxy host
> [!NOTE]
> La dirección ip es la de ingress​

![guia](/imagenes/proxy-0.png)

![guia](/imagenes/proxy-1.png)

![guia](/imagenes/proxy-2.png)
```
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
client_max_body_size 0;
proxy_read_timeout 86400s;
proxy_hide_header Upgrade;
```

## Nextcloud dashboard
Ingresamos a nextcloud desde nuestro navegador con el dominio configurado e ingresamos las credenciales del secrets

![guia](/imagenes/nextcloud-0.png)

Ingresamos al panel de administración, seleccionamos el icono ubicado en la parte superior derecha > **Configuraciones de administración** > **Vista general** y verificamos que no haya errores

![guia](/imagenes/nextcloud-1.png)

Ingresamos al panel de administración, seleccionamos el icono ubicado en la parte superior derecha > **Configuraciones de administración** > **Ajustes básicos** y seleccionamos **cron**.

![guia](/imagenes/nextcloud-2.png)

Ejecutar cronjob
```
kubectl apply -f cron-nextcloud.yaml
```

Verificar los cronjobs
```
kubectl get cronjobs -n nextcloud
```
```
NAME             SCHEDULE       TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
nextcloud-cron   */5 * * * *    <none>     False     0        2m6s            6s
```

### Seguridad de dominio web
Si tenemos publicado en internet nuestro servicio, comprobamos el nivel de seguridad en [scan.nextcloud.com](https://scan.nextcloud.com/)

![guia](/imagenes/nextcloud-3.png)

## Clamav
Instalar clamav
```
kubectl apply -f clamav.yaml
```

Verificar los Pods
```
kubectl get pods -n nextcloud
```
```
NAME                            READY   STATUS      RESTARTS   AGE
clamav-59cc7b479f-mcnmd         1/1     Running     0          7m
```

Para la instalacion de ClamAV seleccionamos el icono ubicado en la parte superior derecha > Aplicaciones > Seguridad e instalamos el aplicativo **Antifirus for files**

![guia](/imagenes/clamav-0.png)

Regresamos a Configuraciones de administracion y nos dirigimos a **seguridad**, hasta la parte inferior donde estara el apartado de **Antivirus para archivos** en la cual configuramos de esta manera

- **modo:** Dominio de clamAV
- **dominio:** clamav
- **puerto:** 3310
- **Longitud de flujo:** 104857600

![guia](/imagenes/clamav-1.png)

Ahora nos dirigimos a **Archivos** e intentamos subir un archivo eicar, para probar el correcto funcionamiento de clamAV

> [!NOTE]
> Puedes descargar los archivos eicar de prueba [aqui](https://www.eicar.org/download-anti-malware-testfile/)

![guia](/imagenes/clamav-2.png)

## Collabora
Instalar collabora
```
kubectl apply -f collabora.yaml
```

Verificar los Pods
```
kubectl get pods -n nextcloud
```
```
NAME                            READY   STATUS      RESTARTS   AGE
collabora-5998759c75-w7txg      1/1     Running     0          7m
```

Instalar ingress de collabora
```
kubectl apply -f ingress-collabora.yaml
```

Verificar los ingress
```
kubectl get ingress -n nextcloud
```
```
NAME                CLASS   HOSTS                       ADDRESS        PORTS     AGE
collabora-ingress   nginx   apu.pitvirtual.uni.edu.pe   192.168.1.201  80, 443   3m
```

Para la instalacion de Collabora seleccionamos el icono ubicado en la parte superior derecha > Aplicaciones > Oficina y texto e instalamos el aplicativo **Nextcloud Office**

![guia](/imagenes/collabora-0.png)

Regresamos a Configuraciones de administracion y nos dirigimos a **Nextcloud Office**, seleccionamos **Use su propio servidor** e ingresamos la url del dominio de Collabora

![guia](/imagenes/collabora-1.png)

Ahora nos dirigimos a **Archivos** e ingresamos a Documentes y abrimos el documento **Welcome to Nextcloud Hub.docx**

![guia](/imagenes/collabora-2.png)

![guia](/imagenes/collabora-3.png)

## Outh2
Para la instalación seleccionamos el icono ubicado en la parte superior derecha > **Aplicaciones** > **Multimedia** e instalamos **Preview Generator**

---

Ingresamos a [la Consola de Desarrolladores de Google](https://console.developers.google.com/) crea un nuevo proyecto

---

En el menú lateral, ve a **APIs y servicios** > **Credenciales**

---

Haz clic en **Configurar pantalla de consentimiento de OAuth**

---

Ingresa un **Nombre de la aplicación** relevante (por ejemplo, *Nextcloud Login*).

---

Selecciona el tipo de usuario (Externo o Interno).

---

Ingresa una cuenta de información.

---

Crea el proyecto

---

De vuelta en **Credenciales**, haz clic en **Crear credenciales** > **ID de cliente de OAuth**.

---

Selecciona **Aplicación web** como **Tipo de aplicación** e ingresa un nombre para el cliente (ej.
*Nextcloud Google OAuth*).

---

En **Orígenes de JavaScript autorizados**, agrega la URL base de tu Nextcloud (ej. *https://tudominio.com*).
En **URIs de redireccionamiento autorizados**, agrega la siguiente URL, reemplazando *tudominio.com* con la URL de tu Nextcloud: **https://tudominio.com/apps/sociallogin/oauth/google**
Haz clic en **Crear**.

---

> [!IMPORTANT]
> Se te proporcionará un ID de cliente y un Secreto de cliente. Anótalos, los necesitarás en el siguiente paso.

> [!NOTE]
> en caso de no anotarlo lo puedes hayar **APIS y servicios** > **Credenciales**

En Nextcloud ingresamos al panel de administración, seleccionamos el icono ubicado en la parte superior derecha > **Configuraciones de administración** > **Social login**.

---

Desplázate hacia abajo hasta la sección de "Google".
Ingresa el ID de aplicación (App id) y el Secreto (Secret) que obtuviste de la Consola de Desarrolladores de Google.
Selecciona "Grupo predeterminado" (Default group) y selecciona Ninguno.
Haz clic en "Guardar"

---

Para probar el inicio de sesión abre una nueva ventana de navegador (o una ventana de incógnito/privada).
Ve a la URL de inicio de sesión de tu Nextcloud.
Deberías ver un botón para iniciar sesión con Google debajo del formulario de inicio de sesión normal
de Nextcloud.

---
