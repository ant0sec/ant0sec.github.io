---
layout: post
title: Hack The Box Active Write-Up
date: 2024-11-25
categories: [Write-Ups, HackTheBox, Machine, Easy]
tags: [smb, PasswordSpraying, AS-REP Roasting, Kerberoasting, LateralMovement, Windows, AD, eCPPTv3, Easy, Web]
---

![Active Logo](/assets/post_details/active/active_logo.png)
# Hack The Box: Active Write-Up
La **maquina easy** [Active](https://app.hackthebox.com/machines/148) de [Hack The Box](https://app.hackthebox.com/) trata de una enumeración por el protocolo **smb** en la que se encontraran usuarios para asi lanzar un ataque **Password Spraying**, consiguiendo acceso total a la maquina mediante el ataque **Kerberoasting**, realizaremos diferentes tecnicas de **Lateral Movement** para acceder a la maquina.

## Enumeration
Realizamos una enumeración a la maquina con nmap con la que encontramos el dominio del servidor, el cual lo ponemos en el archivo /etc/hosts con la ip.
![NMAP](/assets/post_details/active/active_nmap.png)

### 445 - SMB
Con **smbclient** intentamos logearnos con una sesion nula para ver si tenemos acceso.
![CME Null Session](/assets/post_details/active/active_smb_null_session.png)

Al ver que tenemos acceso probamos con la herramienta **smbmap** nos traemos con el parametro **-r** el unico acceso compartido que podemos leer.
![SMBMAP Share](/assets/post_details/active/active_smbmap_show_share_to_read.png)

Con la herramienta **CrackMapExec** realizamos la misma enumeración, en este caso con sesiones nulas.
![CME Null Session](/assets/post_details/active/active_cme_null_session.png)

También probamos si nos podemos conectar con sesiones nulas o con el usuario **guest**.
![RPC Null guest](/assets/post_details/active/active_rpc_null_guest.png)

Podemos ver el contenido del interior del acceso compartido, si seguimos la siguiente ruta, accedemos al fichero **Groups.xml** el cual contiene politicas sobre la contraseña de algunos usuarios, y algunas veces contiene una contraseña protegida. Este archivo nos lo podemos bajar con el parámetro **--download path**
![smbmap read share](/assets/post_details/active/active_smbmap_read_share.png)
![smbmap show grousp](/assets/post_details/active/active_smbmap_show_groups.png)
![smbmap donwload content](/assets/post_details/active/active_smbmap_donwload_content.png)

Otra manera de descargarlo es con **smbclient** y el uso de varios comandos:
```bash
# Realizamos la conexion a traves de smbclient
smbclient \\\\10.10.10.100\\Replication -U '' -N
# Nos metemos en la carpeta
cd active.htb
# Descargar el contenido de la carpeta
prompt off
recurse on
mget *
```

La carpeta **Replication** es una replica de la carpeta **SYSVOL**, la cual en maquinas antiguas de **Windows** tiene una vulneravilidad debido a una desconfiguracion de **Windows** en la que se guardan credenciales **GPP** las cuales podemos desencriptar, ya que existen herramientas como **gpp-decrypt** las cuales tienen la clave de desencriptacion de este tipo de contraseñas, ya que fue comprometida.

Con **grep** realizamos una busqueda recursiva en la que encontramos una credenciales del usuario **SVC_TGS**:
![grep password](/assets/post_details/active/active_grep_password.png)
![PW Groups](/assets/post_details/active/active_show_pw_groups.png)

La contraseña protegida la podemos desencriptar con la herramienta **gpp-decrypt**.
![GPP Decrypt](/assets/post_details/active/active_gpp_decrypt_pw.png)
## AD Explotation
### Users Enum
Realizamos una enumeracion de usuarios con la herramienta **rpcclient**, y el comando `enumdomusers`, especificamos el usuario previamente obtenido:
```bash
cat users.txt | awk -F '[' '{print $2}' | awk -F ']' '{print $1}'
```
![rpclient enumdomusers](/assets/post_details/active/active_rpcclient_enumdomusers.png)
### Password Spraying 
Realizamos un ataque de **Password Spraying** con el que no obtenemos **Password Reuse** de otros usuarios, aunque si validamos el usuario anteriormente obtenido.
![CME PS](/assets/post_details/active/active_cme_password_spraying.png)
### AS-REP ROASTING ATTACK
En este caso probamos un ataque **AS-REP ROASTING**, la herramienta **impacket-GetNPUsers** busca usuarios los cuales tienen la opcion **UF_DONT_REQUIRE_PREAUTH**, estos no requieren de una preautenticacion, por lo que tendriamos la posiblidad de obtener los hashes de los usuarios, aunque no en este caso.
```bash
impacket-GetNPUsers -no-pass -usersfile user.txt active.htb/
```
![getnpuusers](/assets/post_details/active/active_getnpuusers.png)
### KERBEROASTING ATTACK
Si nos fijamos en el fichero **Groups.xml** podemos ver el nombre de un usuario, por lo que esta contraseña será la de este, tendremos que con la herramienta **impacket-GetUserSPNs** poder conseguir el ticket (**TGS**) de Kerberos el cual se corresponde a un hash, el cual lo podemos desencriptar con la herramienta **JohnTheRipper**. 
```bash
impacket-GetUserSPNs active.htb/SVC_TGS
```
![getnpuusers](/assets/post_details/active/active_getnpuusers_show_tgs.png)
![getuserspns](/assets/post_details/active/active_impacket_getuserspns.png)
![john crack](/assets/post_details/active/active_john_crack.png)
Validacion de la contraseña valida.
![CME PS administrator](/assets/post_details/active/active_cme_password_spraying_admin.png)

### Dumping LSA Secrets
Podemos encontrar hashes de usuarios o contraseñas de usuarios antiguos.
![CME Get LSA](/assets/post_details/active/active_cme_get_lsa.png)
### Dumping NTDS
Dumpeamos los hashes de los usuarios los cuales nos pueden servir para realizar **PASS THE HASH**. 
![CME Get NTDS](/assets/post_details/active/active_cme_get_ntds.png)

### Lateral Movement
Una vez tenemos la contraseña podemos acceder al sistema con **evilwinrm** o **impacket-psexec**.
```bash
impacket-psexec active.htb /Administrator:Ticketmaster1968@10.10.10.100 cmd.exe
```
![PSEXEC attack](/assets/post_details/active/active_psexec_attack.png)
#### PASS-THE-HASH
Validamos que tenemos acceso con el **hash** del usuario **Administrator** y **impacket-psexec** realizamos **PASS THE HASH** con **NLHASH:NTHASH**
![CME Verify Pass](/assets/post_details/active/active_cme_verify_password.png)
![PSEXEC GetSystem](/assets/post_details/active/active_psexec_get_system.png)
