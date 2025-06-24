# Nextcloud-kubernetes
Integracion de nextcloud en la plataforma de kubernetes

---
Antes de realizar la instalacion, verificar los archivos y modificar los valores a los que usara
## Instalacion de aplicaciones
- :dark_sunglasses: **secrets** Este manifiesto estan las contraseñas de las demas aplicaciones
> [!NOTE]
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

