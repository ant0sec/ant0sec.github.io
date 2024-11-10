---
layout: post
title: Hack The Box Heist Write-Up
date: 2024-11-10
categories: [Write-Ups, HackTheBox, Machine, Easy]
tags: [Heist, Enumeration, SMB, HTTP, Password-Spraying, RID-BruteForcing, ProcDump, Cisco-Credentials, Easy, Windows]
---

![Heist Logo](/assets/post_details/heist/heist_logo.png)
# Hack The Box: Heist Write-Up
La **maquina easy** [Heist](https://app.hackthebox.com/machines/201) de [Hack The Box](https://app.hackthebox.com) trata una enumeración por servicios **SMB**, **HTTP** y ataques de **fuerza bruta** como **Password Spraying** para conseguir el **FootHold**.
A traves de procesos del sistema con la herramienta **ProcDump** encontramos credenciales para el usuario de mayor privilegio.

## Enumeration
Realizamos una enumeracion de puertos y de servicios en los que se revelan servicios como **IIS http, rpc, smb, winrm**.
```bash
nmap -p135,445,49669,5985,80 -sCV --open 10.10.10.149 -Pn
```
![NMAP](/assets/post_details/heist/heist_nmap.png)

### RPC - SMB
Enumeracion de servicios de cuentas nulas y usuarios por defecto sin exito alguno:
```bash
rpcclient 10.10.10.149 -U ''
rpcclient 10.10.10.149 -U 'guest'

crackmapexec smb 10.10.10.149 -u '' -p ''
crackmapexec smb 10.10.10.149 -u 'guest'
```

### HTTP
Encontramos en el servicio web, un login, el cual podemos logearnos como **guest**.
![WebSite Login](/assets/post_details/heist/heist_website_login.png)

Encontramos una serie de posts los cuales hablan de una configuracion de un router de **Cisco**:
![Chat](/assets/post_details/heist/heist_chat.png)
![Config Leak](/assets/post_details/heist/heist_leak_config.png)

Encontramos varias credenciales hasheadas, buscamos en configuraciones de **Cisco** que estas contraseñas son de tipo 5 y 7, las cuales pueden ser crackeadas con esta [heramienta](https://www.firewall.cx/cisco/cisco-routers/cisco-type7-password-crack.html).
![Cisco Password 1](/assets/post_details/heist/heist_decrypt_cisco_password_1.png)
![Cisco Password 2](/assets/post_details/heist/heist_decrypt_cisco_password_2.png)
## FoothHold
Utilizamos **john** para crackear el secreto encontrado:
```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```
![John Password](/assets/post_details/heist/heist_john.png)

### Password Spraying
Realizamos un ataque de **Password Spraying** con la herramienta **CME**, utilizando un diccionario de usuarios el cual previamente hemos enumerado:
```bash
crackmapexec smb 10.10.10.149 -u users.txt -p 'stealth1agent'
```
![CME Password Spray 1](/assets/post_details/heist/heist_cme_password_spray_1.png)

El usuario `Hazard` no esta en el grupo `Remote Management Users` por lo que no nos logea. Conectandonos a traves de **smbclient** con las credenciales validas no obtenemos ningunos datos.

### RID BruteForcing
Tratamos de enumerar usuarios usando **RID Bruteforce**, **RID** se refiere al **Relative Identifier**, este es una parte del **SID (Security Identifier)** usado unicamente para identificar a un usuario o servicio en un host de Windows.
![RID Structure](/assets/post_details/heist/heist_rid_structure.png)

El dominio o identificador local es una constante dada por la maquina, mientras que el **RID** es unico, por lo que podemos tratar de enviar una query hacia `Local Computer Identifier`, de esta forma enumeraremos todos los **SIDs** validos.
```bash
# --rid-brute
crackmapexec smb 10.10.10.149 -u hazard -p 'stealth1agent' --rid-brute
```
![CME RDI Brute](/assets/post_details/heist/heist_cme_rid_brute.png)

### Password Spraying All Users
Una vez tenemos otra lista de usuarios, realizamos otro ataque **Password Spraying** con las credenciales encontradas anteriormente de las configuraciones de **Cisco**.
```bash
crackmapexec smb 10.10.10.149 -u ad_users.txt -p 'Q4)sJu\Y8qz*A3?d'
```
![CME Password Spray 2](/assets/post_details/heist/heist_cme_password_spray_2.png)
## PrivEsc
Nos conectamos con la herramienta **evil-winrm** a traves del protocolo **winrm**, en el **Desktop** del usuario **Chase** encontramos un **todo** en el que se muestra que el usuario deberá estar chequeando:
![TODO Notes](/assets/post_details/heist/heist_todo_notes.png)

Realizamos una enumeracion de procesos dentro del sistema, en los que encontramos que se esta corriendo **firefox**, esto en una maquina es raro, podria estar alojando el servicio web como administrador.
![Command Get Process Firefox](/assets/post_details/heist/heist_firefox_process.png)

### ProcDump
Nos ayudamos de la utilidad **ProcDump** para dumpear los procesos en memoria, estos procesos pueden contener informacion valiosa, descargamos la herramienta desde la suite de Microsoft.
![Procdump](/assets/post_details/heist/heist_procdump.png)

Subimos la herramienta con `upload` desde **evil-winrm**, debemos aceptar las condiciones de **eula** para poder usar la herramienta. Con la flag **-ma** indicamos el identificador del proceso y por ende queremos un dumpeo a full.
```powershell
# Aceptamos las condiciones
.\procdump64.exe -accepteula

# Hacemos uso de la herramienta .\procdump64.exe -ma id name
.\procdump64.exe -ma 6408 firefox.dmp

# Descargamos el dump
download firefox.dmp

# Buscamos las credenciales por las strings filtrando
strings firefox.dmp | grep password
```
![Procdump](/assets/post_details/heist/heist_get_password.png)

Una vez tenemos las credenciales del usuario **Administrator** accedemos al sistema a traves de **winrm**:
```bash
evil-winrm -i 10.10.10.149 -u Administrator -p '4dD!5}x/re8]FBuZ'

psexec.py 'administrator:4dD!5}x/re8]FBuZ@10.10.10.149'
```

