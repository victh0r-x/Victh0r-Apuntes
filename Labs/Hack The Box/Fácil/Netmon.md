# 🏴 Writeup — Netmon

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Windows
> **🎯 IP:** 10.129.230.176
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box
> **📅 Fecha:** 04-03-2026
> **✅ Pwned:** 🟢 Sí

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración, lanzamos un ping para verificar conectividad con la máquina objetivo. Además del simple echo reply, aprovecharemos el valor del **TTL** para intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```python
> ping -c 1 10.129.230.176
> ```
>
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.230.176` — **IP objetivo** a la que se lanza el ping.

![](/assets/Netmon-img-04-03-2026.png)

Como se puede observar en el output, obtenemos respuesta con un TTL de **127**, lo que nos indica que estamos ante una máquina **Windows**. Confirmada la conectividad, procedemos con la enumeración.

### 👁️ Enumeración con NMAP

#### Enumeración de puertos

Comenzamos la fase de enumeración identificando qué puertos se encuentran abiertos en la máquina objetivo. Para ello utilizamos nmap en su modalidad de SYN Scan, que nos permite realizar un escaneo rápido y sigiloso sobre el rango completo de puertos sin completar el three-way handshake.

> [!tip] Escaneo de puertos completo y sigiloso
> ```python
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.230.176 -vvv -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open). Envía paquetes SYN sin completar el three-way handshake. Sigiloso y rápido, requiere root.
> `-p-` — Escanea los **65535 puertos** posibles.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `10.129.230.176` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN ports` — Guarda el output en formato normal en un archivo llamado `ports`.

![](/assets/Netmon-img-04-03-2026-1.png)

El escaneo revela una superficie de ataque amplia y variada. Destacamos los puertos más relevantes de cara a la explotación:

- **Puerto 21 (FTP)** — Inmediatamente sospechoso, posible login anónimo habilitado.
- **Puerto 80 (HTTP)** — Servicio web activo, pendiente de inspección.
- **Puerto 139/445 (SMB)** — Protocolos de red Windows, útiles en fases posteriores.
- **Puerto 5985 (WinRM)** — Permite ejecución remota de comandos si obtenemos credenciales válidas.

---

#### Enumeración de versiones y servicios

Con los puertos identificados, lanzamos nmap con detección de versiones y scripts por defecto. Esta fase es crítica: conocer la versión exacta de cada servicio es lo que nos permitirá buscar vulnerabilidades concretas y construir el vector de ataque.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```python
> nmap -sC -sV -p21,80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669 --min-rate 5000 -n -Pn -vvv 10.129.230.176 -oN version
> ```
>
> `-sC` — Lanza los **scripts por defecto** del motor NSE de nmap.
> `-sV` — **Detección de versiones** de los servicios que corren en cada puerto.
> `-p21,80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669` — Escanea **únicamente los puertos encontrados** en el paso anterior.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `10.129.230.176` — **IP objetivo** del escaneo.
> `-oN version` — Guarda el output en formato normal en un archivo llamado `version`.

![](/assets/Netmon-img-04-03-2026-2.png)
![](/assets/Netmon-img-04-03-2026-4.png)

Dos hallazgos inmediatos que marcan el rumbo de la explotación. Primero, el script `ftp-anon` de nmap confirma que el servicio FTP tiene el **login anónimo habilitado**, lo que nos da acceso al sistema de archivos del servidor sin necesidad de credenciales. Segundo, el servicio web en el puerto 80 está activo y responde, pendiente de inspección manual.

---

#### Enumeración de posibles vulnerabilidades

Con los servicios y versiones identificados, lanzamos los scripts de la categoría `vuln` de nmap para detectar vulnerabilidades conocidas de forma automática.

> [!tip] Detección automática de vulnerabilidades con scripts NSE
> ```python
> nmap --script vuln -p21,80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669 --min-rate 5000 -n -Pn -vvv -oN vuln 10.129.230.176
> ```
>
> `--script vuln` — Lanza la categoría de scripts **vuln** del motor NSE, que comprueba vulnerabilidades conocidas en los servicios detectados.
> `-p21,80,...` — Apunta únicamente a los **puertos ya identificados** anteriormente.
> `10.129.230.176` — **IP objetivo** del escaneo.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN vuln` — Guarda el output en formato normal en un archivo llamado `vuln`.

![](/assets/Netmon-img-04-03-2026-5.png)

El escaneo no devuelve resultados relevantes, probablemente debido a algún mecanismo de filtrado o firewall activo que está bloqueando o descartando las sondas de los scripts. Esto es habitual en entornos Windows con Defender activo o reglas de firewall restrictivas. No nos bloqueamos aquí y continuamos explorando los vectores identificados manualmente.

---

#### Fuzzing web

Con el puerto **80** activo, procedemos a realizar fuzzing de directorios y archivos para mapear la superficie web expuesta. El objetivo es descubrir rutas ocultas, paneles de administración, archivos de configuración o cualquier recurso no enlazado directamente desde la página principal. Lanzamos gobuster con una wordlist de tamaño medio y un conjunto amplio de extensiones para maximizar la cobertura.

> [!tip] Fuzzing de directorios y archivos sobre puerto 80
> ```python
> gobuster dir -u http://10.129.230.176 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --ne -x php,htm,html,csv,txt,xml --add-slash -t 40 -xl 0
> ```
>
> `dir` — Modo de **fuerza bruta de directorios y archivos** en un servidor web.
> `-u http://10.129.230.176` — **URL objetivo** del escaneo.
> `-w ...DirBuster-2007_directory-list-2.3-medium.txt` — **Wordlist** en su variante medium para una búsqueda exhaustiva.
> `--ne` — **No muestra errores** en el output, manteniendo la salida limpia.
> `-x php,htm,html,csv,txt,xml` — Añade **extensiones de archivo** a cada entrada de la wordlist, ampliando la superficie de búsqueda.
> `--add-slash` — Añade una **barra final** a cada petición, útil para forzar la detección de directorios.
> `-t 40` — Lanza **40 hilos** en paralelo para acelerar el escaneo.
> `-xl 0` — **Excluye respuestas con longitud 0**, filtrando resultados vacíos sin contenido relevante.

---

## 💥 Explotación

### 📂 Acceso FTP anónimo y obtención de la flag de usuario

El primer vector a explotar es el más inmediato: el **login anónimo FTP**. Este fallo de configuración permite a cualquier usuario conectarse al servidor FTP sin necesidad de credenciales, utilizando simplemente `anonymous` como nombre de usuario. En entornos Windows, el servidor FTP suele tener acceso al sistema de archivos del servidor, lo que puede exponer información crítica.

> [!info] ¿Por qué es peligroso el login anónimo FTP?
> Cuando un servidor FTP tiene el acceso anónimo habilitado, cualquier atacante puede conectarse y navegar por los directorios accesibles sin autenticación. En sistemas Windows, esto puede exponer directorios de usuarios, archivos de configuración de aplicaciones instaladas y en el peor de los casos, credenciales almacenadas en texto plano.

> [!tip] Acceso al servidor FTP con usuario anónimo
> ```python
> ftp anonymous@10.129.230.176
> ```
>
> `anonymous` — Usuario especial que permite **acceso sin credenciales** en servidores FTP con login anónimo habilitado.
> `10.129.230.176` — **IP del servidor FTP** objetivo.

Una vez dentro, navegamos por la estructura de directorios del servidor. El acceso FTP anónimo nos posiciona directamente en la raíz del sistema de archivos de Windows, lo que nos permite movernos libremente. Navegamos hasta `C:\Users\Public\Desktop\` donde localizamos la flag de usuario.

![](/assets/Netmon-img-04-03-2026-7.png)

Al intentar leer el archivo directamente desde la sesión FTP comprobamos que no tenemos permisos de lectura en remoto. Esto es habitual en servidores FTP configurados con restricciones de lectura directa. La solución es sencilla: descargamos el archivo a nuestra máquina local con el comando `get` y lo leemos desde ahí con `cat`.

> [!note] La flag no puede leerse directamente en remoto
> Descargamos el archivo a local con `get` y lo leemos desde nuestra máquina.
> ```python
> get user.txt
> ```

![](/assets/Netmon-img-04-03-2026-8.png)

Flag de usuario obtenida. Continuamos con la explotación para escalar hacia root.

---

### 🌐 Reconocimiento del servicio web y exfiltración de credenciales

Accedemos al servicio web en el puerto 80 desde el navegador y nos encontramos con el panel de login de **PRTG Network Monitor**.

![](/assets/Netmon-img-05-03-2026.png)

> [!info] ¿Qué es PRTG Network Monitor?
> **PRTG** es una herramienta de monitorización de infraestructura de red ampliamente utilizada en entornos empresariales. Permite supervisar el estado de dispositivos, servicios, tráfico de red y métricas del sistema en tiempo real. Sus archivos de configuración almacenan parámetros del sistema, tareas programadas y en muchas versiones, **credenciales en texto plano o codificado en Base64**, lo que los convierte en un objetivo de alto valor durante una auditoría.

Con el software identificado, buscamos dónde almacena PRTG sus archivos de configuración. Una búsqueda rápida nos confirma que la ruta por defecto en Windows es:
```
C:\ProgramData\Paessler\PRTG Network Monitor
```

Aprovechando el acceso FTP anónimo ya establecido, navegamos hasta ese directorio. Encontramos varios archivos de configuración: el activo (`PRTG Configuration.dat`), uno antiguo (`PRTG Configuration.old`) y el más interesante: **`PRTG Configuration.old.bak`**, un backup de configuración que podría contener credenciales de una versión anterior de la aplicación.

> [!info] ¿Por qué son tan valiosos los archivos .bak?
> Los archivos de backup suelen conservar configuraciones antiguas que incluyen credenciales que en su momento eran válidas. Aunque la aplicación haya cambiado su contraseña, es frecuente que los administradores sigan patrones predecibles al actualizarlas, lo que los hace explotables mediante **mutación contextual de credenciales**.

Descargamos el archivo con `get` y lo analizamos con `cat`. Revisando el contenido encontramos credenciales en texto plano:

> [!important] Credenciales encontradas en el archivo de configuración
> **Usuario:** `prtgadmin`
> **Contraseña:** `PrTg@dmin2018`

> [!note] Mutación contextual de credenciales
> El archivo de backup data de **2018**, pero la máquina fue creada en **2019**. Es razonable asumir que el administrador actualizó la contraseña ese año siguiendo el mismo patrón. Probamos `PrTg@dmin2019` cambiando únicamente el año — y efectivamente, es la contraseña activa. Este tipo de razonamiento contextual es una técnica habitual en pentesting real: cuando una credencial falla, analiza el patrón e infiere variantes plausibles antes de descartar el vector.

Con las credenciales correctas accedemos al panel de PRTG:

![](/assets/Netmon-img-05-03-2026-1.png)
![](/assets/Netmon-img-05-03-2026-2.png)

---

### 🔓 Explotación de PRTG Network Monitor — RCE autenticado

Con acceso autenticado al panel, el siguiente paso es determinar si la versión instalada tiene vulnerabilidades conocidas. Lanzamos **searchsploit** para buscar exploits públicos:

![](/assets/Netmon-img-05-03-2026-6.png)

Encontramos el exploit **46527**, un script bash que aprovecha una vulnerabilidad de **inyección de comandos autenticada** en PRTG. Aunque la versión no coincide exactamente con la instalada, es suficientemente cercana como para probarlo. La vulnerabilidad permite a un usuario autenticado con privilegios de administrador ejecutar comandos arbitrarios en el sistema operativo subyacente a través de la funcionalidad de notificaciones de PRTG.

![](/assets/Netmon-img-05-03-2026-7.png)

El exploit requiere nuestra **cookie de sesión activa** como parámetro de autenticación para operar contra la API de PRTG. Para obtenerla abrimos las herramientas de desarrollo del navegador con `F12`, navegamos a la pestaña **Storage → Cookies** y copiamos el valor de la cookie de sesión de PRTG.

![](/assets/Netmon-img-05-03-2026-8.png)
![](/assets/Netmon-img-05-03-2026-9.png)

Con la cookie en mano, ejecutamos el exploit:

> [!tip] Ejecutar el exploit de PRTG con la cookie de sesión
> ```python
> ./46527.sh -u http://10.129.230.176 -c "OCTOPUS1813713946=ezkyRTYzN0M3LUQ4MTctNEZCNi1CNEY0LTQ3MjkzNkY1MjNCQ30%3D"
> ```
>
> `-u` — **URL del panel PRTG** objetivo.
> `-c` — **Cookie de sesión activa**, necesaria para que el exploit se autentique correctamente contra la API de PRTG y ejecute los comandos con privilegios de administrador.

![](/assets/Netmon-img-05-03-2026-10.png)

El exploit ejecuta con éxito y crea automáticamente un nuevo usuario local en el sistema: `pentest` con contraseña `P3nT3st!` y perteneciente al grupo **Administrators**. Con estas credenciales utilizamos **psexec** de Impacket para obtener una shell remota interactiva como `NT AUTHORITY\SYSTEM`.

> [!info] ¿Qué es Impacket y psexec?
> **Impacket** es una colección de scripts Python para interactuar con protocolos de red Windows como SMB, RPC o Kerberos. Su script `psexec.py` replica el comportamiento de la herramienta PsExec de Sysinternals: se autentica vía SMB, sube un binario de servicio temporal al objetivo y lo ejecuta, devolviendo una shell interactiva como `SYSTEM`. Es una de las técnicas de ejecución remota más utilizadas en pentesting de entornos Windows.

> [!tip] Obtener shell remota con psexec e Impacket
> ```python
> python3 /usr/share/doc/python3-impacket/examples/psexec.py pentest@10.129.230.176
> ```
>
> `psexec.py` — Script de Impacket que utiliza SMB para ejecutar comandos remotos en sistemas Windows, obteniendo una shell interactiva como `SYSTEM`.
> `pentest@10.129.230.176` — **Usuario creado por el exploit** y **IP objetivo**. La contraseña `P3nT3st!` se introduce de forma interactiva.

![](/assets/Netmon-img-05-03-2026-11.png)

Obtenemos shell directamente como `NT AUTHORITY\SYSTEM`. Navegamos hasta `C:\Users\Administrator\Desktop\` para recuperar la flag de root y completar la máquina.

---

## 🔼 Escalada de Privilegios

La cadena de explotación ejecutada — **credenciales en backup → acceso al panel PRTG → RCE autenticado → psexec** — otorga privilegios de `SYSTEM` directamente desde el acceso inicial. No es necesaria ninguna técnica de escalada adicional.

---

## 🏁 Flags

| Flag | Hash |
|------|------|
| 🧑 User | `...` |
| 👑 Root | `...` |

---

## 📝 Lecciones Aprendidas

> [!tip] Takeaways
> Netmon encadena tres conceptos fundamentales que merece la pena interiorizar. El primero es que el **login anónimo FTP en un servidor Windows** es una vulnerabilidad crítica de configuración: no solo expone archivos del sistema, sino que puede dar acceso a rutas de aplicaciones instaladas con información sensible. El segundo es la técnica de **mutación contextual de credenciales**: encontrar `PrTg@dmin2018` en un backup y deducir que la contraseña activa podría ser `PrTg@dmin2019` es un razonamiento que en entornos reales marca la diferencia entre acceder o no. El tercero es el flujo **credenciales web → exploit autenticado → psexec**, una cadena de ataque muy común en entornos Windows corporativos: obtener acceso a un panel de administración de software empresarial suele ser suficiente para comprometer el sistema operativo subyacente por completo.