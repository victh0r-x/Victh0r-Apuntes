> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Windows
> **🎯 IP:** 10.129.14.91
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración, lanzamos un ping para verificar conectividad con la máquina objetivo. Además del simple echo reply, aprovecharemos el valor del **TTL** para intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```python
> ping -c 1 10.129.14.91
> ```
>
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.14.91` — **IP objetivo** a la que se lanza el ping.

![](assets/A_Plantilla%201-img-05-04-2026.png)

Como se puede observar en el output, obtenemos respuesta con un TTL de **63**, lo que nos indica que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración.

### 👁️ Enumeración con NMAP

#### Enumeración de puertos

Comenzamos la fase de enumeración identificando qué puertos se encuentran abiertos en la máquina objetivo. Para ello utilizamos nmap en su modalidad de SYN Scan, que nos permite realizar un escaneo rápido y sigiloso sobre el rango completo de puertos sin completar el three-way handshake.

> [!tip] Escaneo de puertos completo y sigiloso
> ```python
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.14.91 -vvv -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open). Envía paquetes SYN sin completar el three-way handshake. Sigiloso y rápido, requiere root.
> `-p-` — Escanea los **65535 puertos** posibles.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `10.129.14.91` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN ports` — Guarda el output en formato normal en un archivo llamado `ports`.

![](assets/Kobold-img-05-04-2026.png)

---

#### Enumeración de versiones y servicios

Con los puertos abiertos ya identificados, el siguiente paso es determinar qué servicios y versiones están corriendo en cada uno de ellos. Esto nos permitirá identificar tecnologías concretas y posibles vectores de ataque. Para ello lanzamos nmap con sus scripts por defecto y detección de versiones, apuntando únicamente a los puertos descubiertos en el paso anterior.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```python
> nmap -sC -sV -p22,80,443,3552 --min-rate 5000 -n -Pn -vvv 10.129.14.91 -oN version
> ```
>
> `-sC` — Lanza los **scripts por defecto** del motor NSE de nmap.
> `-sV` — **Detección de versiones** de los servicios que corren en cada puerto.
> `-p22,80,443,3552` — Escanea **únicamente los puertos encontrados** en el paso anterior.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `10.129.14.91` — **IP objetivo** del escaneo.
> `-oN version` — Guarda el output en formato normal en un archivo llamado `version`.

![](assets/Kobold-img-05-04-2026-1.png)
![](assets/Kobold-img-05-04-2026-2.png)
![](assets/Kobold-img-05-04-2026-3.png)
![](assets/Kobold-img-05-04-2026-4.png)

---
Vemos que está intentando hacerse una redireccion al dominio kobold.htb pero no consigue resolverlol Esto es potque se esta aplicando virutal hosting y para solucionarlo debemos añadir la ip y el dominio al archivo etc hosts:

10.129.14.91    kobold.htb

Ahora ya podemos acceder al sitio web:

![](assets/Kobold-img-05-04-2026-5.png)

No parece que haya nada interesante a simple vista, excepto que vemos un correo de contacto: admin@kobold.htb lo que nos sugiere un usuario valido. Ahora vamos a aplicar fuzzing para buscar directorios y archivos 
#### 🌐 Fuzzing Web

Con el puerto 80 activo, procedemos a realizar fuzzing de directorios y archivos para mapear la superficie web expuesta. El objetivo es descubrir rutas ocultas, paneles de administración, archivos de configuración o cualquier recurso no enlazado directamente desde la página principal.

> [!tip] Fuzzing de directorios y archivos sobre el puerto 80
> ```bash
> gobuster dir -u https://kobold.htb -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --no-error -x php,htm,html,csv,txt,xml --add-slash -t 40 -xl 178 -k -o fuzzing 
> ```
>
> `dir` — Modo de fuerza bruta de directorios y archivos en un servidor web.
> `-u http://10.129.14.91` — URL objetivo del escaneo.
> `-w .../DirBuster-2007_directory-list-2.3-medium.txt` — Wordlist en su variante medium para una búsqueda exhaustiva sin ser excesivamente lenta.
> `--no-error` — No muestra errores en el output, manteniendo la salida limpia.
> `-x php,htm,html,csv,txt,xml` — Añade extensiones de archivo a cada entrada de la wordlist.
> `--add-slash` — Añade una barra final a cada petición.
> `-t 40` — Lanza **40 hilos en paralelo** para acelerar el escaneo.

Tras ejecutarse el fuzzing no encuentro nada pero pongo el comando porque es especial para estea caso y merece la pena estudiarlo. 
Siguiendo con la enumeracion, vemos que en el puerto 3552 hay otro servicio web, asi que accedemos:

![](assets/Kobold-img-05-04-2026-6.png)

Vemos que se trata del servicio de arcane (claude define brevemente que es esto) y qye ademas vemos que se trata de la verison 1.13


---

## 💥 Explotación

---

## 🔼 Escalada de Privilegios

---

## 📝 Lecciones Aprendidas