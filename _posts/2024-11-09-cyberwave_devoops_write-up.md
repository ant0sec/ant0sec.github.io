---
layout: post
title: CyberWave DevOops Write-Up
date: 2024-11-09
categories: [Write-Ups, CyberWave, Machine, Easy]
tags: [devoops, Credential-Leak, CMS, WordPress, SimpleJobBoard, CVE-2020-35749, LFI, Docker-Breakout, Easy, Linux ]
---
![Devoops Logo](/assets/post_details/devoops/devoops_logo.png)
# Cyber Wave: DevOops Write-Up
La **maquina easy** [DevOops](https://training.cyberwave.network/machines) de [CyberWave](https://training.cyberwave.network) trata de una mal configuración en **CMS** alojado en la maquina, el cual tiene una vulnerabilidad **LFI**, llevando a cabo el acceso del sistema a traves de **Credential-Leak y Reuse**. En la parte del PrivEsc, debido a una mala configuración de permisos podemos llevar a cabo la famosa tecnica **Docker Breakout** para asi tomar el acceso del sistema.

## Enumeration
Realizamos la enumeración de puertos y de servicios con **nmap**, encontramos los puertos 22(SSH) y 80(HTTP), en el puerto 80 se puede ver que se esta alojando un wordpress.
```bash
nmap -p22,80 -Pn --min-rate 5000 -n -sCV --open 10.10.10.4 -oN full_scan.txt
```
![NMAP](/assets/post_details/devoops/devoops_nmap.png)

### WPScan
La dirección IP nos redirige al dominio **devoops.bsh** por lo que tendremos que editar el archivos **etc/hosts** para que esta redirección sea posible. 
Con la herramienta **wpscan** realizamos una enumeración de usuarios, plugins y temas, con nuestro API-Token, el cual lo podemos obtener gratuitamente desde la web de wpscan registrando una cuenta. De esta manera se mostrara mas info, como puede ser que vulnerabilidades pueden darse en lo anterior dicho a enumerar.
```bash
wpscan --url http://devoops.bsh/wordpress --enumerate u,p,t --api-token <API TOKEN>
```
#### Users Enumeration
Se identifican los usuarios **admin** y **sarah**.
![WPScan Users](/assets/post_details/devoops/devoops_wpscan_users.png)
#### Credential Leak
También encontramos la siguiente ruta, la cual permite un **Directory-Listing**, se nos muestra una seria de carpetas subidas al wordpress, en la que podrian estar subidas credenciales para algunos de los usuarios.
![WPScan Directory Upload](/assets/post_details/devoops/devoops_wpscan_directory_upload.png)

Y en efecto tenemos las posibles credenciales del usuario **sarah** en la ruta `http://devoops.bsh/wordpress/wp-content/uploads/2022/02/credentials.txt`.
![Credential-Leak](/assets/post_details/devoops/devoops_credential_leak.png)

## FootHold
### CVE-2020-35749 - LFI - Simple Job Board
A traves del plugin **Simple Job Board**, el cual lo hemos enumerado anteriormente con **wpscan** encontramos que para la versión 2.9.3 [CVE-2020-35749](https://www.exploit-db.com/exploits/50721) existe un **LFI** desde el cual nos podemos bajar los archivos, como **/etc/hosts** o **/etc/passwd**...
![WPScan Plugin](/assets/post_details/devoops/devoops_wpscan_plugin_vuln.png)

Mediante la ruta `post.php?post=application_id&action=edit&sjb_file=` realizamos el **LFI**, pudiendo descargar el archivo **/etc/passwd** enumeramos los usuarios del sistema entre ellos **root** y **john**.
![Show LFI](/assets/post_details/devoops/devoops_show_lfi.png)

Intentamos descargar la clave privada de **ssh** sin exito, por lo que intentamos descargar el archivo de configuración de wordpress **wp-config.php**, en el cual encontramos otra credencial de usuario.
![LFI wp-config.php](/assets/post_details/devoops/devoops_lfi_wpconfig.png)

Esta credencial la utilizamos para conectarnos mediante **ssh** con el usuario **john** y obtenemos un resultado exitoso.
![SSH FootHold](/assets/post_details/devoops//devoops_sshfoothold.png)

## PrivEsc
### Docker BreakOut
Una vez en el sistema buscamos por posibles escaladas, una de ellas puede ser a traves de docker ya que es una herramienta muy famosa en el ecosistema **devops** y la imagen de la maquina es una cola de una ballena, por lo que todo a punta a **docker** ya que su logo es una ballena.
Vemos que el usuario **john** tiene permisos de ejecución.
![Docker Privilege](/assets/post_details/devoops//devoops_docker_privilege.png)

Mostramos si existe alguna imagen en el sistema con la que poder trabajar y encontramos la imagen **alpine**, esta imagen esta basada en la distribucion linux de **Alpine**.
![Docker BreakOut](/assets/post_details/devoops//devoops_docker_breackout.png)

Con esta imagen nos podemos crear un contenedor, con la opción de montura, con la cual podemos cambiar de permisos al binario bash para asi utilizarlo con privilegio y escalar mayor privilegios.
```bash
docker run --rm -dit -v /:/mnt/root --name privesc alpine

docker exec -it privesc sh
```

Una vez en el contenedor cambios permisos hacia el binario **bash**, de esta manera sera **SUID**, por lo que lo podremos ejecutar de manera temporal con permisos de **superusuario**.
```bash
chmod u+s /bin/bash
```
![SUID 1](/assets/post_details/devoops/devoops_suid_1.png)

Con **bash -p** ejecutamos **bash** con el mayor privilegio el cual permite el binario, en este caso obtener permisos como superusuario, de esta manera ya hemos rooteado la maquina. So nos quedaria obtener la flag, y para una mayor persistencia la clave privada de root para conectarnos a traves de **ssh**.
![SUID 2](/assets/post_details/devoops/devoops_suid_2.png)
