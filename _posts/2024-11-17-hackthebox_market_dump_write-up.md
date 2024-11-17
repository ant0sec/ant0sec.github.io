---
layout: post
title: Hack The Box Market Dump Write-Up
date: 2024-11-17
categories: [Write-Ups, HackTheBox, Challenge, Easy]
tags: [NetworkAnalysis, WireShark, Base58, CyberChef, Easy, Misc]
---
# Hack The Box: Market Dump Write-Up
**MarketDump** es un reto facil centrado en el analisis de paquetes con la ayuda de **wireshark**.

En la descripción del reto nos muestra que un hacker ha logrado acceder a la red local de una empresa, depues de pivotar mediante la plataforma web publica. Ha logrado hacer un bypass del login del programa de stocks de productos para asi obtener la base de datos de los clientes. Aunque se cree que solamente uno de los clientes fue afectado. Tendremos que buscar que cliente fue. 
## Enumeration
Empezamos por abrir la captura de paquetes con **wireshark** mediante `wireshark MarketDump.pcapng`, en la que podemos ver todos los paquetes. Casi todos los paquetes que contiene la captura son **TCP** y llevan incorparada la flag **reset**, la cual indica que el emisor considera que la conexión no debe continuar.
![Reset Flag](/assets/post_details/marketdump/marketdump_shark_reset_flag.png)

Analizamos todos los protocolos que tiene la captura, mediante **Statistics -> Protocol Hierarchy**.
Existen unos cuantos paquetes de **telnet**.
![Protocols](/assets/post_details/marketdump/marketdump_shark_protocols.png)

Filtramos por protocolo **telnet** para visualizar la **Data-Stream** de cada paquete.
![Telnet](/assets/post_details/marketdump/marketdump_shark_telnet.png)

En uno de estos paquetes con el protocolo **telnet**, existe en la **Data-Stream** credenciales de usuario y lo que puede ser una bienvenida de un login, y se nos muestra una serie productos.
![Data Stream](/assets/post_details/marketdump/marketdump_shark_data_stream.png)

Si visualizamos paquetes **TCP** en adelante, vemos el comando `nc.traditional -lvp 9999...` el cual nos esta mostrando el puerto de escucha.
![Data Stream Port](/assets/post_details/marketdump/marketdump_shark_data_stream_port.png)

Filtramos por el puerto con `tcp.port == 9999` y se nos muestra varios paquetes los cuales contienen la siguiente información:
```bash
whoami
root

wc -l costumers.sql
10302 costumers.sql

ls -la
total 344
drwxr-xr-x 2 vigil vigil   4096 Jul  9 13:55 .
drwxr-xr-x 6 root  root    4096 Jul  9 13:38 ..
-rwxr-xr-x 1 vigil vigil 333845 Jul  9 13:55 costumers.sql
-rw-r--r-- 1 root  root    1024 Jul  9 13:55 .costumers.sql.swp
-rwxr-xr-x 1 vigil vigil    593 Jul  9 13:14 login.sh

head -n2 costumers.sql
IssuingNetwork,CardNumber
American Express,377815700308782

cp costumers.sql /tmp/
cd /tmp
ls
config-err-lU04xV
costumers.sql
mozilla_vigil0
snap.1000_telegram-desktop_0UDXXk
ssh-8jVN4Kyx3X69
systemd-private-9ac4f21175984888b953531b43a88a47-apache2.service-lIsVqD
systemd-private-9ac4f21175984888b953531b43a88a47-bolt.service-Fd1LWs
systemd-private-9ac4f21175984888b953531b43a88a47-colord.service-rdNsnK
systemd-private-9ac4f21175984888b953531b43a88a47-fwupd.service-3d8iRg
systemd-private-9ac4f21175984888b953531b43a88a47-rtkit-daemon.service-pzu6lE
systemd-private-9ac4f21175984888b953531b43a88a47-systemd-resolved.service-ZtjIX4
systemd-private-9ac4f21175984888b953531b43a88a47-systemd-timesyncd.service-0BNKmh
Temp-bf8572b5-6aac-4c1d-aff6-063f56964ecb

python -m SimpleHTTPServer 9998
cat costumers.sql
```

Podemos ver que el atacante gana acceso como **root** en la maquina y copia el archivo **costumers.sql** en el directorio **/tmp** para asi luego transferirlo.

## How to find the flag
Siguiendo con el contenido del paquete, tenemos todo el contenido de la tabla el cual se muestran las columnas **IssuingNetwork** y **CardNumber**.
Vemos que solo se muestran numeros en la columna **CardNumber** hasta llegar a un string en lo que parace una cadena encriptada.
![Show Table](/assets/post_details/marketdump/marketdump_show_table.png)

Utilizamos **cyberchef**, pegamos la cadena y nos realiza automaticamente el decrypt de la cadena en **Base58**.
![Decrypt String](/assets/post_details/marketdump/marketdump_decrypt_string.png)

O en consola: 
![Decrypt Console](/assets/post_details/marketdump/marketdump_decrypt_console.png)

