---
layout: post
title: Hack The Box Blue Write-Up
date: 2024-11-10
categories: [Write-Ups, HackTheBox, Machine, Easy]
tags: [Blue, EternalBlue, MS17-010, Metasploit, Easy, Linux]
---
![Blue Logo](/assets/post_details/blue/blue_logo.png)
# Hack The Box: Blue Write-Up
La **maquina easy** [Blue](https://app.hackthebox.com/machines/51) de [Hack The Box](https://app.hackthebox.com/) trata de la explotación de la famosa vulnerabilidad **Eternal Blue MS17-010**, la cual fue muy temida durante muchos años. 
Esta maquina viene muy bien para principiantes y para coger practica con la herramienta **metasploit**. 

## Enumeration
Realizamos una enumeración de puertos y de servicios en los que encontramos varios puertos abiertos a destacar el **445** el cual nos muestra información a traves de los scripts de nmap, el sistema es un **Windows 7 Professional 6.1**, esta versión es vulnerable a la famosa vulnerabilidad **Eternal Blue**. 
![NMAP](/assets/post_details/blue/blue_nmap.png)

## EternalBlue MS17-010
**EternalBlue**es un exploit de seguridad desarrollado originalmente por la Agencia de Seguridad Nacional de Estados Unidos (NSA) que aprovechaba una vulnerabilidad en el protocolo **SMB** (Server Message Block) de Microsoft Windows. Fue filtrado en 2017 por un grupo de hackers conocido como The Shadow Brokers y se convirtió en la base de varios ataques cibernéticos masivos, incluyendo el ransomware WannaCry y NotPetya.

**EternalBlue** permitía a los atacantes ejecutar código malicioso de manera remota y sin autenticación en sistemas Windows vulnerables, lo que les daba acceso total al equipo afectado. Esta vulnerabilidad fue particularmente peligrosa porque se propagaba rápidamente de un equipo a otro dentro de una red, aumentando su alcance y daño. Aunque Microsoft lanzó un parche para esta vulnerabilidad antes de su filtración pública, muchos sistemas no actualizados siguieron siendo vulnerables, lo que permitió que **EternalBlue** tuviera un impacto significativo en la ciberseguridad a nivel mundial.

## System Access
En este caso vamos a utilizar la herramienta **metasploit**, esta herramienta tiene incorparado una cantidad ingente de exploits entre ellos los de **EternalBlue**:
![Eternal Blue](/assets/post_details/blue/blue_eternal_blue.png)

Configuramos el exploit asignando la ip atacante y la victima, y corremos el script. Si es exitoso obtenemos una sesión como el usuario ocn mayor privlegio.
![Exploit Eternal Blue](/assets/post_details/blue/blue_exploit_eternal_blue.png)