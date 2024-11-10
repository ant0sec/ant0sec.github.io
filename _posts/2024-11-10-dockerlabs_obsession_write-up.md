---
layout: post
title: DockerLabs Obsession Write-Up
date: 2024-11-10
categories: [Write-Ups, DockerLabs, Machine, Very-Easy]
tags: [obsession, Enumeration, Data-Leak, Force-Brute, Misconfigured-SUDO, Very-Easy, Linux ]
---

![Obsession Logo](/assets/post_details/obsession/obsession_logo.png)
# DockerLabs: Obsession Write-Up
La **maquina very-easy** [Obsession](https://dockerlabs.es) de [DockerLabs](https://dockerlabs.es) trata una enumeración por servicios **FTP**, **HTTP** y ataques de **fuerza bruta** para conseguir el **FootHold**.
Escalada de privilegios sencilla mediante desconfiguraciones de **SUDO**.

## Enumeration
Realizamos un escaneo de puertos y de servicios, en los que encontramos abiertos los puertos **21(FTP), 22(SSH), 80(HTTP)**, la imagen muestra que a traves del ftp, se permite el login de usuarios Anonimos, en los que se encuentran varios documentos de texto. En el puerto 80, se encuentra una sobre **coaching** de ejercicio. 
```bash
sudo nmap -p21,22,80 -sCV --open 172.17.0.2 -oN full_scan.txt
```
![Obsession NMAP](/assets/post_details/obsession/obsession_nmap.png)

Nos conectamos al ftp con el usuario **Anonymous** de la siguiente manera, y descargamos los ficheros:
```bash
ftp Anonymous@172.17.0.2
get chat-ponza.txt
get pendientes.txt
```
En esta nota, nos muestra un chat con los posibles usuarios.
![Obsession Notes](/assets/post_details/obsession/obsession_note2.png)

En la nota **pendientes.txt**, seria un posible **TODO** de las tareas que hay que hacer, la tarea 4 podemos leer que existen varias configuraciones erroneas en el equipo, sobre permisos que no son del todo seguros.
![Obsession Notes](/assets/post_details/obsession/obsession_note1.png)

### Directory Enumeration
Con la herramienta **gobuster** enumeramos varios directorios en los que encontramos información lekeada.
```bash
gobuster dir -u http://172.17.0.2 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20
```
![Obsession Enum Directory](/assets/post_details/obsession/obsession_enum_directory.png)

### Data Leak
En el directorio **backup** encontramos un archivo **backup.txt** el cual contiene nombre de usuario **russoski**, el cual esta siendo utilizado para todos los servicios.
![Obsession Data Leak](/assets/post_details/obsession/obsession_data_leak.png)

### Force Brute With Hydra
Tenemos en cuenta esta ultimo por lo que con el usuario **russoski** realizamos un ataque de fuerza bruta con **hydra** hacia el servicio **ftp** encontrando la credencial **iloveme**, la cual nos servira para conectarnos a traves de **ssh** con el usuario **russoski**.
```bash
hydra -l russoski -P /usr/share/wordlists/rockyou.txt 172.17.0.2 ftp
```
![Obsession Hydra](/assets/post_details/obsession/obsession_hydra.png)

## PrivEsc
### Misconfigured SUDO
Con `sudo -l` vemos todas las configuracions de sudoers que estan permitidas para el usuario **russoski** y encontramos una en el binario **vim**, esta puede ser el permiso mal habilitado.
![Obsession Sudo](/assets/post_details/obsession/obsession_sudo-l.png)

Existe una manera con la que se puede subir de privilegio con este sudoers, a traves de vim, obtenemos root:
```bash
sudo vim -c ':!/bin/sh'
```
![Obsession Root](/assets/post_details/obsession/obsession_get_root.png)