
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

Como se puede observar en el output, obtenemos respuesta con un TTL de **64**, lo que nos indica que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración.
Como se puede observar en el output, obtenemos respuesta con un TTL de **128**, lo que nos indica que estamos ante una máquina **Windows**. Confirmada la conectividad, procedemos con la enumeración.

### 👁️ Enumeración con NMAP

#### Enumeración de puertos

Comenzamos la fase de enumeración identificando qué puertos se encuentran abiertos en la máquina objetivo. Para ello utilizamos nmap en su modalidad de SYN Scan, que nos permite realizar un escaneo rápido y sigiloso sobre el rango completo de puertos sin completar el three-way handshake.

> [!tip] Escaneo de puertos completo y sigiloso
> ```python
> nmap -sS -p- --open --min-rate 5000 -n -Pn IP_MAQUINA -vvv -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open). Envía paquetes SYN sin completar el three-way handshake. Sigiloso y rápido, requiere root.
> `-p-` — Escanea los **65535 puertos** posibles.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `IP_MAQUINA` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN ports` — Guarda el output en formato normal en un archivo llamado `ports`.

---

#### Enumeración de versiones y servicios

Con los puertos abiertos ya identificados, el siguiente paso es determinar qué servicios y versiones están corriendo en cada uno de ellos. Esto nos permitirá identificar tecnologías concretas y posibles vectores de ataque. Para ello lanzamos nmap con sus scripts por defecto y detección de versiones, apuntando únicamente a los puertos descubiertos en el paso anterior.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```python
> nmap -sC -sV -pPUERTOS,SEPARADOS,POR,COMAS --min-rate 5000 -n -Pn -vvv IP_VICTIMA -oN version
> ```
>
> `-sC` — Lanza los **scripts por defecto** del motor NSE de nmap.
> `-sV` — **Detección de versiones** de los servicios que corren en cada puerto.
> `-pPUERTOS,SEPARADOS,POR,COMAS` — Escanea **únicamente los puertos encontrados** en el paso anterior.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `IP_VICTIMA` — **IP objetivo** del escaneo.
> `-oN version` — Guarda el output en formato normal en un archivo llamado `version`.

---

#### Enumeración de posibles vulnerabilidades

Con los servicios y versiones identificados, procedemos a lanzar el conjunto de scripts de la categoría `vuln` de nmap. Estos scripts comprueban de forma automática si los servicios detectados presentan vulnerabilidades conocidas, lo que nos dará pistas concretas sobre posibles vectores de explotación.

> [!tip] Detección automática de vulnerabilidades con scripts NSE
> ```python
> nmap --script vuln -pPUERTOS IP_VICTIMA --min-rate 5000 -n -Pn -vvv -oN vuln
> ```
>
> `--script vuln` — Lanza la categoría de scripts **vuln** del motor NSE, que comprueba vulnerabilidades conocidas en los servicios detectados.
> `-pPUERTOS` — Apunta únicamente a los **puertos ya identificados** anteriormente.
> `IP_VICTIMA` — **IP objetivo** del escaneo.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo, asume el host activo.
> `-vvv` — **Triple verbose**, muestra información en tiempo real mientras escanea.
> `-oN vuln` — Guarda el output en formato normal en un archivo llamado `vuln`.
---

## 💥 Explotación

---

## 🔼 Escalada de Privilegios

---

## 📝 Lecciones Aprendidas