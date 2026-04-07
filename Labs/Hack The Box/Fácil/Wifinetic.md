# 🗂️ Writeup HTB — Wifinetic

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.129.229.90
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración activa, lanzamos un ping para verificar conectividad con la máquina objetivo. El valor del **TTL** nos permite intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```bash
> ping -c 1 10.129.229.90
> ```
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.229.90` — **IP objetivo** a la que se lanza el ping.

![](assets/Wifinetic-img-27-03-2026.png)

Obtenemos respuesta con un TTL de **63**, lo que confirma que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración de puertos.

---

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

Comenzamos identificando qué puertos se encuentran abiertos sobre el rango completo de los 65535 posibles mediante un SYN Scan, que envía paquetes SYN sin completar el three-way handshake TCP — más rápido y menos ruidoso que un TCP Connect Scan completo.

> [!tip] Escaneo de puertos completo y sigiloso
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.229.90 -vvv -oN ports
> ```
> `-sS` — **SYN Scan** (half-open). Envía SYN, recibe SYN-ACK si el puerto está abierto o RST si está cerrado, sin completar nunca el handshake. Requiere privilegios de root.
> `-p-` — Escanea los **65535 puertos** posibles, desde el 1 hasta el 65535.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando los cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo** para acelerar el escaneo.
> `-n` — Sin resolución DNS — evita latencia adicional y ruido en la red.
> `-Pn` — Sin ping previo — asume el host activo aunque ICMP esté bloqueado.
> `10.129.229.90` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose** — muestra los puertos abiertos en tiempo real sin esperar al final.
> `-oN ports` — Guarda el output en formato legible en el archivo `ports`.

![](assets/Wifinetic-img-27-03-2026-1.png)

El escaneo revela **tres puertos abiertos**: `21` (FTP), `22` (SSH) y `53` (DNS).

---

#### 🔬 Enumeración de versiones y servicios

Con los puertos identificados, lanzamos un segundo escaneo apuntando únicamente a los puertos `21`, `22` y `53` para determinar qué servicios y versiones exactas corren en cada uno, ejecutando los scripts NSE por defecto para obtener información adicional.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```bash
> nmap -sC -sV -p21,22,53 --min-rate 5000 -n -Pn -vvv 10.129.229.90 -oN version
> ```
> `-sC` — Lanza los **scripts por defecto** del motor NSE. Incluye detección de banners y comprobaciones de configuraciones inseguras — en este caso detectará si el FTP permite login anónimo.
> `-sV` — **Detección de versiones** — identifica el software exacto en cada puerto.
> `-p21,22,53` — Escanea **únicamente los puertos 21, 22 y 53** descubiertos anteriormente.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — **Triple verbose**.
> `10.129.229.90` — **IP objetivo**.
> `-oN version` — Guarda el output en el archivo `version`.

![](assets/Wifinetic-img-27-03-2026-8.png)

El resultado más relevante es que el **servicio FTP permite autenticación anónima** — un hallazgo crítico que explotaremos a continuación.

---

### 📂 Enumeración FTP — Acceso anónimo

El login anónimo en FTP permite conectarse sin credenciales usando el usuario `anonymous` con cualquier contraseña o sin ella. Es uno de los vectores de información más frecuentes en entornos mal configurados — permite descargar archivos internos que pueden contener credenciales, configuraciones o datos sensibles.

> [!tip] Conectar al FTP con autenticación anónima y descargar todos los archivos
> ```bash
> ftp 10.129.229.90
> # Usuario: anonymous
> # Contraseña: (Enter — vacía)
>
> # Una vez dentro, descargar todos los archivos recursivamente
> mget *
> ```
> `ftp 10.129.229.90` — Conecta al servicio FTP de la máquina objetivo.
> `anonymous` — Usuario estándar para acceso anónimo en FTP — no requiere contraseña válida.
> `mget *` — Descarga todos los archivos del directorio actual al directorio local.

![](assets/Wifinetic-img-27-03-2026-3.png)

Descargamos todos los archivos disponibles y procedemos a analizarlos uno a uno.

---

### 🔎 Análisis de archivos descargados

**ProjectOpenWRT.pdf** — Contiene información sobre el dominio de la organización. Lo añadimos a `/etc/hosts` para que resuelva correctamente el virtual hosting:

![](assets/Wifinetic-img-27-03-2026-4.png)

**Metadatos del PDF** — Analizando los metadatos del archivo con `exiftool` encontramos un nombre de usuario que corresponde a un posible administrador del sistema:

![](assets/Wifinetic-img-27-03-2026-6.png)

**EmployeeWellness.pdf** — Otro documento que revela un segundo nombre de usuario: `Samantha Wood`:

![](assets/Wifinetic-img-27-03-2026-7.png)

**backup-OpenWrt-2023-07-26.tar** — El archivo más valioso. Se trata de un backup completo de la configuración de un router OpenWrt. Para extraerlo:

> [!tip] Descomprimir el backup de OpenWrt
> ```bash
> tar -xvf backup-OpenWrt-2023-07-26.tar -C /home/vic/Escritorio/HTB/Wifinetic
> ```
> `tar -xvf` — Extrae (`x`) el archivo tar mostrando los archivos procesados (`v`) especificando el archivo fuente (`f`).
> `-C /ruta/destino` — Directorio donde se extraerán los archivos.

El backup contiene el directorio `/etc` completo del router con múltiples archivos de configuración sensibles:

**`/etc/passwd` y `/etc/group`** — Revelan los usuarios del sistema. Destaca la existencia del usuario `netadmin`:

![](assets/Wifinetic-img-27-03-2026-12.png)

**`/etc/config/rpcd`** — Contiene una referencia a credenciales de root en el formato `root: $p$root`:

![](assets/Wifinetic-img-27-03-2026-10.png)

**`/etc/config/wireless`** — El hallazgo más crítico. Contiene la configuración WiFi del router incluyendo la contraseña de la red en texto claro: **`VeRyUniUqWiFIPasswrd1!`**:

![](assets/Wifinetic-img-27-03-2026-11.png)

**`/etc/opkg/keys/`** — Contiene una clave de firma de paquetes: `RWRNAX5vHtXWFmt+n5di7XX8rTu0w+c8X7Ihv4oCyD6tzsUwmH0A6kO0`. Aunque no es directamente explotable, confirma que el backup es genuino y completo.

![](assets/Wifinetic-img-27-03-2026-13.png)

---

## 💥 Explotación

### 🔑 Password Reuse — Acceso SSH como netadmin

La contraseña WiFi encontrada en el backup (`VeRyUniUqWiFIPasswrd1!`) puede ser reutilizada en otros servicios — práctica extremadamente común en entornos reales donde los administradores reciclan credenciales. Probamos con el usuario `netadmin` identificado en el `/etc/passwd` del backup:

> [!tip] Acceso SSH con las credenciales obtenidas del backup
> ```bash
> ssh netadmin@10.129.229.90
> # Contraseña: VeRyUniUqWiFIPasswrd1!
> ```

![](assets/Wifinetic-img-30-03-2026.png)

El acceso tiene éxito — la contraseña WiFi del router OpenWrt fue reutilizada como contraseña SSH del usuario `netadmin` en el sistema principal.

---

## 🔼 Escalada de Privilegios

### 📡 Enumeración de interfaces WiFi

Una vez dentro del sistema como `netadmin`, enumeramos las interfaces de red disponibles. En una máquina con componente WiFi como Wifinetic, esta información es fundamental para entender la superficie de ataque:

> [!tip] Ver todas las interfaces de red del sistema
> ```bash
> ip a
> ```

![](assets/Wifinetic-img-30-03-2026-1.png)

Vemos múltiples interfaces. Para obtener información detallada específicamente sobre las interfaces WiFi y su modo de operación usamos `iwconfig`:

> [!tip] Ver configuración y modo de las interfaces WiFi
> ```bash
> iwconfig
> ```

![](assets/Wifinetic-img-30-03-2026-2.png)

El output revela que `wlan0` está configurada en modo **Master** — lo que indica que es el **Access Point (AP)**. Las interfaces en modo Master son las que actúan como punto de acceso al que otros dispositivos se conectan. También podemos obtener esta información con `iw dev`, que además muestra el **BSSID** (la dirección MAC del AP): `02:00:00:00:00:00`.

---

### ⚡ Ataque WPS PIN con Reaver

Antes de proceder al ataque, verificamos qué capabilities especiales tienen los binarios del sistema — una forma de escalar privilegios sin necesitar sudo:

> [!tip] Buscar binarios con capabilities especiales asignadas
> ```bash
> getcap -r / 2>/dev/null
> ```
> `getcap -r /` — Busca recursivamente desde la raíz cualquier binario que tenga Linux capabilities asignadas.
> `2>/dev/null` — Silencia los errores de permisos al intentar acceder a directorios restringidos.

El output revela que `reaver` tiene la capability `cap_net_raw` asignada — lo que le permite enviar y recibir paquetes de red raw sin necesitar ser root. Esto es exactamente lo que necesitamos para el ataque WPS.

> [!info] **Reaver**
> Herramienta de ataque contra el protocolo WPS (WiFi Protected Setup). WPS permite a dispositivos conectarse a una red WiFi usando un PIN de 8 dígitos en lugar de la contraseña completa. El diseño del protocolo tiene un fallo fundamental: el PIN se verifica en dos mitades de 4 dígitos, lo que reduce el espacio de búsqueda de 100 millones de combinaciones a solo 11.000. Reaver explota este fallo haciendo fuerza bruta del PIN WPS, y al encontrarlo obtiene la contraseña PSK completa de la red.

> [!info] **mon0**
> Interfaz de red en **modo monitor** — permite capturar todos los paquetes WiFi del entorno sin estar asociado a ninguna red. Es el equivalente al modo promiscuo en redes cableadas. Reaver necesita una interfaz en modo monitor para enviar y capturar los paquetes del protocolo WPS durante el ataque.

Con el BSSID del AP (`02:00:00:00:00:00`) obtenido de `iw dev` y la interfaz `mon0` disponible en modo monitor, lanzamos el ataque:

> [!tip] Ataque WPS PIN con Reaver contra el Access Point
> ```bash
> reaver -i mon0 -b 02:00:00:00:00:00 -vv -c 1
> ```
> `-i mon0` — Interfaz en modo monitor a usar para el ataque — captura y envía los paquetes WPS.
> `-b 02:00:00:00:00:00` — **BSSID** del Access Point objetivo — la dirección MAC de `wlan0` obtenida con `iw dev`.
> `-vv` — **Double verbose** — muestra el progreso del ataque pin a pin, incluyendo cada mensaje WPS intercambiado.
> `-c 1` — Canal WiFi donde opera el AP — obtenido previamente de `iw dev` (channel 1).

![](assets/Wifinetic-img-30-03-2026-3.png)

Reaver encuentra el PIN WPS y devuelve la contraseña PSK completa de la red: **`WhatIsRealAnDWhAtIsNot51121!`**

---

### 🏆 Acceso root por reutilización de contraseña

De la misma forma que la contraseña WiFi del backup fue reutilizada para SSH, esta nueva contraseña obtenida del ataque WPS se reutiliza para escalar a root:

> [!tip] Escalar a root usando la contraseña obtenida del ataque WPS
> ```bash
> su root
> # Contraseña: WhatIsRealAnDWhAtIsNot51121!
> ```

![](assets/Wifinetic-img-30-03-2026-4.png)

Somos **root**. La flag de root se encuentra en `/root/root.txt`.

---

## 📝 Lecciones Aprendidas

**El FTP anónimo es una mina de información.** Un servicio FTP con autenticación anónima habilitado en un entorno de producción es un error grave — cualquier archivo expuesto en él es accesible públicamente. En este caso, un backup completo de la configuración de un router fue suficiente para comprometer toda la máquina.

**Los backups de configuración son altamente sensibles.** El archivo `backup-OpenWrt-2023-07-26.tar` contenía contraseñas WiFi en texto claro, usuarios del sistema y claves de firma. Los backups de dispositivos de red nunca deben almacenarse en ubicaciones accesibles sin cifrar.

**La reutilización de contraseñas fue el vector en ambas escaladas.** La contraseña WiFi del router se reutilizó para SSH, y la contraseña PSK obtenida del ataque WPS se reutilizó para root. Este patrón — password reuse — es uno de los vectores más frecuentes en pentesting real y siempre debe probarse al obtener cualquier credencial nueva.

**WPS es un protocolo fundamentalmente inseguro.** El fallo de diseño que permite que Reaver reduzca 100 millones de combinaciones a 11.000 no es un bug parcheable — es inherente al diseño del protocolo. WPS debe estar deshabilitado en cualquier red que requiera un mínimo de seguridad.