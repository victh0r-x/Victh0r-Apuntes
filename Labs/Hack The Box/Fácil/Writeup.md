
> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Windows
> **🎯 IP:** 10.129.14.68
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración, lanzamos un ping para verificar conectividad con la máquina objetivo. Además del simple echo reply, aprovecharemos el valor del **TTL** para intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```python
> ping -c 1 10.129.11.19
> ```
>
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.14.68` — **IP objetivo** a la que se lanza el ping.

![](assets/Writeup-img-30-03-2026.png)

Como se puede observar en el output, obtenemos respuesta con un TTL de **63**, lo que nos indica que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración.


### 👁️ Enumeración con NMAP

#### Enumeración de puertos

Comenzamos la fase de enumeración identificando qué puertos se encuentran abiertos en la máquina objetivo. Para ello utilizamos nmap en su modalidad de SYN Scan, que nos permite realizar un escaneo rápido y sigiloso sobre el rango completo de puertos sin completar el three-way handshake.

> [!tip] Escaneo de puertos completo y sigiloso
> ```python
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.11.19 -vvv -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open). Envía paquetes SYN sin completar el three-way handshake. Sigiloso y rápido, requiere root.
> `-p-` — Escanea los **65535 puertos** posibles.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `IP_MAQUINA` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN ports` — Guarda el output en formato normal en un archivo llamado `ports`.

![](assets/Writeup-img-30-03-2026-1.png)

---
#### Enumeración de versiones y servicios

Con los puertos abiertos ya identificados, el siguiente paso es determinar qué servicios y versiones están corriendo en cada uno de ellos. Esto nos permitirá identificar tecnologías concretas y posibles vectores de ataque. Para ello lanzamos nmap con sus scripts por defecto y detección de versiones, apuntando únicamente a los puertos descubiertos en el paso anterior.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```python
> nmap -sC -sV -p22,80 --min-rate 5000 -n -Pn -vvv 10.129.11.19 -oN version
> ```
>
> `-sC` — Lanza los **scripts por defecto** del motor NSE de nmap.
> `-sV` — **Detección de versiones** de los servicios que corren en cada puerto.
> `-pPUERTOS,SEPARADOS,POR,COMAS` — Escanea **únicamente los puertos encontrados** en el paso anterior.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `IP_VICTIMA` — **IP objetivo** del escaneo.
> `-oN version` — Guarda el output en formato normal en un archivo llamado `version`.

![](assets/Writeup-img-30-03-2026-2.png)

Visitamos el servicio web  y vemis lo siguiebnte:

![](assets/Writeup-img-30-03-2026-3.png)

Vemos que hay un usuario llamado jkr y vemos que la pagina esta creada con vi, que es un editor de texto.

En el archivo version, tambien vemos que hay un directorio /writeup/, accedenos:

![](assets/Writeup-img-30-03-2026-4.png)

Viendo el codigo fuente de la web podemos ver que esta usando el cms de cms made simple, donde ademas vemos que se ha quedado abandonado el proyecto en 2019

![](assets/Writeup-img-30-03-2026-5.png)

## 💥 Explotación
---
Teniendo en cuenta que la web se ha descontinuado en 2019 podemos decir que la version de cms made simplw debe ser de 2019, por lo que si buscamos en google cms made simple cve podemis buscar alguna de 2019 y probar. encontramos la cve-2019-9053 viendo que adecta a las versiones menores de la 2.2.10, asi que hacemos una busqueda en searchsploit de esta verison:

![](assets/Writeup-img-31-03-2026.png)

Es vulnerable a una sql injection, asi que descargamos el exploit con searchsploit -m php/webapps/46635.py

lo ejecutamos con python2 con el coando: python2 exploit.py -u http://writeup.htb/writeup/ y obtenemos lo siguiente:

![](assets/Writeup-img-31-03-2026-1.png)

Un usuario que ya conociamos, jkr, y una contraseña encriptada. 
Ahora uso el comando hash-identifier para ver de que se trata: hash-identifier 62def4866937f08cc13bab43bb14e6f7  
![](assets/Writeup-img-31-03-2026-2.png)

Lo desencryptamos y obtengo 5a599ef579066807raykayjay9
La contraseña era raykayjay9, nos conectamos con el comando ssh jkr@10.129.11.19 y la contraseña

![](assets/Writeup-img-31-03-2026-3.png)

Estamos dentro como usuario y pueod leer la flag.



## 🔼 Escalada de Privilegios
---
Para escalar privilegios uso la herramienta pspy64 que la descargo de github en mi maquina atacante y la envio a la maquina victima usando uns ervidor web de python por el puerto 8080. Una vez en la maquina victima, lo ejecuto y me conecto por ssh con otra shell para ver que ocurre, y vemos lo siguiente:

![](assets/Writeup-img-31-03-2026-5.png)

Vemos que root  usa el comando sh con el path cuyo primer directorio es usr local bin, al cual tenemos acceso de escritura por ser del grupo staff, cosa que comprobamos al ganar acceso a la maquina
Para explotar esto, lo que hare es crear un archiovo llamado run-parts dentro del directori /usr local bin, para que al hacer ssh con el usuario jkr, root lo ejecute.
En este caso el archivo contendra un codigo que convertira el binario /bin/bash en SUID. Para ello creo el archivo con el siguiente codigo: #!/bin/bash chmod 4000 /bin/bash lo guardamos y luego chmod 755 run-parts y nos conectamos otra vez por ssh.

Al hacer esto, ya podemos comprobar que la bash tiene permisos SUID con el comando ls -la /bin/bash

![](assets/Writeup-img-31-03-2026-7.png)

Ahora ejecuto bash -p y soy root:

![](assets/Writeup-img-31-03-2026-8.png)







## 📝 Lecciones Aprendidas