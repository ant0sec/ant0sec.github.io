---
layout: post
title: Hack The Box Emdee Five For Life Write-Up
date: 2024-11-17
categories: [Write-Ups, HackTheBox, Challenge, Easy]
tags: [EmdeeFiveForLife, md5, Easy, Web]
---
# Hack The Box: Emdee Five For Life Write-Up
Este reto se titula **emdee five for live** o **MD5 para toda la vida**, ya vemos que por el titulo se trata sobre el metodo de encriptación **md5**, y en la descripción del reto nos muestra si somos capaces de encriptar lo suficiente rapido.

Podemos ver que se trata de una web en donde tenemos que encriptar el siguiente **string** en **md5**, cuando lo encriptamos y lo introducimos vemos que no ocurre nada por lo que tenemos que ser más rapidos.
![WebPage](/assets/post_details/emdeefiveforlife/md5forlive_web_page.png)

Existe un manera, automatizando el proceso mediante herramientas de consola como **curl**, la cual permite capturar strings de la respuesta y enviar parametros.

Realizamos un script en bash con **curl**:
- Cogemos el primer elemento con `$1`
- Capturamos la respuesta con las cabeceras y la guardamos en una variable `curl -si $host`
- Filtramos por la cookie en las cabeceras, todo el contenido hasta hasta el `;`, luego lo pipeamos con tr y lo eleminamos `grep -oE 'Cookie: .*?;' <<< $res | tr -d ';'`
- Capturamos el string a encriptar filtrando la respuesta `grep -oE "<h3 align='center'>.*?</h3>" <<< $res`
- Cogemos el string a encriptar `echo -n ${string:19:20}`, lo encriptamos con `md5sum` y cogemos solamente la primera columna `awk '{ print $1 }')`
- Mandamos la petición autorizada con la cookie, con la flag **-sd** indicamos el hash, en este caso la variable y filtramos por la flag **HTB{}**, `curl -H "$cookie" -sd hash=$md5 $host | grep -oE 'HTB{.*?}'`

```bash
#!/usr/bin/env bash

# Coger elemento
host=$1

# Capturar cookie
res=$(curl -si $host)
cookie=$(grep -oE 'Cookie: .*?;' <<< $res | tr -d ';')

# Capturar string y convertir a md5
string=$(grep -oE "<h3 align='center'>.*?</h3>" <<< $res)
md5=$(echo -n ${string:19:20} | md5sum | awk '{ print $1 }')

# Mandar la peticion autorizada con el md5 y filtrar por la flag
curl -H "$cookie" -sd hash=$md5 $host | grep -oE 'HTB{.*?}'
```

Este seria el resultado, primero sin filtrar por la flag y luego lo filtramos.
![Script](/assets/post_details/emdeefiveforlife/md5forlive_script.png)