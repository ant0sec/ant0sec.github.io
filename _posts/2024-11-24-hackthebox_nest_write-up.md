---
layout: post
title: Hack The Box Curling Write-Up
date: 2024-11-24
categories: [Write-Ups, HackTheBox, Machine, Easy]
tags: [smb, Reporting Service, telnet, cme, RU Scanner, DataStream, Decrypt, eJPT, IntroToDante, Easy, Web]
---
![Nest Logo](/assets/post_details/nest/nest_logo.png)
# Hack The Box: Nest Write-Up
La **maquina easy** [Nest](https://app.hackthebox.com/machines/225) de [Hack The Box](https://app.hackthebox.com/) trata de una explotación a traves de la enumeración del puerto 445 - SMB, con la que deberemos de realizar scripts de desencriptación hasta tomar acceso al sistema mediante un servicio desconocido.

## Enumeration
Realizamos la enumeración de puertos y de servicios con **nmap** a la maquina **Nest**, encontramos:
- **445 - SMB**
- **4386 - Reporting Service V1.2** -> Se trata de un servicio que permite a los usuarios ejecutar queries a bases de datos utilizando el antiguo formato **HQK**. Le enumeración muestra posibles comandos que podemos ejecutar mediante una conexion via **TELNET** a este puerto.
```bash
# Enumeración de puertos
nmap -p- -sS -n --min-rate 5000 --open 10.10.10.178 -oN tcp_scan.txt
# Enumeración de puertos y servicios 445, 4386
nmap -p445,4386 -Pn --min-rate 5000 -n -sCV --open 10.10.10.178 -oN full_scan.txt
```
![NMAP](/assets/post_details/nest/nest_nmap.png)

### 445 - SMB
Empezamos con una enumeración **smb**, al parecer por el usuario **guest** sin credencial nos permite mostrar los **shares**, por sesión nula no los muestra. También enumeramos el nombre de la maquina **HTB-NEST**, el sistema operativo **Windows 7 / Server 2008 R2 Build**.
![CME Guest](/assets/post_details/nest/nest_cme_guest.png)

Nos conectamos mediante **smbclient** al share **Data** y nos descargamos el contenido, en el share **Users** nos muestra todos los posibles usuarios existentes del sistema, aunque no tenemos acceso a ninguno, por lo que habra que buscar credenciales:
```bash
prompt off
recurse on
mget *

smbclient \\\\10.10.10.178\\Data -U 'guest'
```
![Tree](/assets/post_details/nest/nest_show_files.png)

Nos descargamos los directorios y archivos que tenemos permisos de lectura, en ellos encontramos una alerta de mantenimiento `Maintenance Alerts.txt`, en la que no se muestra nada interesante, aunque en la otra nota de bienvenida `Welcome Email.txt` se encuentran credenciales para el usuario **TempUser**:
![Notes](/assets/post_details/nest/nest_show_notes.png)

Si nos conectamos con el usuario **TempUser** por **smb**, tenemos acceso de lectura al share **Secure$**, entramos y nos descargamos los archivos.
![CME TempUser](/assets/post_details/nest/nest_cme_tempuser.png)

En uno de estos archivos, `IT/Configs/RU Scanner/RU_config.xml` encontramos posibles credenciales del usuario c.smith, en este archivo de configuración.
![RU Config Creeds](/assets/post_details/nest/nest_ru_config_creeds.png)

En las configuraciones de **NotepadPlusPlus**, `IT/Configs/NotepadPlusPlus/config.xml`, encontramos una ruta la cual no hemos podido listar anteriormente:
![Carl Path Share](/assets/post_details/nest/nest_path_carl_share.png)

Intentamos acceder a el directorio `IT/Carl` con el usuario **TempUser**, accedemos con exito y nos descargamos el contenido.
![Carl smbclient](/assets/post_details/nest/nest_path_carl_smbclient.png)

En el contenido descargado existe el proyecto **RU Scanner**, el cual hace referencia a las credenciales anteriormente encontradas. En la ruta `IT/VB Projects/WIP/RU/RUScanner` existe un archivo de código **Visual Basic** el cual contiene una función de desencriptar un string:
![Utils Decrypt String](/assets/post_details/nest/nest_utils_decrypt_string.png)

Mediante el siguiente exploit [CrackPasswordString](https://dotnetfiddle.net/bjoBP6) construido para desencriptar el string encontrado anteriormente, nos permite sacar lo que parece una contraseña:
![Utils Script](/assets/post_details/nest/nest_utils_script.png)

Una vez tenemos la credencial nos conectamos con **c.smith** a traves de **smb** y nos volvemos a descargar el contenido, en el que encontramos la flag de **user.txt** y el directorio **HQK Reporting**, el cual contiene el archivo **Debug Mode Password.txt**, el cual no contiene nada, y el archivo de **HQK_Config_Backup.xml** el cual es un archivo de configuración de **HQK** el cual contiene el puerto y el directorio de ejecución del programa.
![Get Flag and Notes](/assets/post_details/nest/nest_get_flag_notes.png)

Vamos a intentar toda la posbile información del archivo **Debug Mode Password.txt**, ya que es un archivo muy sospechoso, por lo que nos conectamos a traves de **smb** y nos metemos al directorio **HQK Reporting**, `cd "HQK Reporting"` y realizamos el comando **allinfo** al archivo, el cual muestra toda las propiedades del archivo, entre ellas muestra varios **stream** de datos en los que existen información. Existe una manera de obtener esta **Data Stream**.
![Allinfo File](/assets/post_details/nest/nest_allinfo_note.png)

Con el comando `get "DEBUGM~1.TXT:Password:$DATA"` descargamos el data stream, para ello necesitamos el **altname** y el stream con datos en cuestion:
```shell
smb: \C.Smith\HQK Reporting\> get "DEBUGM~1.TXT:Password:$DATA"
getting file \C.Smith\HQK Reporting\DEBUGM~1.TXT:Password:$DATA of size 15 as DEBUGM~1.TXT:Password:$DATA (0.1 KiloBytes/sec) (average 0.1 KiloBytes/sec)
```

El cual contiene un string la cual nos servira para loguearnos con el comando **debug** en el servicio que corre en el puerto 4386:
![Data Stream](/assets/post_details/nest/nest_get_data_stream.png)

Nos conectamos mediante `telnet 10.10.10.178 4386`, introducimos la contraseña y hacemos uso del comando **LIST** para listar las posibles queries a ejecutar.
![Telnet Service](/assets/post_details/nest/nest_telnet_service.png)

Mediante el comando **SETDIR** nos movemos entre directorios, por lo que nos movemos al siguiente directorio, en el cual encontramos la configuración de **Ldap**.
```bash
SETDIR C:\Program Files\HQK\LDAP
```
![Script Service](/assets/post_details/nest/nest_get_script_service.png)

Con el comando **SHOWQUERY** listamos el archivo, en este caso vemos lo que puede ser una posible contraseña para el usuario **Administrator** aunque esta esta encriptada con algun tipo de metodo.
![Show Data Service](/assets/post_details/nest/nest_show_data_service.png)

El ejecutable anterior **HqkLdap.exe**, si analizamos el codigo de este podemos crear un decodificador el descifre la contraseña, el cual es el siguiente [codigo](https://onecompiler.com/python/42yb8ky93):
```python
#!/usr/bin/env python3
import binascii
import base64
from Crypto.Cipher import AES
from Crypto.Protocol import KDF
from Crypto.Util.Padding import unpad
    
password = "667912"
salt = "1313Rf99"
ivs = b"1L1SA61493DRV53Z"
iteration_for_key = 3
key_len = 256//8
ciphertext = "yyEq0Uvvhq2uQOcWG8peLoeRQehqip/fKdeG/kjEVb4="
    
# Step 1: Create the key to be used with AES-256-CBC cipher
key = KDF.PBKDF2(password, salt, count=iteration_for_key, dkLen=key_len)
    
# Step 2: Create the cipher according to the file Utils.vb
cipher = AES.new(key, AES.MODE_CBC, iv=ivs)
    
# Step 3: Decrypt the base64 string using the created cipher.
ciphertext_decoded = base64.b64decode(ciphertext)
padded_plaintext = cipher.decrypt(ciphertext_decoded)
plaintext = unpad(padded_plaintext, cipher.block_size)
    
print(f"Password recovered: {plaintext}")
```

Introducimos la contraseña a decodificar y obtenemos la contraseña decodificada:
![Decrypt Passoword](/assets/post_details/nest/nest_script_decrypt_password.png)

A traves de **smbexec** nos conectamos con las credenciales del usuario **Administrator** y obtenemos la flag `type "C:\Users\Administrator\Desktop\root.txt"`.
```bash
impacket-smbexec -target-ip 10.10.10.178 'administrator:XtH4nkS4Pl4y1nGX'@10.10.10.178
```
![Get System](/assets/post_details/nest/nest_get_system_smbexec.png)