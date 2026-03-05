tags:
____
## 🧠 Concepto clave
`SeDebugPrivilege` es un privilegio de Windows que permite a un proceso hacer attach, leer y modificar la memoria de cualquier otro proceso del sistema, incluyendo procesos del SO y del kernel. Por defecto solo lo tienen los administradores, pero puede asignarse a usuarios concretos vía Group Policy. Su abuso permite dos cosas muy poderosas: **volcar la memoria de LSASS para extraer credenciales** y **escalar directamente a SYSTEM mediante impersonación de procesos privilegiados**.

| Privilegio | Qué es | Riesgo / Explotación |
|---|---|---|
| `SeDebugPrivilege` | Permite hacer attach a cualquier proceso del sistema y leer/modificar su memoria, independientemente de quién sea su propietario | Acceso total a la memoria de LSASS (credenciales en texto claro y hashes NTLM) + escalada a SYSTEM heredando el token de un proceso privilegiado |

## 📌 Puntos importantes

### ¿Cuándo encontramos este privilegio?

Aunque por defecto solo los administradores lo tienen, en entornos reales puede aparecer asignado a perfiles técnicos como **desarrolladores** que necesitan depurar componentes del sistema. Durante un pentest interno, plataformas como LinkedIn pueden ayudar a identificar estos perfiles y priorizarlos como objetivos al crackear hashes NTLMv2 capturados con herramientas como Responder o Inveigh. Un usuario puede tener este privilegio sin ser administrador local, y herramientas como BloodHound no siempre lo enumeran remotamente — por eso vale la pena comprobarlo manualmente cuando se tiene acceso RDP a un host.

Para confirmar que el privilegio está presente se usa `whoami /priv` desde una consola elevada. Aparecerá en estado `Disabled` — que no significa que no se pueda usar, sino simplemente que no está activo en ese momento en el token. Muchos exploits lo activan automáticamente al ejecutarse.

---

### Vía 1 — Volcado de LSASS y extracción de credenciales

**LSASS** (Local Security Authority Subsystem Service) es el proceso de Windows responsable de gestionar la autenticación: verifica credenciales, genera tokens de acceso y, lo más importante desde el punto de vista ofensivo, **almacena en memoria las credenciales de los usuarios que han iniciado sesión** en forma de hashes NTLM, tickets Kerberos y en algunos casos contraseñas en texto claro. 📄 [LSASS — Wikipedia](https://en.wikipedia.org/wiki/Local_Security_Authority_Subsystem_Service)

Con `SeDebugPrivilege` se puede volcar la memoria de LSASS y procesarla offline con Mimikatz para extraer esas credenciales. El flujo es:

**Paso 1 — Volcar la memoria de LSASS con ProcDump** 📄 [ProcDump — Sysinternals](https://docs.microsoft.com/en-us/sysinternals/downloads/procdump)

ProcDump es una herramienta de Sysinternals 📄 [Sysinternals Suite](https://docs.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite) que permite crear volcados de memoria de procesos. Al apuntar a `lsass.exe`, genera un archivo `.dmp` con todo su contenido en memoria.
```cmd
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```

**Paso 2 — Cargar el volcado en Mimikatz y extraer credenciales**

Una vez transferido el `.dmp` a la máquina atacante, se procesa con Mimikatz. Es recomendable activar el logging antes de ejecutar cualquier comando para guardar toda la salida en un `.txt` — especialmente útil en servidores con muchas sesiones activas.
```cmd
mimikatz.exe
log
sekurlsa::minidump lsass.dmp
sekurlsa::logonpasswords
```

Esto devuelve los hashes NTLM y en algunos casos contraseñas en texto claro de todos los usuarios con sesión activa. Con esos hashes NTLM se pueden realizar ataques **pass-the-hash** para moverse lateralmente a otros sistemas — especialmente efectivo si la misma contraseña de administrador local se reutiliza en múltiples máquinas, algo muy común en organizaciones grandes.

**Alternativa sin herramientas — Volcado manual desde Task Manager**

Si no es posible subir herramientas al sistema pero se tiene acceso RDP, se puede crear el volcado directamente desde el Task Manager: pestaña **Details** → clic derecho sobre `lsass.exe` → **Create dump file**. El archivo resultante se descarga y se procesa con Mimikatz exactamente igual.

---

### Vía 2 — RCE como SYSTEM mediante herencia de token

`SeDebugPrivilege` también permite escalar a SYSTEM sin tocar LSASS, mediante la **herencia del token de un proceso padre privilegiado**. La técnica consiste en lanzar un proceso hijo que hereda el token de un proceso que ya corre como SYSTEM (como `winlogon.exe` o `lsass.exe`), obteniendo así una shell con sus privilegios.

El flujo con el script **psgetsys.ps1** 📄 [psgetsys.ps1 — decoder-it](https://raw.githubusercontent.com/decoder-it/psgetsystem/master/psgetsys.ps1) es:

**Paso 1 — Abrir una consola PowerShell elevada y listar procesos SYSTEM**
```powershell
tasklist
```

Buscar procesos que corran como SYSTEM. Un objetivo clásico y estable es `winlogon.exe`. También se puede usar `Get-Process` 📄 [Get-Process — Microsoft Docs](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process?view=powershell-7.2) para obtener el PID directamente:
```powershell
Get-Process winlogon
```

**Paso 2 — Cargar el script y ejecutarlo pasando el PID objetivo**
```powershell
Import-Module .\psgetsys.ps1
[MyProcess]::CreateProcessFromParent(<PID_SYSTEM>, <comando>, "")
```

Por ejemplo, para obtener una shell como SYSTEM:
```powershell
[MyProcess]::CreateProcessFromParent(612, "cmd.exe", "")
```

El tercer argumento vacío `""` es obligatorio para que el PoC funcione correctamente. Si no se tiene sesión interactiva (shell reversa, web shell, command injection), el comando puede modificarse para lanzar una reverse shell o añadir un usuario administrador en lugar de abrir `cmd.exe`.

Para otros PoCs alternativos que también explotan `SeDebugPrivilege` para obtener una shell SYSTEM: 📄 [SeDebugPrivilegePoC — PrivFu](https://github.com/daem0nc0re/PrivFu/tree/main/PrivilegedOperations/SeDebugPrivilegePoC)

---

## 🛠️ Comandos / Herramientas

> [!tip] Verificar si SeDebugPrivilege está asignado al usuario actual
> ```cmd
> whoami /priv
> ```
> `/priv` — Lista todos los privilegios asignados. SeDebugPrivilege aparecerá en estado Disabled aunque sea explotable.

> [!tip] Volcar la memoria de LSASS con ProcDump
> ```cmd
> procdump.exe -accepteula -ma lsass.exe lsass.dmp
> ```
> `-accepteula` — Acepta automáticamente el EULA de Sysinternals.
> `-ma` — Crea un volcado completo de memoria del proceso (full dump).
> `lsass.exe` — Proceso objetivo. Contiene las credenciales de usuarios autenticados.
> `lsass.dmp` — Nombre del archivo de salida con el volcado.

> [!tip] Procesar el volcado de LSASS con Mimikatz
> ```cmd
> mimikatz.exe
> log
> sekurlsa::minidump lsass.dmp
> sekurlsa::logonpasswords
> ```
> `log` — Activa el logging de toda la sesión a un archivo `.txt`. Recomendable ejecutarlo siempre antes de cualquier otro comando.
> `sekurlsa::minidump lsass.dmp` — Carga el volcado de memoria en lugar de leer directamente de la memoria del sistema.
> `sekurlsa::logonpasswords` — Extrae hashes NTLM, tickets Kerberos y contraseñas en texto claro de todas las sesiones presentes en el volcado.

> [!tip] Obtener el PID de un proceso SYSTEM con PowerShell
> ```powershell
> Get-Process winlogon
> ```
> `winlogon` — Proceso objetivo que corre como SYSTEM. Se puede sustituir por `lsass` u otros procesos SYSTEM estables.

> [!tip] Escalar a SYSTEM heredando el token de un proceso padre con psgetsys.ps1
> ```powershell
> Import-Module .\psgetsys.ps1
> [MyProcess]::CreateProcessFromParent(612, "cmd.exe", "")
> ```
> `612` — PID del proceso SYSTEM objetivo (en este caso winlogon.exe). Obtener con `tasklist` o `Get-Process`.
> `"cmd.exe"` — Comando a ejecutar en el contexto SYSTEM. Puede cambiarse por una reverse shell u otro payload.
> `""` — Tercer argumento obligatorio para el correcto funcionamiento del PoC.

## 🔗 Relaciones / Contexto

`SeDebugPrivilege` es uno de los privilegios más peligrosos que se pueden encontrar en un usuario no administrador. A diferencia de `SeImpersonatePrivilege`, que requiere que un proceso privilegiado se conecte al nuestro, aquí el atacante toma la iniciativa directamente sobre cualquier proceso del sistema. La vía del volcado de LSASS conecta directamente con técnicas de movimiento lateral mediante pass-the-hash, mientras que la vía de herencia de token es especialmente útil cuando LSASS no tiene credenciales útiles en memoria o cuando se necesita persistencia como SYSTEM sin depender de sesiones activas de otros usuarios.