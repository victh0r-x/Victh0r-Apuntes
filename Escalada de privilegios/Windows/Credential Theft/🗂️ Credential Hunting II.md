
## 🧠 Concepto clave

Más allá de los archivos de configuración y el historial de PowerShell, Windows acumula credenciales en lugares inesperados: bases de datos SQLite de aplicaciones de notas, archivos de reparación del sistema con copias de la SAM, logs de instalación de red, archivos de perfil de usuario, o simplemente en documentos Word y Excel guardados en carpetas compartidas con permisos excesivamente permisivos. En entornos de Active Directory, cada empleado suele tener una carpeta personal en un share corporativo que considera "privada" pero que con frecuencia es legible por todos los usuarios del dominio.

| Fuente                               | Tipo de credencial                               | Herramienta de acceso             |
| ------------------------------------ | ------------------------------------------------ | --------------------------------- |
| Archivos en filesystem local         | Contraseñas en texto claro en docs               | `findstr`, `Select-String`, `dir` |
| Carpetas compartidas de red          | Credenciales en archivos personales              | Snaffler, acceso manual           |
| StickyNotes (plum.sqlite)            | Notas con contraseñas almacenadas por el usuario | PSSQLite, DB Browser, strings     |
| Archivos de reparación Windows       | Copias offline de SAM/SYSTEM/SECURITY            | Acceso directo, secretsdump       |
| Archivos de paginación y hibernación | Credenciales en memoria volcadas a disco         | strings, volatility               |
| Documentos ofimáticos                | Contraseñas embebidas en Word/Excel/OneNote      | Búsqueda manual, Snaffler         |
| Archivos de configuración VNC/RDP    | Contraseñas de acceso remoto                     | Búsqueda por extensión            |

---

## 📌 Búsqueda manual en el filesystem

### Buscar contenido en archivos con findstr

> [!info] **findstr**
> Herramienta nativa de Windows equivalente a `grep` en Linux. Permite buscar cadenas literales o patrones en el contenido de archivos, de forma recursiva a través de directorios y con soporte para múltiples extensiones en una sola invocación. Es la herramienta de búsqueda más disponible en cualquier sistema Windows sin dependencias externas.

Los flags más relevantes de `findstr` para credential hunting:

| Flag | Función |
|---|---|
| `/S` | Buscar recursivamente en subdirectorios |
| `/I` | Case-insensitive |
| `/M` | Mostrar solo el nombre del archivo (no el contenido de la línea) |
| `/N` | Mostrar número de línea de cada coincidencia |
| `/P` | Ignorar archivos con caracteres no imprimibles |
| `/I` | Sin distinción mayúsculas/minúsculas |
| `/C:"cadena"` | Cadena de búsqueda literal (preserva espacios) |
| `/spin` | Combinación de `/S /P /I /N` — el combo más útil |

> [!tip] Buscar "password" en archivos comunes — solo nombres de archivo
> ```cmd
> cd c:\Users\htb-student\Documents
> findstr /SI /M "password" *.xml *.ini *.txt
> ```
> `/S` — Búsqueda recursiva desde el directorio actual.
> `/I` — Case-insensitive: encuentra `password`, `Password`, `PASSWORD`.
> `/M` — Muestra solo el nombre del archivo, no el contenido. Ideal para un primer barrido rápido cuando hay muchos archivos — si hay coincidencia, se examina el archivo después.

> [!tip] Buscar "password" mostrando el contenido de las líneas coincidentes
> ```cmd
> findstr /si password *.xml *.ini *.txt *.config
> ```
> Sin `/M` — Muestra `nombrearchivo:contenido_de_la_línea` para cada coincidencia, lo que permite ver la credencial directamente sin abrir el archivo.

> [!tip] Búsqueda recursiva completa con número de línea en todos los archivos
> ```cmd
> findstr /spin "password" *.*
> ```
> `/spin` — Combinación de cuatro flags: `/S` (recursivo) + `/P` (saltar binarios) + `/I` (case-insensitive) + `/N` (número de línea).
> `*.*` — Todos los archivos del directorio actual y subdirectorios. Más agresivo que especificar extensiones — puede ser lento en directorios grandes pero maximiza la cobertura.

> [!tip] Búsqueda ampliada con múltiples términos en archivos de configuración
> ```cmd
> findstr /spin "password" *.txt *.xml *.ini *.config *.ps1 *.bat *.vbs *.sql
> findstr /spin "passwd" *.*
> findstr /spin "credential" *.*
> findstr /spin "connectionstring" *.config *.xml
> ```

---

### Buscar contenido con PowerShell

> [!info] **Select-String**
> Cmdlet de PowerShell equivalente a `grep`. Busca patrones (texto literal o regex) en el contenido de archivos. Devuelve objetos con las propiedades `Path`, `LineNumber` y `Line`, lo que facilita el post-procesamiento. Soporta múltiples archivos con wildcards y búsqueda recursiva combinado con `Get-ChildItem`.

> [!tip] Buscar "password" en archivos de texto con Select-String
> ```powershell
> Select-String -Path C:\Users\htb-student\Documents\*.txt -Pattern "password"
> ```
> `-Path` — Ruta con wildcard. Soporta múltiples patrones separados por comas: `-Path *.txt,*.xml,*.ini`.
> `-Pattern` — La cadena o regex a buscar. Por defecto es case-insensitive.

> [!tip] Búsqueda recursiva en todo el árbol de directorios
> ```powershell
> Get-ChildItem -Path C:\Users -Recurse -ErrorAction SilentlyContinue -Include *.txt,*.xml,*.ini,*.config,*.ps1 |
>     Select-String -Pattern "password|passwd|credential|secret" |
>     Select-Object Path, LineNumber, Line
> ```
> `Get-ChildItem -Recurse` — Encuentra todos los archivos de las extensiones especificadas recursivamente.
> `Select-String -Pattern` — Acepta regex: `password|passwd|credential` busca cualquiera de los tres términos.
> `Select-Object Path, LineNumber, Line` — Muestra ruta del archivo, número de línea y contenido de la línea.

---

### Buscar archivos por nombre o extensión

> [!tip] Buscar archivos por nombre con dir
> ```cmd
> dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
> ```
> `/S` — Recursivo en todos los subdirectorios.
> `/B` — Bare format — solo muestra las rutas completas, sin cabeceras ni metadatos.
> `*pass*.txt` — Cualquier archivo `.txt` cuyo nombre contenga "pass".
> `*cred*` — Cualquier archivo cuyo nombre contenga "cred" (cualquier extensión).
> `*vnc*` — Archivos VNC frecuentemente contienen contraseñas cifradas débilmente.

> [!tip] Buscar archivos .config con where
> ```cmd
> where /R C:\ *.config
> ```
> `/R C:\` — Búsqueda recursiva comenzando desde `C:\`. `where` es más rápido que `dir /S` para búsquedas de extensión específica.

> [!tip] Buscar extensiones de interés con PowerShell
> ```powershell
> Get-ChildItem C:\ -Recurse -Include *.rdp,*.config,*.vnc,*.cred,*.kdbx,*.ppk -ErrorAction Ignore
> ```
> `-Include` — Lista de extensiones a incluir. Cada extensión separada por coma.
> `-ErrorAction Ignore` — Silencia los errores de acceso denegado sin interrumpir la búsqueda.
> `.rdp` — Archivos de conexión RDP pueden contener credenciales guardadas.
> `.kdbx` — Base de datos KeePass — requiere crackeo de master password.
> `.ppk` — Claves privadas PuTTY — pueden usarse directamente o crackearse con `putty2john`.

> [!tip] Búsqueda ampliada de extensiones de interés ofensivo
> ```powershell
> $extensions = @("*.rdp","*.config","*.vnc","*.cred","*.kdbx","*.ppk",
>                 "*.key","*.pem","*.pfx","*.p12","*.ovpn","*.vmdk",
>                 "*.vhdx","*.id_rsa","*id_ed25519*","*.xlsx","*.docx")
> Get-ChildItem C:\ -Recurse -Include $extensions -ErrorAction Ignore |
>     Select-Object FullName, Length, LastWriteTime
> ```
> `.ovpn` — Perfiles de VPN — frecuentemente incluyen certificados y credenciales embebidas.
> `.vmdk` / `.vhdx` — Discos virtuales — pueden montarse para extraer hashes de la SAM.
> `.pem` / `.p12` / `.pfx` — Certificados con clave privada — útiles para autenticación.

---

## 📌 StickyNotes — Base de datos SQLite con notas del usuario

StickyNotes es la aplicación de notas adhesivas incluida en Windows 10/11. Los usuarios la usan con frecuencia para guardar contraseñas, IPs, credenciales de acceso remoto y otra información sensible sin ser conscientes de que los datos se almacenan en una **base de datos SQLite sin cifrado** en el perfil del usuario.

**Ruta de la base de datos:**
```
C:\Users\<usuario>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite
```

Los tres archivos relevantes son `plum.sqlite`, `plum.sqlite-shm` y `plum.sqlite-wal`. El archivo WAL (Write-Ahead Log) contiene las transacciones recientes y frecuentemente incluye las notas más actuales — hay que examinar los tres.

> [!info] **plum.sqlite**
> Base de datos SQLite donde StickyNotes almacena todas las notas del usuario. La tabla `Note` contiene el campo `Text` con el contenido de cada nota en texto plano. No hay cifrado ni protección de acceso más allá de los permisos del sistema de archivos del perfil del usuario. Cualquier proceso ejecutándose como ese usuario — o como administrador — puede leer el contenido íntegro.

> [!tip] Localizar bases de datos StickyNotes en todos los perfiles
> ```powershell
> Get-ChildItem -Path "C:\Users\*\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite" -ErrorAction SilentlyContinue
> ```

### Método 1 — PSSQLite (PowerShell directo)

> [!info] **PSSQLite** 📄 [RamblingCookieMonster/PSSQLite — GitHub](https://github.com/RamblingCookieMonster/PSSQLite)
> Módulo de PowerShell que permite ejecutar consultas SQL contra bases de datos SQLite directamente desde PowerShell, sin necesitar herramientas externas instaladas. Especialmente útil en entornos donde no se puede instalar software o no se tiene acceso GUI. Permite volcar el contenido de `plum.sqlite` con una simple query SQL.

> [!tip] Leer contenido de StickyNotes con PSSQLite
> ```powershell
> Set-ExecutionPolicy Bypass -Scope Process -Force
>
> # Si PSSQLite está disponible localmente
> Import-Module .\PSSQLite.psd1
>
> $db = "C:\Users\htb-student\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite"
>
> Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | Format-Table -Wrap
> ```
> `Set-ExecutionPolicy Bypass -Scope Process` — Permite ejecutar scripts solo en la sesión actual sin modificar la política del sistema.
> `Invoke-SqliteQuery -Database $db -Query` — Ejecuta la query SQL contra el archivo SQLite especificado.
> `"SELECT Text FROM Note"` — Extrae el campo `Text` de todas las filas de la tabla `Note`.
> `Format-Table -Wrap` — Muestra el resultado en tabla permitiendo que el texto largo se divida en múltiples líneas.

> [!tip] Query completa para extraer toda la información relevante de StickyNotes
> ```powershell
> Invoke-SqliteQuery -Database $db -Query "SELECT Id, Text, CreatedAt, UpdatedAt FROM Note WHERE DeletedAt IS NULL" | Format-Table -Wrap
> ```
> `WHERE DeletedAt IS NULL` — Excluye notas eliminadas (que pueden seguir en el archivo).

### Método 2 — DB Browser for SQLite (herramienta gráfica)

> [!info] **DB Browser for SQLite** 📄 [sqlitebrowser.org](https://sqlitebrowser.org/dl/)
> Aplicación GUI gratuita para explorar, consultar y modificar bases de datos SQLite. Permite navegar por la estructura de tablas, ejecutar queries SQL y ver el contenido de forma visual. En pentesting se usa descargando los archivos `plum.sqlite*` al equipo del atacante para analizarlos offline.

Flujo de uso: copiar los tres archivos `plum.sqlite`, `plum.sqlite-shm` y `plum.sqlite-wal` al equipo atacante → abrir `plum.sqlite` en DB Browser → pestaña "Execute SQL" → ejecutar `SELECT Text FROM Note;`.

### Método 3 — strings (análisis rápido sin herramientas)

> [!tip] Extraer strings legibles del archivo WAL directamente
> ```bash
> # En el equipo atacante (Linux) tras transferir los archivos
> strings plum.sqlite-wal | grep -v "^.\{1\}$"
> strings plum.sqlite | grep -iE "(password|pass|user|admin|root|secret)"
> ```
> `strings` — Extrae todas las secuencias de caracteres imprimibles de longitud mínima (por defecto 4) de cualquier archivo binario. No requiere parsear el formato SQLite — simplemente vuelca texto legible del binario, lo que frecuentemente incluye el contenido de las notas.
> `-v "^.\{1\}$"` — Filtra líneas de un solo carácter para reducir el ruido.

---

## 📌 Archivos del sistema con copias de credenciales

Windows mantiene en varias ubicaciones copias de archivos del sistema que pueden contener hashes de contraseñas o credenciales. Muchos de estos son archivos legítimos del sistema operativo que quedan accesibles con ciertos permisos o en circunstancias específicas.

### Archivos de reparación y backup del sistema

| Ruta | Contenido potencial |
|---|---|
| `%WINDIR%\repair\sam` | Copia antigua de la SAM — hashes de contraseñas locales |
| `%WINDIR%\repair\system` | Copia antigua del hive SYSTEM — necesario para descifrar SAM |
| `%WINDIR%\repair\security` | Copia antigua del hive SECURITY — LSA secrets |
| `%WINDIR%\repair\software` | Copia antigua del hive SOFTWARE |
| `%WINDIR%\system32\config\default.sav` | Backup del hive DEFAULT |
| `%WINDIR%\system32\config\security.sav` | Backup del hive SECURITY |
| `%WINDIR%\system32\config\software.sav` | Backup del hive SOFTWARE |
| `%WINDIR%\system32\config\system.sav` | Backup del hive SYSTEM |

> [!important] Las copias de SAM en `%WINDIR%\repair\` son versiones de backup creadas durante la instalación o reparación del sistema. Pueden ser antiguas y no reflejar las contraseñas actuales, pero en entornos donde las contraseñas no se cambian frecuentemente, pueden ser perfectamente válidas. Para extraer los hashes se necesita tanto la SAM como el SYSTEM (para obtener la Boot Key que los descifra): `impacket-secretsdump -sam SAM -system SYSTEM local`.

### Archivos de memoria volcados a disco

| Ruta | Contenido potencial |
|---|---|
| `%SYSTEMDRIVE%\pagefile.sys` | Archivo de paginación — puede contener credenciales que estaban en RAM |
| `C:\hiberfil.sys` | Archivo de hibernación — volcado completo de la RAM al disco |
| `C:\Windows\MEMORY.DMP` | Volcado de memoria del sistema (tras BSOD) |

> [!important] `pagefile.sys` e `hiberfil.sys` son archivos bloqueados por el sistema y no accesibles directamente en un sistema en ejecución. Se accede a ellos montando el disco desde otro sistema o usando herramientas especializadas. `hiberfil.sys` es especialmente valioso porque es un volcado completo de la RAM y puede contener tokens de Kerberos, hashes NTLM en memoria, credenciales de sesiones activas, etc. Se analiza con Volatility o se convierte a formato de volcado estándar con `volatility -f hiberfil.sys --profile=Win10x64 hashdump`.

### Logs y archivos de configuración del sistema

| Ruta | Contenido potencial |
|---|---|
| `%WINDIR%\debug\NetSetup.log` | Log de unión al dominio — puede contener nombre de dominio, credenciales de unión |
| `%WINDIR%\iis6.log` | Logs IIS — pueden contener credenciales en parámetros de URLs |
| `%WINDIR%\system32\CCM\logs\*.log` | Logs de SCCM/ConfigMgr — credenciales de despliegue |
| `%USERPROFILE%\ntuser.dat` | Registro del usuario — MRU, configuración, credenciales guardadas |
| `%WINDIR%\System32\drivers\etc\hosts` | Resolución de nombres — revela infraestructura interna |
| `C:\ProgramData\Configs\*` | Configuraciones de aplicaciones corporativas |
| `C:\Program Files\Windows PowerShell\*` | Scripts de PowerShell del sistema |

> [!tip] Buscar todos los archivos de interés en ubicaciones del sistema
> ```powershell
> $targets = @(
>     "$env:WINDIR\repair\sam",
>     "$env:WINDIR\repair\system",
>     "$env:WINDIR\repair\security",
>     "$env:WINDIR\debug\NetSetup.log",
>     "$env:WINDIR\system32\config\default.sav",
>     "$env:WINDIR\system32\config\security.sav",
>     "$env:WINDIR\system32\config\system.sav"
> )
>
> foreach($t in $targets){
>     if(Test-Path $t){
>         $size = (Get-Item $t).Length
>         Write-Host "[FOUND] $t ($size bytes)"
>     }
> }
> ```

---

## 📌 Snaffler — Búsqueda automatizada en shares de red

> [!info] **Snaffler** 📄 [SnaffCon/Snaffler — GitHub](https://github.com/SnaffCon/Snaffler)
> Herramienta especializada en enumerar shares de red de Active Directory buscando archivos con extensiones o contenido de interés ofensivo. Autentica con las credenciales del usuario actual del dominio, enumera todos los shares accesibles, y clasifica los archivos encontrados por nivel de interés (rojo, amarillo, verde) según reglas predefinidas. Busca extensiones como `.kdbx` (KeePass), `.vmdk/.vhdx` (discos virtuales), `.ppk` (claves PuTTY), `.rdp`, archivos de configuración con contraseñas, archivos de Office, y muchos más. Es significativamente más eficiente que la búsqueda manual en entornos de dominio grandes.

> [!tip] Ejecutar Snaffler contra el dominio actual
> ```cmd
> Snaffler.exe -s -d dominio.local -o snaffler_output.log -v data
> ```
> `-s` — Imprimir resultados en stdout además del archivo de log.
> `-d dominio.local` — Dominio contra el que enumerar shares. Si se omite, usa el dominio del usuario actual.
> `-o snaffler_output.log` — Archivo de salida para los resultados.
> `-v data` — Nivel de verbosidad: `data` muestra el contenido de los archivos encontrados. Otras opciones: `info`, `debug`.

> [!tip] Snaffler contra un servidor específico
> ```cmd
> Snaffler.exe -s -t FILESERVER01 -o output.log
> ```
> `-t` — Target: enumerar solo el servidor especificado en lugar de todo el dominio.

Snaffler clasifica los hallazgos por colores: **rojo** (credenciales confirmadas o material de alto valor), **amarillo** (probable interés), **verde** (posible interés). Los hallazgos rojos son la primera prioridad de revisión.

---

## 📌 Archivos ofimáticos y documentos personales

En entornos corporativos, los usuarios guardan con frecuencia contraseñas en documentos de Office, archivos de texto, o workbooks de OneNote en sus carpetas de red personales. Estas carpetas suelen tener permisos de lectura para todos los usuarios del dominio — cada usuario asume que su carpeta es privada cuando en realidad es accesible por toda la organización.

> [!tip] Buscar documentos de Office y archivos de interés en shares accesibles
> ```powershell
> # Buscar archivos de Office que puedan contener credenciales
> Get-ChildItem -Path \\SERVIDOR\shares -Recurse -ErrorAction SilentlyContinue -Include *.xlsx,*.docx,*.onetoc2,*.txt,*.csv | Where-Object {$_.Name -match "pass|cred|secret|login|key"}
>
> # Buscar dentro de archivos de texto en shares
> Get-ChildItem -Path \\SERVIDOR\users -Recurse -ErrorAction SilentlyContinue -Include *.txt,*.csv,*.ini | Select-String -Pattern "password|passwd" -ErrorAction SilentlyContinue
> ```

> [!tip] Buscar archivo clásico passwords.txt en todo el dominio
> ```cmd
> findstr /SI /M "password" \\SERVIDOR\shares\*.txt
> dir /S /B \\SERVIDOR\users\*pass*.txt \\SERVIDOR\users\*cred*.txt
> ```

> [!important] Antes de abrir archivos `.docx` o `.xlsx` encontrados en shares de red, considerar que pueden contener macros maliciosas si el entorno ha sido comprometido por otro actor. Abrir en un sandbox o usar herramientas de extracción de texto sin ejecutar macros (`python-docx`, `strings`, `oletools`).

---

## 📐 Checklist de credential hunting en filesystem

> [!tip] One-liner de búsqueda completa para credential hunting rápido
> ```powershell
> # Búsqueda combinada — archivos por nombre y contenido
> Write-Host "=== Archivos con 'pass' en el nombre ===" -ForegroundColor Yellow
> Get-ChildItem C:\Users -Recurse -ErrorAction SilentlyContinue |
>     Where-Object {$_.Name -match "pass|cred|secret"} | Select-Object FullName
>
> Write-Host "=== Extensiones de alto valor ===" -ForegroundColor Yellow
> Get-ChildItem C:\ -Recurse -Include *.kdbx,*.ppk,*.rdp,*.vnc,*.config,*.ovpn -ErrorAction Ignore |
>     Select-Object FullName
>
> Write-Host "=== StickyNotes ===" -ForegroundColor Yellow
> Get-ChildItem "C:\Users\*\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite" -ErrorAction SilentlyContinue |
>     Select-Object FullName
>
> Write-Host "=== Archivos de reparación del sistema ===" -ForegroundColor Yellow
> @("$env:WINDIR\repair\sam","$env:WINDIR\repair\system","$env:WINDIR\repair\security") |
>     Where-Object {Test-Path $_} | ForEach-Object {Write-Host "[FOUND] $_" -ForegroundColor Red}
>
> Write-Host "=== Historial PowerShell todos los usuarios ===" -ForegroundColor Yellow
> foreach($user in (ls C:\users).fullname){
>     $h = "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt"
>     if(Test-Path $h){ Write-Host "[FOUND] $h"; gc $h | Select-String -Pattern "pass|cred|secret" }
> }
> ```

---

## 🔗 Relaciones / Contexto

Esta fase de credential hunting en archivos complementa directamente la búsqueda en configuraciones y el historial de PowerShell vistos anteriormente. La diferencia clave en entornos de Active Directory es la dimensión de **shares de red compartidos** — donde Snaffler automatiza lo que sería imposible hacer manualmente en una red corporativa con cientos de shares. Los archivos de reparación del sistema (`repair\sam` + `repair\system`) se encadenan directamente con `impacket-secretsdump` para extraer hashes sin necesidad de Mimikatz ni de tocar LSASS, evitando las alertas más comunes de los EDR modernos. StickyNotes es un hallazgo frecuentemente subestimado en auditorías — la combinación de que los usuarios la usan para guardar credenciales y que nadie la considera una superficie de ataque la convierte en una fuente muy productiva en entornos reales.