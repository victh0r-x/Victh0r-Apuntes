# 🗂️ Writeup HTB — Return

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Windows
> **🎯 IP:** 10.129.95.241
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración activa, lanzamos un ping para verificar conectividad con la máquina objetivo. El valor del **TTL** nos permite intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```bash
> ping -c 1 10.129.95.241
> ```
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.95.241` — **IP objetivo** a la que se lanza el ping.

Obtenemos respuesta con un TTL de **127**, lo que confirma que estamos ante una máquina **Windows**. Confirmada la conectividad, procedemos con la enumeración de puertos.

---

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

Comenzamos identificando qué puertos se encuentran abiertos sobre el rango completo de los 65535 posibles mediante un SYN Scan, que envía paquetes SYN sin completar el three-way handshake TCP — más rápido y menos ruidoso que un TCP Connect Scan completo.

> [!tip] Escaneo de puertos completo y sigiloso
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.95.241 -vvv -oN ports
> ```
> `-sS` — **SYN Scan** (half-open). Envía SYN, recibe SYN-ACK si el puerto está abierto o RST si está cerrado, sin completar nunca el handshake. Requiere privilegios de root.
> `-p-` — Escanea los **65535 puertos** posibles, desde el 1 hasta el 65535.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando los cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo** para acelerar el escaneo.
> `-n` — Sin resolución DNS — evita latencia adicional y ruido en la red.
> `-Pn` — Sin ping previo — asume el host activo aunque ICMP esté bloqueado.
> `10.129.95.241` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose** — muestra los puertos abiertos en tiempo real sin esperar al final.
> `-oN ports` — Guarda el output en formato legible en el archivo `ports`.

![](assets/Return-img-04-04-2026-2.png)

El escaneo revela múltiples puertos abiertos típicos de un **controlador de dominio Windows**: `53` (DNS), `80` (HTTP), `88` (Kerberos), `135` (RPC), `139/445` (SMB), `389/636` (LDAP/LDAPS), `464` (Kerberos password change), `593` (RPC over HTTP), `3268/3269` (Global Catalog), `5985` (WinRM) y múltiples puertos RPC dinámicos altos.

---

#### 🔬 Enumeración de versiones y servicios

Con los puertos identificados, lanzamos un segundo escaneo apuntando únicamente a los puertos descubiertos para determinar qué servicios y versiones exactas corren en cada uno, ejecutando los scripts NSE por defecto para obtener información adicional.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```bash
> nmap -sC -sV -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49674,49675,49678,49681,49697,49723 --min-rate 5000 -n -Pn -vvv 10.129.95.241 -oN version
> ```
> `-sC` — Lanza los **scripts por defecto** del motor NSE (Nmap Scripting Engine). Incluye detección de banners, enumeración básica de servicios y comprobaciones de configuraciones inseguras.
> `-sV` — **Detección de versiones** — envía probes adicionales para identificar el software exacto y su versión en cada puerto.
> `-p53,80,88,...` — Escanea **únicamente los puertos descubiertos** en el paso anterior.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — **Triple verbose**.
> `10.129.95.241` — **IP objetivo**.
> `-oN version` — Guarda el output en el archivo `version`.

![](assets/Return-img-04-04-2026.png)
![](assets/Return-img-04-04-2026-1.png)

---

#### 🌐 Fuzzing Web — Puerto 80

Con el puerto 80 activo, procedemos a realizar fuzzing de directorios y archivos para mapear la superficie web expuesta. El objetivo es descubrir rutas ocultas, paneles de administración, archivos de configuración o cualquier recurso no enlazado directamente desde la página principal. Lanzamos gobuster con una wordlist de tamaño medio y un conjunto amplio de extensiones para maximizar la cobertura:

> [!tip] Fuzzing de directorios y archivos sobre el puerto 80
> ```bash
> gobuster dir -u http://10.129.95.241 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --no-error -x php,htm,html,asp,aspx,txt,xml -t 40
> ```
> `dir` — Modo de fuerza bruta de directorios y archivos en un servidor web.
> `-u http://10.129.95.241` — URL objetivo del escaneo.
> `-w .../DirBuster-2007_directory-list-2.3-medium.txt` — Wordlist en su variante medium para una búsqueda exhaustiva sin ser excesivamente lenta.
> `--no-error` — No muestra errores en el output, manteniendo la salida limpia.
> `-x php,htm,html,asp,aspx,txt,xml` — Extensiones adaptadas a un servidor Windows IIS — se añaden `asp` y `aspx` que son las tecnologías web nativas de Windows, más relevantes que extensiones de Linux como `.py` o `.rb`.
> `-t 40` — Lanza **40 hilos en paralelo** para acelerar el escaneo.


En el apartado settings.php podemos ver lo siguiehte;![](assets/Return-img-05-04-2026.png)

Antes hemos visto que se esta usando el puerto 389 el cual envia las credenciales en texto plano, por lo que podemos ponernos en escucha por cualqueir puerto y ponerlo ahi, darle a enviar y ver que recibimos en la conexion;



---

## 💥 Explotación

---
![](assets/Return-img-05-04-2026-1.png)

Recibimos una contraseña en texto plano: 1edFg43012!!

![](assets/Return-img-05-04-2026-2.png)

## 🔼 Escalada de Privilegios

---

## 📝 Lecciones Aprendidas