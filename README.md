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

> [!WARNING]
> Para funcionar correctamente, Rook-Ceph requiere que los nodos de almacenamiento dispongan de discos o particiones en bruto (sin formato). Si necesitas preparar un nodo con esta configuración, puedes seguir mi [guía de instalación](https://github.com/1237446/Nextcloud-kubernetes/blob/main/imagenes/Instalaci%C3%B3n%20de%20Ubuntu%20server%20para%20Rook-Ceph%20(1).pdf).
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
      helm install --create-namespace --namespace rook-ceph rook-ceph rook-release/rook-ceph -f values.yaml
      ```
  
  * **Despliega el clúster de Ceph:**
  
      ```bash
      helm install --create-namespace --namespace rook-ceph rook-ceph-cluster \
         --set operatorNamespace=rook-ceph rook-release/rook-ceph-cluster -f values.yaml
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
      helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds
      helm install mariadb-operator mariadb-operator/mariadb-operator
      ```
  
  * **Despliega el clúster de MariaDB Galera:**
  
      ```bash
      kubectl apply -f mariadb-galera.yaml -n nextcloud
      ```
  
  * **Verifica que los Pods del clúster estén listos:**
  
      ```bash
      kubectl get pods -n nextcloud
      
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
      kubectl get database,user,grant -n nextcloud
      
      NAME                                 READY   STATUS              CHARSET   COLLATE           MARIADB          AGE   NAME
      database.k8s.mariadb.com/nextcloud   False   MariaDB not found   utf8      utf8_general_ci   mariadb-galera   19s   
      
      NAME                             READY   STATUS              MAXCONNS   MARIADB          AGE
      user.k8s.mariadb.com/nextcloud   False   MariaDB not found   20000      mariadb-galera   19s
      
      NAME                          READY   STATUS              DATABASE    TABLE   USERNAME    GRANTOPT   MARIADB          AGE
      grant.k8s.mariadb.com/grant   False   MariaDB not found   nextcloud   *       nextcloud   true       mariadb-galera   19s
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

      NAME                        READY   STATUS    RESTARTS   AGE
      redis-replication-0         1/1     Running   0          3m
      redis-replication-1         1/1     Running   0          3m
      redis-replication-2         1/1     Running   0          3m
      ```

## 3\. Instalación y Configuración de Nextcloud

### Despliegue de la Aplicación

  * **Despliega Nextcloud:**
  
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

  * **Despliega el Ingress para Nextcloud:**
  
      ```bash
      kubectl apply -f ingress-nextcloud.yaml -n nextcloud
      ```
  
  * **Verifica el Ingress:**
  
      ```bash
      kubectl get ingress -n nextcloud
      NAME                CLASS   HOSTS             ADDRESS         PORTS     AGE
      nextcloud-ingress   nginx   nextcloud.midominio.com   192.168.1.200   80, 443   3s
      ```

Apunta el `HOST` configurado en tu Ingress a la `ADDRESS` (IP de MetalLB) en tu servidor DNS o en tu Nginx Proxy Manager.

### Ajustes Finales de Configuración

Para un rendimiento óptimo y una configuración correcta detrás de un proxy inverso, debemos editar el archivo `config.php`.

  * **Ingresa al Pod de Nextcloud:**
  
      ```bash
      # Reemplaza con el nombre de tu pod
      kubectl exec -it -n nextcloud <nombre-del-pod-de-nextcloud> -- bash
      ```
  
  * **Instala un editor de texto (ej. nano) y edita el archivo de configuración:**
  
      ```bash
      # Dentro del pod
      apt update && apt install -y nano
      nano config/config.php
      ```
  
  * **Añade las siguientes líneas** dentro del array `$CONFIG = array (...)`. Asegúrate de reemplazar `nextcloud.midominio.com` con tu dominio real.

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
     Guarda el archivo (`Ctrl+O`) y sal (`Ctrl+X`).

### Mantenimiento de la Base de Datos

Aplica un Job para realizar conversiones y reparaciones necesarias en la base de datos de Nextcloud.

  * **Ejecuta el Job:**
  
      ```bash
      kubectl apply -f set-db.yaml -n nextcloud
      ```
  
  * **Verifica que se haya completado:**
  
      ```bash
      kubectl get jobs -n nextcloud
      NAME                      COMPLETIONS   DURATION   AGE
      nextcloud-db-repair-job   1/1           69s        2m23s
      ```

-----

## 4\. Configuración de Nginx Proxy Manager

Si usas Nginx Proxy Manager, sigue estos pasos para configurar el acceso a Nextcloud.

  * **Crea o importa tu certificado SSL.**

      ![guia](/imagenes/proxy-3.png)

  * **Crea un nuevo Proxy Host.**

      ![guia](/imagenes/proxy-0.png)

> [\!NOTE]
> La dirección IP que debes usar es la que te asignó MetalLB a tu Ingress Controller.

  * **Configura la pestaña de SSL.**

      ![guia](/imagenes/proxy-1.png)
    
  * **Agrega la configuración avanzada.**
    
      En la pestaña **Advanced**, pega el siguiente código:

      ![guia](/imagenes/proxy-2.png)
    
      ```nginx
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-Proto $scheme;
      client_max_body_size 0;
      proxy_read_timeout 86400s;
      proxy_hide_header Upgrade;
      ```

-----

## 5\. Configuración de Nextcloud

  * **Accede a Nextcloud** a través de tu navegador usando el dominio que configuraste. Inicia sesión con las credenciales definidas en tu archivo `secrets.yaml`.

      ![guia](/imagenes/nextcloud-0.png)

  
  * **Ve a la configuración de administración** (icono de perfil \> **Ajustes de administración**) y revisa la sección **Vista general**. No debería mostrarse ninguna advertencia o error de configuración.

      ![guia](/imagenes/nextcloud-1.png)
  
  * **Configura las tareas en segundo plano.** En **Ajustes básicos**, selecciona la opción **Cron** para manejar las tareas de fondo.

      ![guia](/imagenes/nextcloud-2.png)
  
  * **Despliega el CronJob en Kubernetes** para que ejecute las tareas de mantenimiento de Nextcloud cada 5 minutos.
      
      ```bash
      kubectl apply -f cron-preview.yaml -n nextcloud
      ```
  
  * **Verifica que el CronJob se haya creado:**
  
      ```bash
      kubectl get cronjobs -n nextcloud
      ```
      
      ```
      NAME             SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
      preview-cron     */10 * * * *   False     0        <none>          6s
      ```

### Seguridad del Dominio Web

Si tu servicio está expuesto a internet, puedes verificar su nivel de seguridad usando el escáner oficial de Nextcloud: [scan.nextcloud.com](https://scan.nextcloud.com/)

![guia](/imagenes/nextcloud-3.png)

## 6\. Integración de Aplicaciones Adicionales

### Preview Generator

  * **Despliega el CronJob en Kubernetes** para que ejecute las tareas de mantenimiento de Nextcloud cada 5 minutos.
      
      ```bash
      kubectl apply -f cron-nextcloud.yaml -n nextcloud
      ```
  
  * **Verifica que el CronJob se haya creado:**
  
      ```bash
      kubectl get cronjobs -n nextcloud
      ```
      
      ```
      NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
      nextcloud-cron   */5 * * * * False     0        <none>          6s
      ```

  * **Ingresa al Pod de Nextcloud:**
  
      ```bash
      # Reemplaza con el nombre de tu pod
      kubectl exec -it -n nextcloud <nombre-del-pod-de-nextcloud> -- bash
      ```

  * **Instala el programa sudo**
  
      ```bash
      # Dentro del pod
      apt update && apt install -y sudo
      ```
  
  * **Ejecutar el siguiente comando:**
  
      ```bash
      # Generacion de de imagenes de previsualizacion
      sudo -u www-data ./occ preview:generate-all -vvv
      ```

### Antivirus con ClamAV

  * **Despliega ClamAV:**
  
      ```bash
      kubectl apply -f clamav.yaml -n nextcloud
      ```
  
  * **Verifica que el Pod esté en ejecución:**
  
      ```bash
      kubectl get pods -n nextcloud
      NAME                      READY   STATUS    RESTARTS   AGE
      clamav-59cc7b479f-mcnmd   1/1     Running   0          7m
      ```
  
  * **Configura la integración en Nextcloud:**
  
    * Ve a **Aplicaciones** (icono de perfil \> Aplicaciones) \> **Seguridad** e instala **Antivirus for files**.

      ![guia](/imagenes/clamav-0.png)
  
    * Regresa a **Ajustes de administración** \> **Seguridad**. En la sección "Antivirus para archivos", configura lo siguiente:
  
        * **Modo:** Demonio de ClamAV (Socket)
        * **Host:** `clamav`
        * **Puerto:** `3310`
        * **Longitud de flujo:** `104857600`

      ![guia](/imagenes/clamav-1.png)
  
    * Guarda la configuración y prueba subiendo un [archivo de prueba EICAR](https://www.eicar.org/download-anti-malware-testfile/) para confirmar que es bloqueado.

      ![guia](/imagenes/clamav-2.png)

### Suite Ofimática con Collabora Online

  * **Despliega Collabora:**
  
      ```bash
      kubectl apply -f collabora.yaml -n nextcloud
      ```
  
  * **Verifica que el Pod esté en ejecución:**
  
      ```bash
      kubectl get pods -n nextcloud
      NAME                        READY   STATUS    RESTARTS   AGE
      collabora-5998759c75-w7txg   1/1     Running   0          7m
      ```
  
  * **Despliega el Ingress para Collabora:**
      Este paso expone Collabora en su propio subdominio.
  
      ```bash
      kubectl apply -f ingress-collabora.yaml -n nextcloud
      ```
  
    Asegúrate de configurar el DNS para este nuevo subdominio (ej. `office.midominio.com`) para que apunte a la misma IP de tu Ingress.
  
  * **Configura la integración en Nextcloud:**
  
    * Ve a **Aplicaciones** \> **Oficina y texto** e instala **Nextcloud Office**.

      ![guia](/imagenes/collabora-0.png)
  
    * Regresa a **Ajustes de administración** \> **Nextcloud Office**.
  
    * Selecciona **Usar tu propio servidor** e ingresa la URL de tu instancia de Collabora (ej. `https://collabora.local.test`).

      ![guia](/imagenes/collabora-1.png)      
  
    * Guarda los cambios. Ahora deberías poder crear y editar documentos de Office directamente en Nextcloud.

      ![guia](/imagenes/collabora-2.png)
      
      ![guia](/imagenes/collabora-3.png)

-----

## 6\. Configurar Inicio de Sesión con Google (OAuth 2.0)

### Configuración en la Google Cloud Console

* #### Crear un nuevo proyecto

  Primero, dirígete a la [Consola de Desarrolladores de Google](https://console.cloud.google.com/projectcreate).

  * **Nombre del proyecto:** Asígnale un nombre descriptivo, como "Nextcloud Login".
  * **Organización (si aplica):** Selecciona tu organización o déjalo como "Sin organización".
  * Haz clic en **Crear**.

* #### Habilitar la API necesaria

  Para que la autenticación funcione, debemos habilitar la **Google People API**.

  * En el menú de navegación (`☰`), ve a **APIs y servicios** \> **Biblioteca**.
  * En la barra de búsqueda, escribe `Google People API` y selecciónala.
  * Haz clic en el botón **Habilitar**.

* #### Configurar la Pantalla de Consentimiento de OAuth

  Esta es la pantalla que verán tus usuarios cuando intenten iniciar sesión por primera vez.

  * En el menú de navegación, ve a **APIs y servicios** \> **Pantalla de consentimiento de OAuth**.

  * **Tipo de usuario:**

      * **Interno:** Solo para usuarios dentro de tu organización de Google Workspace.
      * **Externo:** Para cualquier usuario con una cuenta de Google. (Esta es la opción más común).

  * Haz clic en **Crear**.

  * **Información de la app**

      * **Nombre de la aplicación:** Un nombre relevante para tus usuarios (ej. *Acceso a Nextcloud*).
      * **Correo electrónico de asistencia del usuario:** Tu correo electrónico.
      * **Información de contacto del desarrollador:** Ingresa uno o más correos electrónicos.
      * Haz clic en **Guardar y continuar**.

  * **Permisos (Scopes)**

      * No es necesario añadir permisos aquí para el inicio de sesión básico. Haz clic en **Guardar y continuar**.

  * **Usuarios de prueba**

      * Puedes añadir correos de usuarios para probar la configuración antes de publicarla. Es opcional.
      * Haz clic en **Guardar y continuar**.

  * **Publicar la aplicación**

      * Vuelve al panel de la **Pantalla de consentimiento de OAuth** y haz clic en **Publicar la aplicación** para que esté disponible para todos tus usuarios.

* #### Crear las Credenciales (ID de cliente de OAuth)

  Ahora generaremos las claves que conectarás con Nextcloud.

  * En el menú de navegación, ve a **APIs y servicios** \> **Credenciales**.

  * Haz clic en **+ CREAR CREDENCIALES** y selecciona **ID de cliente de OAuth**.

  * **Tipo de aplicación:** Selecciona **Aplicación web**.

  * **Nombre:** Asígnale un nombre para identificarla (ej. *Cliente Web de Nextcloud*).

  * **Orígenes de JavaScript autorizados:**

      * Haz clic en **+ AÑADIR URI**.
      * Ingresa la URL base de tu Nextcloud (ej. `https://nextcloud.test.local`).

  * **URIs de redireccionamiento autorizados:**

      * Haz clic en **+ AÑADIR URI**.
      * Ingresa la siguiente URL, reemplazando `nextcloud.test.local` con tu dominio real:
        ```
        https://nextcloud.test.local/apps/sociallogin/oauth/google
        ```

  * Haz clic en **Crear**.

> [\!IMPORTANT]
> Se te mostrará un **ID de cliente** y un **Secreto del cliente**. Cópialos y guárdalos en un lugar seguro. Los necesitarás en el siguiente paso.

> [\!NOTE]
> Si no los anotaste, puedes encontrarlos nuevamente en la sección **APIs y servicios** \> **Credenciales**, haciendo clic en el nombre del cliente que acabas de crear.

-----

### Configuración en Nextcloud

* #### Ingresar Credenciales en Social Login

  * En tu Nextcloud, ve a tu panel de administrador.
  * Navega a **Configuración** (en el menú bajo tu icono de perfil) \> **Social login** (en el panel lateral izquierdo).
  * Busca la sección de **Google**.
  * Rellena los siguientes campos:
        * **App id:** Pega el **ID de cliente** que obtuviste de Google.
        * **Secret:** Pega el **Secreto del cliente**.
  * **"Default group":** Si deseas que los nuevos usuarios que se registren con Google sean añadidos automáticamente a un grupo en Nextcloud, selecciónalo aquí. Si prefieres gestionar los usuarios manualmente o no permitir el registro automático, déjalo en **Ninguno**.
  * Haz clic en **Guardar**.

-----

### Verificación

Para probar que la integración funciona correctamente:

  * Abre una nueva ventana de navegador en modo incógnito o privado.
  * Ve a la página de inicio de sesión de tu Nextcloud.
  * Debajo del formulario tradicional de usuario y contraseña, ahora deberías ver un botón para **Iniciar sesión con Google**.
  * Haz clic en él y sigue el proceso de autenticación de Google. Si todo está correcto, deberías acceder a tu cuenta de Nextcloud.
