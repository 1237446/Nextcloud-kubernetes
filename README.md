# Nextcloud en kubernetes
Integracion de nextcloud en la plataforma de kubernetes

![guia](/imagenes/diagrama.png)

Antes de realizar la instalacion, verificar los archivos y modificar los valores a los que usara
- :key: **secrets** Este manifiesto estan las contraseñas de las demas aplicaciones
> [!IMPORTANT]
> en el archivo publicado estan credenciales de ejemplo, se recomienta cambiarlas al momento de pase a produccion
- :floppy_disk: **mariadb** Servira como base de datos para almacenar la informacion de configuracion y metadatos
- :dvd: **redis** Servira como almacenamiento cache de datos de navegacion dando mayor velocidad
- :open_file_folder: **nextcloud** Es el aplicativo principal, el cual creara nuestra nube privada
- :japanese_ogre: **clamAV** Servira como antivirus para los archivos alojados y escaneara los archivos que se quierean subir
- :page_with_curl: **collabora** Servira como suite de ofimatica para los documentos via web 

## Requisitos previos
> [!TIP]
> Si no tienes instalado Kuberntes puedes usar esta [guia](https://nginxproxymanager.com/guide/)

> [!NOTE]
> El server de NPM debes intarlo en otra instancia fuera de kubernetes. Puedes instalarlo siguiendo esta [guia](https://pabpereza.dev/docs/cursos/kubernetes/instalacion_de_kubernetes_cluster_completo_ubuntu_server_con_kubeadm)

### :bookmark:Ingress Controller
Agrega el repositorio de Helm
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Instala el controlador
```
helm install ingress-nginx ingress-nginx/ingress-nginx
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

### :computer:MetalLB LoadBalancer
Instala MetalLB mediante manifiesto
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

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
> Utilice sus propias direcciones IP cambiando el parámetro "addresses:" en el archivo layer2advertisement.yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ingress-nginx
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.200-192.168.1.250 # **¡Cambia este rango por uno libre en tu red!**
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
```

Aplica el manifiesto
```
kubectl apply -f layer2advertisement.yaml
```

### :open_file_folder:NFS Provisioner
Instala el paquete nfs en todos los nodos
```
sudo apt-get update
sudo apt-get install nfs-common -y
```

Agrega el repositorio de Helm
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
```

creamos el manifiesto
> [!WARNING]
> Utilice sus propios datos, ruta nfs y direccion ip del servidor nfs
```
helm template nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=10.124.0.9 \
--set nfs.path=/data/nfs \
--set storageClass.onDelete=true > dynamic-nfs.yaml
```

Instala el manifiesto
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

### :open_file_folder:Longhorn
Instala el paquete nfs en todos los nodos
```
sudo apt-get update
sudo apt-get install open-iscsi -y
sudo systemctl enable --now iscsid
sudo systemctl status iscsid
```

Crea el namespace
```
kubectl create namespace longhorn-system
```

Agrega el repositorio de Helm
```
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Instala el controlador
> [!NOTE]
> Verifica que la version sea la actual 
```
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.9.0
```

Verificar los Pods
```
kubectl get pods -n longhorn-system
```
```
NAMEREADYSTATUSRESTARTSAGE
NAME                                                     READY   STATUS    RESTARTS        AGE
csi-attacher-7bfc99fdf6-2r97s                            1/1     Running   0               14m
csi-attacher-7bfc99fdf6-k2bkv                            1/1     Running   0               14m
csi-attacher-7bfc99fdf6-kfvwq                            1/1     Running   0               14m
csi-provisioner-5bfbf9655-9pwbd                          1/1     Running   0               14m
csi-provisioner-5bfbf9655-qgn8r                          1/1     Running   0               14m
csi-provisioner-5bfbf9655-zgs6v                          1/1     Running   0               14m
csi-resizer-6d8bdc48b6-cvrt4                             1/1     Running   0               14m
csi-resizer-6d8bdc48b6-pzbpd                             1/1     Running   0               14m
csi-resizer-6d8bdc48b6-v4pv9                             1/1     Running   0               14m
csi-snapshotter-68467df45d-9wp2p                         1/1     Running   0               14m
csi-snapshotter-68467df45d-lklv4                         1/1     Running   0               14m
csi-snapshotter-68467df45d-m77jw                         1/1     Running   0               14m
engine-image-ei-e09d8caa-ggczp                           1/1     Running   0               20d
engine-image-ei-e09d8caa-p6kqx                           1/1     Running   0               20d
engine-image-ei-e09d8caa-sg6g6                           1/1     Running   0               20d
instance-manager-69a4c110e39850503288e780e2b60bc1        1/1     Running   0               20d
instance-manager-cbffff0651010bfef8a4203382d798d5        1/1     Running   0               14m
instance-manager-da8f677e9d223e452f1b0fd03fbd9c07        1/1     Running   0               14m
longhorn-csi-plugin-946ch                                3/3     Running   0               14m
longhorn-csi-plugin-qc6wl                                3/3     Running   0               14m
longhorn-csi-plugin-tzh8x                                3/3     Running   0               14m
longhorn-driver-deployer-54599c447c-696ks                1/1     Running   0               14m
longhorn-manager-42r8x                                   2/2     Running   0               14m
longhorn-manager-dvgmv                                   2/2     Running   0               14m
longhorn-manager-wz45f                                   2/2     Running   0               14m
longhorn-ui-6b44759848-b6bv2                             1/1     Running   0               14m
longhorn-ui-6b44759848-p6lfg                             1/1     Running   0               14m
share-manager-pvc-47121f3e-96ce-476f-8327-9b1aa9686606   1/1     Running   0               14m
share-manager-pvc-90fe8ee8-ea24-4860-bad3-5625dcdaf2d5   1/1     Running   0               14m
```

## Instalacion de aplicaciones
Creamos el namespace que usaremos para nextcloud
```
kubectl create namespace nextcloud
```

instalamos los secrets
```
kubectl apply -f secrets.yaml
```

### Mariadb Operador
Agrega el repositorio de Helm
```
helm repo add mariadb-operator https://helm.mariadb.com/mariadb-operator
```

Instala el controlador
```
helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds
helm install mariadb-operator mariadb-operator/mariadb-operator
```
Verificar los Pods
```
kubectl get pods
```
```
NAME                                                READY   STATUS    RESTARTS        AGE
mariadb-operator-5d4cb9794b-49djf                   1/1     Running   0               18s
mariadb-operator-cert-controller-6c974b5796-xb52s   1/1     Running   0               18s
mariadb-operator-webhook-6fdc55687d-hzrn5           1/1     Running   0               18s
```

Instalar mariadb
```
kubectl apply -f mariadb-galera.yaml
```

Verificar los Pods
```
kubectl get pods -n nextcloud
```
```
NAME                            READY   STATUS      RESTARTS   AGE
mariadb-galera-0                2/2     Running     0          4m
mariadb-galera-1                2/2     Running     0          4m
mariadb-galera-2                2/2     Running     0          4m
```

Instalar la base de datos, usuario y permisos
> [!WARNING]
> Aplique el manifiesto despues que todos los pods de mariadb estan en **Running**
```
kubectl apply -f mariadb-extra.yaml
```

Verificar la base de datos
```
kubectl get databases -n nextcloud
```
```
NAME        READY   STATUS    CHARSET   COLLATE           MARIADB          AGE     NAME
nextcloud   True    Created   utf8      utf8_general_ci   mariadb-galera   7s 
```

Verificar usuario
```
kubectl get user -n nextcloud
```
```
NAME                         READY   STATUS    MAXCONNS   MARIADB          AGE
mariadb-galera-mariadb-sys   True    Created   20         mariadb-galera   5m
nextcloud                    True    Created   50         mariadb-galera   10s
```

Verificar permisos de usuario
```
kubectl get grant -n nextcloud
```
```
NAME                                     READY   STATUS    DATABASE    TABLE         USERNAME      GRANTOPT   MARIADB          AGE
grant                                    True    Created   nextcloud   *             nextcloud     true       mariadb-galera   14s
mariadb-galera-mariadb-sys-global-priv   True    Created   mysql       global_priv   mariadb.sys   false      mariadb-galera   5m
```

### Redis Operador
Agrega el repositorio de Helm
```
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
```

Instala el controlador
```
helm install redis-operator ot-helm/redis-operator --namespace ot-operators --create-namespace
```

Verificar el Pod
```
kubectl get pods -n ot-operators
```
```
NAME                             READY   STATUS    RESTARTS        AGE
redis-operator-d9d5d8769-6kjht   1/1     Running   0               20s
```

Instalar redis
```
kubectl apply -f redis-sentinel.yaml
```

Verificar los Pods
```
kubectl get pods -n nextcloud
```
```
NAME                            READY   STATUS      RESTARTS   AGE
redis-replication-0             1/1     Running     0          3m
redis-replication-1             1/1     Running     0          3m
redis-replication-2             1/1     Running     0          3m
redis-sentinel-sentinel-0       1/1     Running     0          3m
redis-sentinel-sentinel-1       1/1     Running     0          3m
redis-sentinel-sentinel-2       1/1     Running     0          3m
```

## Nextcloud
Instalar nextcloud
```
kubectl apply -f nextcloud.yaml
```

Verificar el Pod
```
kubectl get pods -n nextcloud
```
```
NAME                            READY   STATUS      RESTARTS   AGE
nextcloud-75df488cff-fg4ll      1/1     Running     0          3s
```

Esperar a que se instale completamente nextcloud
> [!NOTE]
> El tiempo de instalacion dependera del tipo de instalacion usado
```
kubectl logs -n nextcloud nextcloud-5dff54784f-pqmqn
Defaulted container "nextcloud" out of: nextcloud, wait-for-mariadb-cluster (init),
wait-for-redis-cluster (init)
Configuring Redis as session handler
Initializing nextcloud 31.0.6.2 ...
New nextcloud instance
Installing with MySQL database
=> Searching for hook scripts (*.sh) to run, located in the folder
"/docker-entrypoint-hooks.d/pre-installation"
==> Skipped: the "pre-installation" folder is empty (or does not exist)
Starting nextcloud installation
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
AH00558: apache2: Could not reliably determine the server's fully qualified domain
name, using 10.244.1.10. Set the 'ServerName' directive globally to suppress this
message
7[Fri Aug 01 15:40:08.770988 2025] [mpm_prefork:notice] [pid 1:tid 1] AH00163:
Apache/2.4.62 (Debian) PHP/8.3.23 configured -- resuming normal operations
[Fri Aug 01 15:40:08.771023 2025] [core:notice] [pid 1:tid 1] AH00094: Command
line: 'apache2 -D FOREGROUND'
```

Instalar ingress de nextcloud
```
kubectl apply -f ingress-nextcloud.yaml
```

Verificar el ingress
```
NAME                CLASS   HOSTS                       ADDRESS        PORTS     AGE
nextcloud-ingress   nginx   apu.uni.edu.pe              172.16.9.246   80, 443   3s
```

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

Editamos el archivo config.php y añadimos lo que está comentado
```
root@nextcloud-5dff54784f-pqmqn:/var/www/html# nano config/config.php
```
```php
<?php
$CONFIG = array (
  'htaccess.RewriteBase' => '/',
  'memcache.local' => '\\OC\\Memcache\\APCu',
  'apps_paths' => 
  array (
    0 => 
    array (
      'path' => '/var/www/html/apps',
      'url' => '/apps',
      'writable' => false,
    ),
    1 => 
    array (
      'path' => '/var/www/html/custom_apps',
      'url' => '/custom_apps',
      'writable' => true,
    ),
  ),
  'memcache.distributed' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  //'filelocking.enabled' => true,
  'redis' => 
  array (
    'host' => 'redis-replication-master',
    'password' => 'redis',
    'port' => 6379,
  ),
  'upgrade.disable-web' => true,
  'passwordsalt' => 'mFYaed0z3v4WOambO7lQS1fFPAoh4w',
  'secret' => 'yG+NuLVJRs1fOKyMeaCIQuJFZL3YEKHHVTglv+3bsu2gD5RM',
  //'trusted_domains' => 
  //array (
  //  0 => 'localhost',
  //  1 => 'apu.uni.edu.pe',
  //),
  'datadirectory' => '/var/www/html/data',
  'dbtype' => 'mysql',
  'version' => '31.0.7.1',
  //'overwrite.cli.url' => 'http://apu.uni.edu.pe',
  //'overwritehost' => 'apu.uni.edu.pe',
  //'overwriteprotocol' => 'https',
  //'trusted_proxies' => 
  //array (
  //  0 => '10.0.0.0/8',
  //  1 => '192.168.1.0/24',
  //),
  //'forwarded_for_headers' => 
  //array (
  //  0 => 'HTTP_X_FORWARDED_FOR',
  //  1 => 'HTTP_X_FORWARDED',
  //  2 => 'HTTP_X_FORWARDED_PROTO',
  //),
  //'remote_address_header' => 'HTTP_X_REAL_IP',
  'dbname' => 'nextcloud',
  'dbhost' => 'mariadb-galera',
  'dbport' => '',
  'dbtableprefix' => 'oc_',
  'mysql.utf8mb4' => true,
  'dbuser' => 'nextcloud',
  'dbpassword' => 'redis',
  //'asset-pipeline.enabled' => true,
  //'asset-pipeline.minify' => true,
  //'asset-pipeline.combine' => true,
  //'has_internet_connection' => true,
  //'check_for_working_htaccess' => true,
  //'default_phone_region' => 'PE',
  //'cookie_same_site' => 'Lax',
  'installed' => true,
  'instanceid' => 'oc3m9a2x0tcn',
  //'enable_previews' => true,
  //'enabledPreviewProviders' => 
  //array (
  //  0 => 'OC\\Preview\\PNG',
  //  1 => 'OC\\Preview\\JPEG',
  //  2 => 'OC\\Preview\\GIF',
  //  3 => 'OC\\Preview\\BMP',
  //  4 => 'OC\\Preview\\XBitmap',
  //  5 => 'OC\\Preview\\MP3',
  //  6 => 'OC\\Preview\\TXT',
  //  7 => 'OC\\Preview\\MarkDown',
  //  8 => 'OC\\Preview\\SVG',
  //  9 => 'OC\\Preview\\Image',
  //  10 => 'OC\\Preview\\TIFF',
  //  11 => 'OC\\Preview\\HEIC',
  ),
);
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
> Si no tienes instalado NPM puedes usar el dockerfile de este repositorio
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
12proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
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
