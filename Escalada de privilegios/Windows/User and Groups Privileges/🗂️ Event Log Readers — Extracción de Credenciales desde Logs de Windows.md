tags:
___
## 🧠 Concepto clave
El grupo **Event Log Readers** permite a sus miembros leer los logs de eventos del sistema local sin necesitar privilegios administrativos. Si la organización tiene habilitada la auditoría de creación de procesos con sus líneas de comandos completas (evento 4688), un miembro de este grupo puede leer esos logs y encontrar credenciales que hayan sido pasadas como parámetros en comandos — algo más común de lo que parece en entornos reales.

| Concepto | Qué es | Riesgo |
|---|---|---|
| **Evento 4688** | Evento de Windows que registra cada nuevo proceso creado, incluyendo opcionalmente la línea de comandos completa 📄 [Audit Process Creation — Microsoft](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/audit-process-creation) — 📄 [Event 4688 — Microsoft](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4688) | Si se registran las líneas de comandos, cualquier credencial pasada como parámetro (ej: `/user:tim MyStr0ngP@ssword`) queda guardada en texto claro en el log |
| **Event Log Readers** | Grupo built-in de Windows cuyos miembros pueden leer logs de eventos del sistema local | Permite a un atacante con esta membresía leer el Security log y buscar credenciales expuestas en comandos auditados |

## 📌 Puntos importantes

### Contexto — Auditoría de procesos como vector de fuga de credenciales

Cuando una organización habilita la auditoría de creación de procesos y el registro de líneas de comandos, cada comando ejecutado en el sistema queda registrado en el Security log con su línea de comandos completa. Esto es una herramienta defensiva valiosa — permite a los defensores detectar comportamiento malicioso, identificar binarios no autorizados y enviar los datos a un SIEM como ElasticSearch para tener visibilidad centralizada. Muchos comandos típicos de reconocimiento post-acceso (`whoami`, `netstat`, `tasklist`, `ipconfig`, `systeminfo`) o de movimiento lateral (`wmic`, `reg`, `at`) pueden detectarse así, incluso sin un EDR empresarial.

Sin embargo, este mismo mecanismo crea un riesgo: **muchos comandos de Windows aceptan contraseñas como parámetros** — `net use`, `runas`, `wevtutil` entre otros — y si se ejecutan en un sistema con esta auditoría activa, las credenciales quedan registradas en texto claro en el Security log. Un miembro del grupo Event Log Readers puede aprovechar esto.

Un ejemplo real: durante un pentest contra una organización mediana con auditoría de procesos activa pero sin EDR, el equipo fue detectado cuando uno de sus miembros ejecutó `tasklist` desde la workstation de un empleado de finanzas — tras capturar y crackear sus credenciales con Responder. La organización detectó y contuvo el incidente gracias únicamente a esta configuración nativa de Windows.

### Acceso al Security log — Limitaciones importantes

Hay una distinción crítica que tener en cuenta:

- **`wevtutil`** — Un miembro del grupo Event Log Readers puede usarlo para leer el Security log directamente.
- **`Get-WinEvent`** — Requiere acceso de administrador **o** permisos modificados explícitamente en la clave de registro `HKLM\System\CurrentControlSet\Services\Eventlog\Security`. La membresía en Event Log Readers por sí sola **no es suficiente** para usar este cmdlet contra el Security log.

Además del Security log, el **PowerShell Operational log** puede contener información sensible o credenciales si el script block logging o el module logging están habilitados — y este log **sí es accesible para usuarios sin privilegios**. 📄 [About Logging Windows — Microsoft](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_logging_windows?view=powershell-7.1)

---

### Flujo de explotación — Búsqueda de credenciales en logs

**Paso 1 — Confirmar membresía en el grupo**

> [!tip] Listar miembros del grupo Event Log Readers
> ```cmd
> net localgroup "Event Log Readers"
> ```
> `"Event Log Readers"` — Nombre del grupo a consultar. Muestra todos los miembros actuales y confirma si el usuario actual pertenece a él.

**Paso 2 — Buscar credenciales en el Security log con wevtutil** 📄 [wevtutil — Microsoft Docs](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/wevtutil)

`wevtutil` es la herramienta nativa de Windows para consultar y gestionar logs de eventos. Se puede filtrar la salida buscando cadenas como `/user`, que aparecen frecuentemente en comandos que incluyen credenciales:

> [!tip] Buscar credenciales en el Security log filtrando por el patrón /user
> ```powershell
> wevtutil qe Security /rd:true /f:text | Select-String "/user"
> ```
> `qe Security` — Query Events del log de seguridad de Windows.
> `/rd:true` — Lee los eventos en orden inverso, mostrando los más recientes primero.
> `/f:text` — Formato de salida en texto plano, más fácil de filtrar con Select-String o findstr.
> `Select-String "/user"` — Filtra las líneas que contienen `/user`, patrón común en comandos con credenciales como `net use`.

> [!tip] Buscar credenciales en el Security log de un sistema remoto con credenciales alternativas
> ```cmd
> wevtutil qe Security /rd:true /f:text /r:share01 /u:julie.clay /p:Welcome1 | findstr "/user"
> ```
> `qe Security` — Query Events del log de seguridad.
> `/rd:true` — Eventos en orden inverso, más recientes primero.
> `/f:text` — Salida en texto plano.
> `/r:share01` — Sistema remoto contra el que ejecutar la consulta.
> `/u:julie.clay` — Usuario alternativo para autenticarse en el sistema remoto.
> `/p:Welcome1` — Contraseña del usuario alternativo.
> `findstr "/user"` — Equivalente a Select-String en cmd, filtra líneas con el patrón indicado.

**Paso 3 — Buscar credenciales con Get-WinEvent (requiere permisos adicionales)** 📄 [Get-WinEvent — Microsoft Docs](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-7.1)

> [!tip] Filtrar eventos de creación de proceso (4688) que contengan credenciales
> ```powershell
> Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}
> ```
> `-LogName security` — Especifica el log de seguridad de Windows como fuente de eventos.
> `$_.ID -eq 4688` — Filtra únicamente eventos de creación de proceso (New Process Created).
> `$_.Properties[8].Value` — Accede al campo de línea de comandos completa del evento 4688.
> `-like '*/user*'` — Filtra eventos cuya línea de comandos contenga el patrón `/user`, indicativo de credenciales embebidas.
> `Select-Object @{name='CommandLine'...}` — Presenta el resultado con un nombre de columna legible.

---

## 🔗 Relaciones / Contexto

Este vector es especialmente interesante porque **no requiere herramientas externas ni técnicas agresivas** — solo leer logs con permisos legítimos. Combinado con la auditoría de procesos habilitada por el propio equipo defensivo, puede convertirse paradójicamente en un vector de fuga de credenciales. Las credenciales encontradas en logs pueden usarse para movimiento lateral, acceso a shares, o escalar privilegios si pertenecen a cuentas con más derechos. Es también una buena razón para que los administradores eviten pasar contraseñas como parámetros en la línea de comandos, usando variables de entorno o gestores de credenciales en su lugar.