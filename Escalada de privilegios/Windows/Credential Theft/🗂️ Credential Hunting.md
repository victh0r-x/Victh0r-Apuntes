
## 🧠 Concepto clave

La búsqueda de credenciales es una de las fases más rentables de la escalada de privilegios en Windows. Los administradores de sistemas, desarrolladores y usuarios cometen con frecuencia errores de higiene de credenciales: contraseñas en archivos de configuración, scripts de automatización con credenciales embebidas, historial de comandos con passwords pasadas como parámetros, o archivos de instalación desatendida olvidados en el sistema. Encontrar una sola credencial válida puede significar acceso directo a una cuenta administradora local, un pivote al dominio de Active Directory, o la escalada horizontal a otro usuario con más privilegios.

| Fuente | Tipo de credencial habitual | Probabilidad en entornos reales |
|---|---|---|
| Archivos de configuración de aplicaciones | Contraseñas de BBDD, APIs, servicios | Alta |
| Archivos de instalación desatendida | Contraseñas de administrador local | Media |
| Historial de PowerShell | Comandos con `-Password`, `-p`, credenciales en texto claro | Alta |
| Credenciales cifradas PowerShell (DPAPI) | Contraseñas de cuentas de servicio y administración | Media |
| Diccionarios del navegador | Contraseñas accidentalmente añadidas como "palabras" | Baja-Media |
| web.config de IIS | Cadenas de conexión a BBDD, credenciales de app pools | Media |

---
## 📌 Fuentes de credenciales

### 1 — Archivos de configuración de aplicaciones

Las aplicaciones frecuentemente almacenan credenciales en texto claro en archivos de configuración, en contra de todas las buenas prácticas. Estos archivos suelen tener extensiones como `.txt`, `.ini`, `.cfg`, `.config` o `.xml` y pueden contener cadenas de conexión a bases de datos, credenciales de APIs, contraseñas de servicios o cuentas de administrador.

> [!info] **findstr** 📄 [findstr — SS64](https://ss64.com/nt/findstr.html)
> Herramienta nativa de Windows equivalente a `grep` en Linux. Permite buscar cadenas de texto o patrones en archivos de forma recursiva. Es el punto de partida estándar para credential hunting en Windows por su disponibilidad universal — no requiere herramientas externas. Soporta expresiones regulares básicas, búsqueda en múltiples extensiones simultáneamente y búsqueda recursiva en subdirectorios.

> [!tip] Buscar la cadena "password" de forma recursiva en archivos de configuración comunes
> ```powershell
> findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
> ```
> `/S` — Búsqueda **recursiva** en el directorio actual y todos sus subdirectorios.
> `/I` — **Case-insensitive**: encuentra `password`, `Password`, `PASSWORD`, etc.
> `/M` — Muestra solo el **nombre del archivo** que contiene la coincidencia, no el contenido. Útil para un primer barrido rápido. Quitar `/M` para ver las líneas completas.
> `/C:"password"` — Cadena de búsqueda literal. Usar `/C:` garantiza que la cadena se trata como literal aunque contenga espacios.
> `*.txt *.ini *.cfg *.config *.xml` — Extensiones objetivo. Ampliar con `*.ps1 *.bat *.vbs *.sql` para mayor cobertura.

Ampliar la búsqueda con otras cadenas frecuentes:

> [!tip] Búsqueda ampliada con múltiples términos de interés
> ```powershell
> findstr /SIM /C:"password" /C:"passwd" /C:"pwd" /C:"secret" /C:"credential" /C:"connectionstring" *.txt *.ini *.cfg *.config *.xml *.ps1 *.bat
> ```

**Archivos de especial interés:**

| Archivo | Ubicación habitual | Contenido potencial |
|---|---|---|
| `web.config` | `C:\inetpub\wwwroot\web.config` | Cadenas de conexión a BBDD, credenciales de app pools IIS |
| `appsettings.json` | Directorio de la aplicación .NET | Credenciales de servicios, APIs, cadenas de conexión |
| `*.config` | Directorios de instalación de aplicaciones | Configuración de servicios con credenciales embebidas |
| `unattend.xml` / `sysprep.xml` | `C:\Windows\Panther\`, `C:\Windows\System32\Sysprep\` | Credenciales de administrador del proceso de instalación |
| `Groups.xml` | `C:\ProgramData\Microsoft\Group Policy\` (si existe localmente) | Contraseñas de GPP (Group Policy Preferences) — cifradas con AES pero con clave pública conocida |

> [!tip] Buscar web.config recursivamente en todo el sistema
> ```powershell
> Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue -Filter "web.config" | Select-Object FullName
> ```
> `-Recurse` — Búsqueda recursiva desde `C:\`.
> `-ErrorAction SilentlyContinue` — Ignora errores de acceso denegado sin interrumpir la búsqueda.
> `-Filter "web.config"` — Filtra por nombre de archivo exacto.

---

### 2 — Diccionarios del navegador Chrome

Un vector poco conocido pero sorprendentemente efectivo. Cuando un usuario escribe en un campo de texto en Chrome (formularios web, aplicaciones SaaS, webmail), Chrome subraya en rojo las palabras que no reconoce. Si el usuario añade esa "palabra" al diccionario para eliminar el subrayado, y esa "palabra" era en realidad una contraseña, queda almacenada en texto claro en el archivo del diccionario personalizado.

> [!info] **Chrome Custom Dictionary**
> Archivo de texto plano donde Chrome almacena las palabras añadidas por el usuario al diccionario personalizado. Se ubica en `C:\Users\<usuario>\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt`. Cada palabra añadida ocupa una línea. No hay ningún tipo de cifrado ni protección — cualquier proceso con acceso al perfil del usuario puede leerlo.

> [!tip] Leer el diccionario personalizado de Chrome buscando contraseñas
> ```powershell
> gc 'C:\Users\htb-student\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
> ```
> `gc` — Alias de `Get-Content`. Lee el contenido del archivo línea a línea.
> `Select-String password` — Filtra las líneas que contengan la cadena `password`. Quitar el filtro para ver todas las entradas del diccionario.

> [!tip] Buscar diccionarios de Chrome en todos los perfiles de usuario del sistema
> ```powershell
> foreach($user in ((ls C:\Users).fullname)){
>     $dict = "$user\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt"
>     if(Test-Path $dict){ gc $dict }
> }
> ```

---

### 3 — Archivos de instalación desatendida (Unattend.xml)

Los archivos de instalación desatendida (`unattend.xml`, `sysprep.xml`, `autounattend.xml`) se usan para automatizar la instalación de Windows en múltiples máquinas, definiendo configuraciones como el nombre del equipo, configuración de red, cuentas a crear y **credenciales de inicio de sesión automático**. Deben eliminarse automáticamente tras la instalación, pero frecuentemente los administradores conservan copias en otras carpetas durante el desarrollo de la imagen.

Las contraseñas en estos archivos pueden estar en **texto claro** o codificadas en **Base64** — lo que no es cifrado, solo codificación. Decodificar Base64 con `[System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String("valor"))` revela la contraseña inmediatamente.
```xml
<AutoLogon>
    <Password>
        <Value>local_4dmin_p@ss</Value>   <!-- Texto claro -->
        <PlainText>true</PlainText>
    </Password>
    <Enabled>true</Enabled>
    <Username>Administrator</Username>
</AutoLogon>
```

> [!tip] Buscar archivos de instalación desatendida en ubicaciones comunes
> ```powershell
> Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue -Include "unattend.xml","sysprep.xml","autounattend.xml" | Select-Object FullName
> ```

> [!tip] Buscar credenciales directamente en archivos unattend con findstr
> ```powershell
> findstr /SIM /C:"password" C:\Windows\Panther\unattend.xml
> findstr /SIM /C:"password" C:\Windows\System32\Sysprep\sysprep.xml
> ```

**Ubicaciones estándar donde buscar estos archivos:**
```
C:\Windows\Panther\
C:\Windows\Panther\Unattend\
C:\Windows\System32\Sysprep\
C:\Windows\System32\Sysprep\Panther\
C:\
C:\Windows\
```

---

### 4 — Historial de PowerShell (PSReadLine)

Desde PowerShell 5.0 (Windows 10), **PSReadLine** guarda automáticamente el historial de comandos en un archivo de texto plano. Si un administrador alguna vez ejecutó un comando pasando una contraseña como parámetro — usando `wevtutil`, `net use`, `Invoke-Command`, conexiones a bases de datos, etc. — esa contraseña queda registrada en texto claro en el historial.

> [!info] **PSReadLine**
> Módulo de PowerShell que mejora la experiencia de línea de comandos con autocompletado, historial persistente entre sesiones, sintaxis coloreada y más. Su historial se guarda en `C:\Users\<usuario>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`. El archivo no tiene cifrado ni protección adicional — cualquier proceso con acceso al perfil del usuario puede leerlo.

> [!info] **wevtutil** 📄 [wevtutil — SS64](https://ss64.com/nt/wevtutil.html)
> Herramienta nativa de Windows para gestionar logs de eventos: consultarlos, exportarlos, limpiarlos, etc. Acepta credenciales como parámetros (`/u:usuario /p:contraseña`) para acceder a logs de sistemas remotos. Es uno de los comandos más frecuentes en el historial de PowerShell que filtra credenciales en texto claro. 📄 [Windows Commands PDF — Microsoft](https://download.microsoft.com/download/5/8/9/58911986-D4AD-4695-BF63-F734CD4DF8F2/ws-commands.pdf)

> [!tip] Obtener la ruta del archivo de historial de PowerShell del usuario actual
> ```powershell
> (Get-PSReadLineOption).HistorySavePath
> ```
> `Get-PSReadLineOption` — Devuelve la configuración actual de PSReadLine, incluyendo `HistorySavePath` con la ruta exacta del archivo de historial. Útil si el administrador ha cambiado la ruta por defecto.

> [!tip] Leer el historial de PowerShell del usuario actual
> ```powershell
> gc (Get-PSReadLineOption).HistorySavePath
> ```
> `gc` — `Get-Content`. Lee el archivo de historial línea a línea. Buscar manualmente comandos con `password`, `-p `, `/p:`, `credential`, `secure`, `secret`.

> [!tip] Leer el historial de PowerShell de TODOS los usuarios del sistema
> ```powershell
> foreach($user in ((ls C:\users).fullname)){
>     cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue
> }
> ```
> `ls C:\users` — Lista todos los directorios de perfil de usuario en `C:\Users\`.
> `.fullname` — Obtiene la ruta completa de cada directorio.
> `-ErrorAction SilentlyContinue` — Ignora silenciosamente los archivos a los que no tenemos acceso, sin interrumpir el bucle.

> [!important] Este one-liner solo devuelve historiales de usuarios cuyos perfiles son accesibles con los permisos actuales. Si se obtiene acceso de administrador local posteriormente, **relanzarlo** — los historiales de otros usuarios pueden contener credenciales de cuentas de dominio o de servicio que no eran accesibles antes.

**Ejemplo de credencial filtrada en historial:**
```powershell
wevtutil qe Application "/q:*[Application [(EventID=3005)]]" /f:text /rd:true /u:WEB02\administrator /p:5erv3rAdmin! /r:WEB02
```
En este ejemplo, la contraseña `5erv3rAdmin!` del administrador de `WEB02` quedó registrada en texto claro en el historial de quien ejecutó ese comando.

---

### 5 — Credenciales cifradas de PowerShell (DPAPI / Export-Clixml)

Los administradores de sistemas frecuentemente crean scripts de PowerShell para tareas automatizadas (conexiones a vCenter, bases de datos, APIs, etc.) que necesitan autenticarse. Una práctica común es almacenar las credenciales en un archivo XML cifrado con `Export-Clixml` y recuperarlas en el script con `Import-Clixml`. Aunque el cifrado es real (usa DPAPI), si tenemos ejecución de código en el contexto del usuario que creó esas credenciales, podemos descifrarlas trivialmente.

> [!info] **DPAPI (Data Protection API)** 📄 [DPAPI — Wikipedia](https://en.wikipedia.org/wiki/Data_Protection_API)
> API de Windows que proporciona cifrado y descifrado de datos vinculado a la identidad del usuario o del sistema. Los datos cifrados con DPAPI **solo pueden descifrarse por el mismo usuario en el mismo equipo** — la clave de cifrado se deriva de las credenciales del usuario y un secreto del sistema. Se usa internamente en Windows para proteger contraseñas del navegador, credenciales guardadas, certificados privados y, en este contexto, credenciales exportadas con `Export-Clixml`. Si tenemos ejecución de código como ese usuario (por shell, RDP, etc.), DPAPI descifrará transparentemente sin necesitar la clave explícita.

> [!info] **Export-Clixml / Import-Clixml**
> Cmdlets de PowerShell para serializar y deserializar objetos .NET a/desde XML. Cuando se usa con objetos `PSCredential`, el campo contraseña se cifra automáticamente con DPAPI antes de escribirse en el XML. El archivo resultante parece protegido, pero cualquier proceso corriendo como el mismo usuario puede descifrarlo llamando a `Import-Clixml` seguido de `GetNetworkCredential().Password`.

Un ejemplo típico de script vulnerable:
```powershell
# Connect-VC.ps1 — Script de administrador para conectar a vCenter
# El administrador generó el XML así:
# Get-Credential | Export-Clixml -Path 'C:\scripts\pass.xml'

$encryptedPassword = Import-Clixml -Path 'C:\scripts\pass.xml'
$decryptedPassword = $encryptedPassword.GetNetworkCredential().Password
Connect-VIServer -Server 'VC-01' -User 'bob_adm' -Password $decryptedPassword
```

> [!tip] Buscar archivos XML de credenciales exportadas con Export-Clixml
> ```powershell
> Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue -Filter "*.xml" | Select-String "SecureString" | Select-Object Path
> ```
> `Select-String "SecureString"` — Los XMLs generados por `Export-Clixml` con credenciales contienen la cadena `SecureString` en su interior, lo que los identifica inequívocamente.

> [!tip] Descifrar credenciales DPAPI almacenadas en un archivo XML
> ```powershell
> $credential = Import-Clixml -Path 'C:\scripts\pass.xml'
> $credential.GetNetworkCredential().username
> $credential.GetNetworkCredential().password
> ```
> `Import-Clixml` — Deserializa el XML y descifra automáticamente el campo `SecureString` usando DPAPI en el contexto del usuario actual.
> `.GetNetworkCredential()` — Convierte el objeto `PSCredential` en un objeto `NetworkCredential` que expone usuario y contraseña en texto claro.
> `.username` / `.password` — Propiedades que devuelven las credenciales descifradas en texto plano.

> [!important] `Import-Clixml` solo descifra correctamente si se ejecuta como el **mismo usuario** que cifró las credenciales originalmente. Si se intenta desde otro usuario (incluso administrador), DPAPI devolverá un error de acceso. Para descifrar credenciales DPAPI de otro usuario se necesitan técnicas adicionales como volcar la Master Key de DPAPI con Mimikatz o usar `dpapi::cred` con la clave del sistema.

---

## 🛠️ Búsqueda automatizada — One-liners de cobertura amplia

> [!tip] Buscar contraseñas en todos los archivos de texto del sistema (búsqueda agresiva)
> ```powershell
> Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue | Select-String -Pattern "password|passwd|pwd|secret|credential" -ErrorAction SilentlyContinue | Select-Object Path, LineNumber, Line
> ```
> `Select-String -Pattern` — Acepta expresiones regulares. El patrón `password|passwd|pwd` busca cualquiera de los términos.
> `Select-Object Path, LineNumber, Line` — Muestra la ruta del archivo, el número de línea y el contenido de la línea coincidente.

> [!tip] Buscar todos los archivos con "pass" o "cred" en el nombre
> ```powershell
> Get-ChildItem -Path C:\Users -Recurse -ErrorAction SilentlyContinue | Where-Object {$_.Name -match "pass|cred|secret|key"} | Select-Object FullName
> ```
> Busca archivos cuyo nombre contenga `pass`, `cred`, `secret` o `key` — frecuentemente archivos como `passwords.txt`, `credentials.xml`, `api_keys.txt`, etc.

> [!tip] Buscar archivos unattend, sysprep y configuración en ubicaciones estándar
> ```powershell
> $paths = @(
>     "C:\Windows\Panther\unattend.xml",
>     "C:\Windows\System32\Sysprep\sysprep.xml",
>     "C:\inetpub\wwwroot\web.config",
>     "C:\Windows\system32\GroupPolicy\DataStore"
> )
> foreach($p in $paths){ if(Test-Path $p){ Write-Host "[FOUND] $p"; gc $p } }
> ```

---

## 🔗 Relaciones / Contexto

El credential hunting es un paso transversal que debe realizarse en **paralelo** con la enumeración de vectores de escalada, no de forma secuencial. Una credencial encontrada puede resolver directamente la escalada sin necesidad de explotar ninguna vulnerabilidad técnica — lo que en un pentest real es el hallazgo más impactante posible, ya que demuestra que la postura de seguridad falla en lo más básico: la higiene de credenciales. Los archivos de historial de PowerShell y los scripts de automatización con `Export-Clixml` son especialmente frecuentes en entornos empresariales maduros que han automatizado tareas de administración. Los archivos `unattend.xml` son más habituales en entornos con despliegue masivo de equipos (empresas grandes, universidades, entornos de laboratorio). Las credenciales encontradas deben probarse siempre en múltiples servicios — reutilización de contraseñas en WinRM, SMB, RDP, y especialmente en el dominio de Active Directory.