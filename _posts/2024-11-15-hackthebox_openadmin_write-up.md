---
layout: post
title: Hack The Box OpenAdmin Write-Up
date: 2024-11-15
categories: [Write-Ups, HackTheBox, Machine, Easy]
tags: [OpenAdmin, Enumeration, HTTP]
---

![OpenAdmin Logo](/assets/post_details/openadmin/openadmin_logo.png)
# Hack The Box: OpenAdin Write-Up

## Enumeration
Realizamos la enumeración de puertos y de servicios, encontramos los puertos abiertos 80 (HTTP) y 22 (SSH), por lo que el escaneo indica toca enumerar la parte web.
```bash
# Escaneo todos los puertos abiertos
nmap -p- -sS -n --min-rate 5000 --open 10.10.10.171 -oN tcp_scan.txt
# Escaneo puertos abiertos encontrados
nmap -p80,22 -sCV -Pn -n 10.10.10.171 -oN full_scan.txt
```
![nmap](/assets/post_details/openadmin/openadmin_nmap.png)
### Directory Enum
La web no nos da nada en el directorio raiz, por lo que tendremos que hacer enumeración de directorios, con la herramienta **gobuster** encontramos los directorios `/music`,`/artwork`,`/sierra`.
```bash
gobuster dir -u http://10.10.10.171 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20
```
![directory enum](/assets/post_details/openadmin/openadmin_directory_enum.png)

Estas webs son las tipicas que haria una empresa para ofreces su tipo de servicios etc... Enumerando las webs manualmente, podemos apreciar que en el directorio `/music` existe un boton en el **header** llamado **Login** el cual nos redirige hacia el directorio `/ona`, de este modo encontramos un panel de la aplicación web **OpenNetAdmin**.
![button login ona](/assets/post_details/openadmin/openadmin_button_login_ona.png)
## FootHold
### OpenNetAdmin - CVE-2019-25065
**OpenNetAdmin** es una herramienta de administración de red basada en web que permite a los administradores gestionar y documentar la infraestructura de su red de manera centralizada y eficiente. Proporciona un conjunto de funcionalidades para monitorear, organizar y documentar recursos relacionados con la red, como direcciones IP, dispositivos, dominios, subredes, y más.

Nos encontramos con la siguiente aplicación web, en la que podemos ver nuestra version actual la cual el la **18.1.1**, para esta version de **OpenNetAdmin** existe la vulnerabilidad [CVE-2019-25065](https://nvd.nist.gov/vuln/detail/CVE-2019-25065), esta permite la inyección de comandos del sistema operativo debido a una neutralización inadecuada de las entradas del usuario. Esto significa que un atacante puede ejecutar comandos arbitrarios en el servidor, lo que puede resultar en escalamiento de privilegios y comprometer la seguridad del sistema. 
![ona](/assets/post_details/openadmin/openadmin_ona.png)

Buscamos por el exploit a traves de internet, y en la base de datos de exploits **exploit-db.com** encontramos el siguiente **exploit** [CVE-2019-25065](https://nvd.nist.gov/vuln/detail/cve-2019-25065) el cual copiamos y pegamos en un archivo en nuestro sistema.
![exploit ona](/assets/post_details/openadmin/openadmin_ona_exploit.png)

Una vez dados los permisos de ejecución al **exploit**, lo ejecutamos mediante `./exploit.sh <url>` y obtenemos acceso al sistema. Hay que tener en cuanta que estamos ejecutando comandos desde el dom o url, por lo que no podremos meternos en directorios.
![exploit bash](/assets/post_details/openadmin/openadmin_ona_exploit_bash.png)
## PrivEsc
### www-data -> jimmy
Enumeramos los usuarios mediante el **/etc/passwd**, los usuarios activos en el sistema son `root`, `jimmy`, `joanna`.
```bash
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
jimmy:x:1000:1000:jimmy:/home/jimmy:/bin/bash
joanna:x:1001:1001:,,,:/home/joanna:/bin/bash
```

En el directorio actual nos encontramos en los archivos de la aplicación web **OpenNetAdmin**, buscando por archivos nos topamos con una ruta hacia la configuración de la base de datos.
```bash
cat /config/config.inc.php
```
![path database](/assets/post_details/openadmin/openadmin_ona_path_databae.png)

En la ruta `local/config/database_settings.inc.php` existen credenciales, las cuales nos pueden servir para escalar privilegio hacia el siguiente usuario.
```bash
cat local/config/database_settings.inc.php
```
![database settings](/assets/post_details/openadmin/openadmin_ona_database_settings.png)
### jimmy -> joanna
Nos intentamos logear mediante **ssh** hacia el usuario **jimmy** con la credencial previamente obtenida, con la que obtenemos la conexión al usuario **jimmy**.
```bash
ssh jimmy@10.10.10.171
```
![jimmy ssh](/assets/post_details/openadmin/openadmin_jimmy_ssh.png)

Con el uso del comando **netstat -tulnp** podemos ver todos los procesos virtuales que se estan produciendo en la maquina, uno de ellos esta el puerto **52846** de la maquina en local. Existe un directorio en **apache** para la configuración de los sitios webs disponibles, `/etc/apache2/sites-available`, en este directorio nos encontramos con la configuración `internal.conf` en el que existe un **VirtualHost** escuchando por **127.0.0.1:52846**, y este proceso esta asignado al usuario **joanna**.
![apache sites](/assets/post_details/openadmin/openadmin_apache_sites_conf.png)

En la imagen anterior también aparece la propiedad **DocumentRoot** o raiz del documento la cual apunta hacia `/var/www/internal`. En esta ruta existen los archivos `index.php`, `logout.php`, `main.php`.
![internal files](/assets/post_details/openadmin/openadmin_internal_files.png)

En el archivo **index.php** podemos ver una aplicación login, en la que se esta validando con el **username** el usuario **jimmy** y un hash como **password** el cual lo podemos crackear utilizando la plataforma [hashes.com](https://hashes.com), esta hash corresponde al string **Revealed**.
![jimmy hash](/assets/post_details/openadmin/openadmin_jimmy_hash.png)
![hash crack](/assets/post_details/openadmin/openadmin_hash_revealed.png)
#### SSH TUNNELING
Realizamos **ssh-tunneling** mediante la flag **-L** al cual le asignamos el puerto remoto y local.
```bash
ssh jimmy@10.10.10.171 -L 52846:localhost:52846
```

Una vez hecho, visitamos en nuestra maquina local, el localhost por el puerto indicado, nos encontramos un **login** el cual nos pide la cuenta de usuario y la contraseña antes crackeada.
![login internal](/assets/post_details/openadmin/openadmin_login_internal.png)

Una vez nos logeamos aparece el ssh del usuario **joanna**, esto lo sabemos porque anteriormente analizando el codigo del archivo `main.php` sabemos que si se da exitoso el login se nos da la **id_rsa** del usuario **joanna**. Podemos ver que esta **id_rsa** esta encriptada y tambien se nos deja una keyword en el mensaje, `No te olvides de contraseña "ninja"`.
![joanna id_rsa](/assets/post_details/openadmin/openadmin_joanna_id_rsa.png)

Podemos crackear la clave privada encriptada con **john** aunque antes tenemos que sacar un hash de esta clave. Esto lo hacemos con la herramienta **ssh2john** la cual crea un hash desde la clave privada encriptada.
Por ultimo utilizamos **john** para crackear la contraseña, obteniendo con exito la credencial:
![john id_rsa](/assets/post_details/openadmin/openadmin_john_id_rsa.png)
### joanna -> root
Nos conectamos mediante **ssh** utilizando la calve privada encriptada, a la que le damos los permisos indicados e introducimos la contraseña como la **passphrase** de la clave **id_rsa**: 
```bash
# Permisos clave privada id_rsa
chmod 600 id_rsa
# Nos conectamos mediante ssh
ssh -i id_rsa joanna@10.10.10.171
```
![joanna ssh](/assets/post_details/openadmin/openadmin_joanna_ssh.png)

Una vez nos conectamos con exito, realizamos un `sudo -l` para comprobar si existen desconfiguraciones de sudo las cuales nos pueden llevar hacia escalar privilegio como superusuario.
![SUDORES](/assets/post_details/openadmin/openadmin_sudo-l.png)

En este caso podemos utilizar nano con permisos de root para modificar el archivo `/opt/priv`, aunque utilizando el siguiente paylaod de [GTFObins](https://gtfobins.github.io) podemos escalar privilegio a traves de **nano**.
```bash
# En nuestro caso cambiamos el XTERM, de kitty-xterm a xterm, para un mejor uso de nano
export XTERM=xterm
# Ejecutamos el sudoer
sudo /bin/nano /opt/priv
# Abrir archivo, ejecutar comando
^R^X
# Reseteamos y lanzamos sh
reset; sh 1>&0 2>&0
```

De esta manera obtenemos la **sh** como root:
![Get Root](/assets/post_details/openadmin/openadmin_get_root.png)