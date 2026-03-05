tags:
____
# 🗂️ Herramientas de Enumeración para Privilege Escalation en Windows

## 🧠 Concepto clave
Visión general de las principales herramientas usadas para enumerar vectores de escalada de privilegios en Windows, junto con consideraciones prácticas sobre su uso en entornos reales de pentesting.

## 📌 Puntos importantes

### Herramientas disponibles

El ecosistema de herramientas para privesc en Windows es amplio. Las más relevantes son:

| Herramienta | Tipo | Descripción |
|---|---|---|
| Seatbelt | C# | Amplia variedad de comprobaciones locales de privesc |
| winPEAS | Script | Busca posibles rutas de escalada en hosts Windows |
| PowerUp | PowerShell | Encuentra y explota misconfiguraciones de privesc |
| SharpUp | C# | Versión C# de PowerUp |
| JAWS | PowerShell 2.0 | Enumeración de vectores de privesc |
| SessionGopher | PowerShell | Extrae sesiones guardadas de PuTTY, WinSCP, FileZilla, RDP... |
| Watson | .NET | Enumera KBs ausentes y sugiere exploits aplicables |
| LaZagne | Binario | Recupera contraseñas de navegadores, Git, Wi-Fi, email y más |
| WES-NG | Python | Cruza el output de `systeminfo` con una BD de vulnerabilidades y exploits |
| Sysinternals Suite | Suite | Se usan especialmente AccessChk, PipeList y PsService |

### Consideraciones prácticas

A la hora de subir herramientas al sistema objetivo, `C:\Windows\Temp` es siempre la opción más segura, ya que el grupo `BUILTIN\Users` tiene permisos de escritura garantizados. Siempre que sea posible, se recomienda **compilar las herramientas desde el código fuente** en lugar de usar binarios precompilados, especialmente en entornos de cliente real.

Estas herramientas son potentes, pero tienen un lado negativo: la mayoría son **ampliamente conocidas y detectadas por soluciones AV/EDR**. Un ejemplo claro es LaZagne, que en el momento de redacción de los apuntes era flaggeado por 47 de 70 motores en VirusTotal. En engagements evasivos, será necesario aplicar técnicas de ofuscación como eliminar comentarios, renombrar funciones o cifrar el ejecutable.

### El problema de depender solo de herramientas

Las herramientas automatizan y agilizan la enumeración, pero pueden generar **falsos positivos y negativos**. Herramientas como winPEAS devuelven tal volumen de información que identificar lo realmente relevante puede ser difícil. Por eso es imprescindible conocer las técnicas de enumeración y explotación de forma manual, para poder:

- Operar en entornos **air-gapped** o sin posibilidad de cargar herramientas externas.
- Detectar lo que una herramienta pasa por alto.
- Explicar al cliente con precisión qué se está haciendo en cada paso del proceso.

## 🔗 Relaciones / Contexto

Aunque estas herramientas están orientadas a pentesting, también son muy útiles para **sysadmins** que quieran revisar la postura de seguridad de sus sistemas, analizar el impacto de cambios, o auditar una imagen nueva antes de desplegarla en producción.
# 🗂️ Situational Awareness — Enumeración de Red y Protecciones en Windows

## 🧠 Concepto clave
Antes de intentar escalar privilegios en un sistema Windows, es fundamental orientarse: conocer la red en la que se está y las protecciones activas. Esta información determina qué técnicas y herramientas son viables y qué caminos existen para movimiento lateral.

## 📌 Puntos importantes

### Información de red

Lo primero es entender en qué red estamos y si el host tiene acceso a otras. Un host **dual-homed** pertenece a dos o más redes simultáneamente mediante múltiples interfaces físicas o virtuales — comprometer uno de estos hosts puede abrir el acceso a segmentos de red previamente inaccesibles.

Los datos más valiosos a recopilar son:

| Qué buscar | Por qué importa |
|---|---|
| Interfaces y direcciones IP | Identificar si el host es dual-homed |
| Tabla de routing | Ver qué redes son alcanzables desde el host |
| Caché ARP | Ver con qué hosts ha comunicado recientemente (posibles targets para lateral movement) |
| Servidores DNS | Pueden revelar si el host está en un entorno Active Directory |

La caché ARP es especialmente interesante: puede indicar qué hosts están activos y a cuáles se conectan los administradores vía RDP o WinRM, lo que facilita el movimiento lateral tras obtener credenciales.

### Enumeración de protecciones

Los entornos modernos suelen tener soluciones antivirus o EDR activas que pueden interferir directamente con la enumeración y la escalada de privilegios. Conocer qué protecciones están en juego permite adaptar las herramientas antes de usarlas, evitando detecciones innecesarias.

Dos elementos clave a revisar:

**Windows Defender** — Comprobar si está activo y qué módulos tiene habilitados (protección en tiempo real, monitorización de comportamiento, etc.). Un Defender con `RealTimeProtectionEnabled: False` es mucho más permisivo.

**AppLocker** — Solución de Microsoft para controlar qué binarios y tipos de archivo pueden ejecutar los usuarios. Puede bloquear herramientas comunes como `cmd.exe`, `powershell.exe` o scripts. Es importante enumerarlo para saber si será necesario aplicar algún bypass antes de ejecutar herramientas o técnicas de privesc.

Algunos EDR también bloquean o generan alertas ante el uso de binarios legítimos del sistema como `net.exe` o `tasklist.exe`, por lo que entender el entorno con antelación puede marcar la diferencia entre pasar desapercibido o ser detectado al instante.

## 🛠️ Comandos / Herramientas

> [!tip] Obtener información completa de interfaces de red, IPs y DNS
> ```cmd
> ipconfig /all
> ```
> `/all` — Muestra información detallada de todas las interfaces: IP, máscara, gateway, servidores DNS, MAC address y estado DHCP.

> [!tip] Ver la caché ARP — hosts con los que se ha comunicado recientemente
> ```cmd
> arp -a
> ```
> `-a` — Lista todas las entradas ARP conocidas por interfaz, con su IP, MAC y si son dinámicas o estáticas.

> [!tip] Ver la tabla de routing — redes alcanzables desde el host
> ```cmd
> route print
> ```
> `print` — Muestra las rutas IPv4 e IPv6 activas y persistentes, indicando qué redes son accesibles y a través de qué gateway.

> [!tip] Comprobar el estado de Windows Defender
> ```powershell
> Get-MpComputerStatus
> ```
> `Get-MpComputerStatus` — Devuelve el estado de todos los módulos de Defender: antivirus, antispyware, protección en tiempo real, monitorización de comportamiento, etc.

> [!tip] Enumerar las reglas activas de AppLocker
> ```powershell
> Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
> ```
> `-Effective` — Obtiene la política AppLocker actualmente en vigor (combinación de local y de dominio).
> `select -ExpandProperty RuleCollections` — Expande y muestra todas las reglas definidas con sus condiciones y acciones.

> [!tip] Testear si AppLocker bloquea un binario concreto
> ```powershell
> Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone
> ```
> `-Local` — Usa la política AppLocker local como base del test.
> `-path` — Especifica el binario a comprobar.
> `-User` — Define el usuario o grupo contra el que se evalúa la política.

## 🔗 Relaciones / Contexto

Esta fase de reconocimiento precede directamente a la explotación. La información de red puede revelar rutas de movimiento lateral incluso antes de escalar privilegios en el sistema actual. La enumeración de protecciones, por su parte, es imprescindible para decidir qué herramientas usar, si necesitan ser modificadas o compiladas de nuevo, y si será necesario aplicar técnicas de bypass de AppLocker o evasión de AV/EDR.

# 🗂️ Enumeración Manual de Sistemas Windows — Sistema, Usuarios y Procesos

## 🧠 Concepto clave
Tras obtener una shell con bajos privilegios en un sistema Windows, el siguiente paso es enumerar el entorno para identificar vectores de escalada de privilegios. Esta sección cubre los datos clave a recopilar manualmente: información del sistema, procesos, variables de entorno, parches, software instalado, usuarios y grupos.

## 📌 Puntos importantes

### Objetivos de escalada de privilegios

Antes de enumerar, conviene tener claro a qué se puede aspirar escalar. No siempre el objetivo es SYSTEM — a veces basta con llegar a un usuario que tenga acceso a información sensible o pueda moverse lateralmente:

| Cuenta objetivo | Por qué es relevante |
|---|---|
| `NT AUTHORITY\SYSTEM` | Máximo privilegio del sistema. Ejecuta la mayoría de servicios de Windows y tiene más permisos que cualquier administrador local |
| Administrador local built-in | Muchas organizaciones no lo deshabilitan y se reutiliza en múltiples máquinas del entorno |
| Miembro del grupo local Administrators | Tiene exactamente los mismos privilegios que el administrador built-in |
| Usuario de dominio en Administrators local | Un usuario estándar de AD que, localmente, tiene control total sobre la máquina |
| Domain Admin en Administrators local | El objetivo más valioso: control total sobre el entorno Active Directory |

La enumeración es absolutamente la clave de la escalada de privilegios. Aunque existen herramientas que automatizan este proceso, es esencial saber hacerlo de forma manual, especialmente en entornos donde no se pueden cargar herramientas externas por restricciones de red, falta de acceso a internet o soluciones EDR activas.

---

### Información del sistema

#### Tasklist

Lo primero que interesa saber es qué está corriendo en el sistema. `tasklist /svc` muestra todos los procesos activos junto al servicio de Windows que tienen asociado. La clave aquí es **aprender a reconocer los procesos estándar del SO** — smss.exe (Session Manager), csrss.exe (Client Server Runtime), lsass.exe (Local Security Authority), svchost.exe (Service Host) — para poder detectar rápidamente lo que se sale de la norma.

Un servidor FileZilla corriendo en la máquina, por ejemplo, es inmediatamente interesante: podría tener acceso anónimo FTP habilitado, credenciales almacenadas recuperables con LaZagne, o una versión vulnerable con exploit público disponible. De forma similar, ver `MsMpEng.exe` (Windows Defender) en la lista nos confirma que hay protecciones activas que habrá que tener en cuenta.

#### Variables de entorno

`set` lista todas las variables de entorno del usuario actual y es una fuente de información que se pasa por alto con demasiada frecuencia. Tres variables merecen atención especial:

- **PATH** — Windows busca los ejecutables de izquierda a derecha en esta variable. Si un administrador añadió una carpeta personalizada (por ejemplo, para Python o Java) y esa carpeta es **escribible por nuestro usuario**, se puede realizar DLL hijacking o sustituir binarios. El riesgo es mayor si la carpeta personalizada aparece a la **izquierda** de `C:\Windows\System32`.
- **HOMEDRIVE / HOMEPATH** — En entornos empresariales, el home drive suele apuntar a un **file share de red**. Navegar por ese share puede revelar directorios de TI con inventarios, scripts o incluso hojas de cálculo con contraseñas.
- **USERPROFILE** — Si se coloca un archivo malicioso en `USERPROFILE\AppData\Microsoft\Windows\Start Menu\Programs\Startup`, se ejecutará automáticamente al iniciar sesión. Si el usuario tiene **Roaming Profile**, ese archivo viajará con él a cualquier máquina donde inicie sesión.

#### Información detallada de configuración

`systeminfo` es uno de los comandos más valiosos en esta fase. De un solo golpe revela la versión exacta del SO, el nivel de build, los hotfixes instalados, cuándo arrancó el sistema por última vez y si se está corriendo dentro de una VM. 

Un sistema que no ha reiniciado en más de seis meses probablemente tampoco está siendo parcheado regularmente — lo que lo convierte en un candidato ideal para exploits públicos. Los KBs que aparecen bajo "Hotfix(s)" se pueden buscar en Google para determinar hasta qué fecha está parcheado el sistema y qué CVEs siguen sin aplicarse.

#### Parches y actualizaciones

Si `systeminfo` no muestra los hotfixes (es posible ocultarlos a usuarios no administradores), `wmic qfe` los lista directamente consultando WMI. `Get-HotFix` en PowerShell ofrece la misma información en un formato más limpio. En ambos casos, el objetivo es el mismo: identificar parches que faltan y buscar exploits asociados. Hay que tener cuidado con los exploits de kernel de Windows — pueden causar inestabilidad o un crash total del sistema, especialmente en producción.

#### Programas instalados

`wmic product get name` o su equivalente en PowerShell `Get-WmiObject -Class Win32_Product` listan todo el software instalado en el sistema. Esta información es valiosa en dos sentidos: por un lado, pueden existir **versiones vulnerables** de aplicaciones con exploits públicos conocidos; por otro, herramientas como FileZilla, PuTTY o clientes de base de datos suelen almacenar credenciales que LaZagne puede recuperar directamente.

#### Procesos en ejecución y puertos

`netstat -ano` muestra todas las conexiones TCP/UDP activas y los puertos en escucha, incluyendo el PID del proceso responsable de cada uno. Lo más interesante aquí son los **servicios que solo escuchan en localhost** (dirección `0.0.0.0` o `127.0.0.1`) — servicios que no son accesibles desde fuera pero sí desde la propia máquina. Si uno de esos servicios es vulnerable, puede explotarse localmente para escalar privilegios.

---

### Información de usuarios y grupos

Los usuarios son frecuentemente el eslabón más débil, incluso en sistemas bien parcheados. Un directorio de usuario accesible que contenga un `logins.xlsx` puede ser un vector de compromiso trivial.

#### Usuarios conectados al sistema

`query user` muestra qué usuarios tienen sesión activa, qué tipo de sesión tienen (RDP, consola...) y cuánto tiempo llevan inactivos. Esta información es tácticamente importante: en un engagement evasivo, operar en una máquina donde hay un usuario activo trabajando aumenta enormemente el riesgo de ser detectado.

#### Usuario actual y sus privilegios

Lo primero antes de cualquier otra acción es saber bajo qué contexto se está operando. `echo %USERNAME%` confirma el usuario actual — en ocasiones se puede haber obtenido acceso directamente como SYSTEM sin saberlo. Si se accede como cuenta de servicio, es probable que tenga asignado `SeImpersonatePrivilege`, un privilege que permite hacerse pasar por otros usuarios y que herramientas como **Juicy Potato** explotan de forma muy efectiva para escalar a SYSTEM.

`whoami /priv` lista todos los privileges del usuario actual y su estado (Enabled/Disabled). Algunos de estos privileges son vectores de escalada directa y se cubrirán en detalle más adelante en el módulo.

#### Grupos del usuario actual

`whoami /groups` muestra todos los grupos a los que pertenece el usuario, con sus SIDs y atributos. La pertenencia a ciertos grupos como `Backup Operators`, `Remote Management Users` o `Event Log Readers` puede otorgar capacidades que, bien aprovechadas, permiten escalar privilegios tanto en local como en el entorno de Active Directory.

#### Todos los usuarios y grupos del sistema

`net user` lista todas las cuentas del sistema. Si se está operando como `bob` y existe un `bob_adm` en el grupo de administradores locales, vale la pena comprobar reutilización de credenciales. Los directorios personales de otros usuarios también pueden contener scripts con contraseñas en texto claro o claves SSH.

`net localgroup` lista todos los grupos locales. La presencia de grupos no estándar puede indicar el propósito del host o revelar misconfiguraciones — por ejemplo, todos los usuarios del dominio dentro del grupo Remote Desktop Users. Con `net localgroup administrators` se pueden ver exactamente quiénes son administradores locales, y a veces la descripción del grupo puede contener información sensible dejada por un administrador descuidado.

#### Política de contraseñas

`net accounts` muestra la política de contraseñas del sistema: longitud mínima, historial, caducidad y umbral de bloqueo de cuenta. Una política sin longitud mínima, sin historial y sin bloqueo de cuenta es una invitación abierta a ataques de fuerza bruta o password spraying sin riesgo de bloquear cuentas.

---

## 🛠️ Comandos / Herramientas

> [!tip] Ver procesos en ejecución y sus servicios asociados
> ```cmd
> tasklist /svc
> ```
> `/svc` — Muestra el nombre del servicio de Windows vinculado a cada proceso. Permite identificar servicios de terceros potencialmente vulnerables o mal configurados.

> [!tip] Ver todas las variables de entorno del usuario actual
> ```cmd
> set
> ```
> `set` — Lista todas las variables de entorno activas. Prestar especial atención a PATH, HOMEDRIVE y USERPROFILE.

> [!tip] Información detallada del sistema operativo y parches
> ```cmd
> systeminfo
> ```
> `systeminfo` — Muestra versión del SO, build, hotfixes instalados, fecha de último arranque, NICs y si el host es una VM.

> [!tip] Listar hotfixes instalados vía WMI
> ```cmd
> wmic qfe
> ```
> `qfe` — Quick Fix Engineering. Lista todos los parches instalados con su KB, fecha de instalación e instalador.

> [!tip] Listar hotfixes instalados vía PowerShell
> ```powershell
> Get-HotFix | ft -AutoSize
> ```
> `ft -AutoSize` — Formatea la salida en tabla ajustando el ancho de columna automáticamente para mejor legibilidad.

> [!tip] Listar programas instalados vía WMI
> ```cmd
> wmic product get name
> ```
> `product get name` — Devuelve el nombre de todos los programas instalados en el sistema.

> [!tip] Listar programas instalados con versión vía PowerShell
> ```powershell
> Get-WmiObject -Class Win32_Product | select Name, Version
> ```
> `-Class Win32_Product` — Consulta la clase WMI que contiene el inventario de software instalado.
> `select Name, Version` — Filtra la salida mostrando solo nombre y versión de cada programa.

> [!tip] Ver conexiones activas y puertos en escucha
> ```cmd
> netstat -ano
> ```
> `-a` — Muestra todas las conexiones y puertos en escucha.
> `-n` — Muestra IPs y puertos en formato numérico en lugar de resolver nombres.
> `-o` — Incluye el PID del proceso responsable de cada conexión.

> [!tip] Ver usuarios con sesión activa en el sistema
> ```cmd
> query user
> ```
> `query user` — Lista sesiones activas con nombre de usuario, tipo de sesión (RDP, consola...), estado y tiempo de inactividad.

> [!tip] Ver el usuario actual
> ```cmd
> echo %USERNAME%
> ```

> [!tip] Ver los privilegios del usuario actual
> ```cmd
> whoami /priv
> ```
> `/priv` — Lista todos los privileges asignados al usuario y su estado (Enabled/Disabled).

> [!tip] Ver los grupos del usuario actual
> ```cmd
> whoami /groups
> ```
> `/groups` — Muestra todos los grupos a los que pertenece el usuario, con sus SIDs y atributos de membresía.

> [!tip] Listar todas las cuentas de usuario del sistema
> ```cmd
> net user
> ```

> [!tip] Listar todos los grupos locales del sistema
> ```cmd
> net localgroup
> ```

> [!tip] Ver los miembros de un grupo específico
> ```cmd
> net localgroup administrators
> ```

> [!tip] Ver la política de contraseñas y configuración de cuentas
> ```cmd
> net accounts
> ```
> `net accounts` — Muestra la política de contraseñas completa: longitud mínima, historial, caducidad y umbral de bloqueo de cuenta.

## 🔗 Relaciones / Contexto

Toda la información recopilada aquí alimenta directamente las decisiones sobre qué vector explotar a continuación. Un SO sin parchear lleva a buscar CVEs públicos; un `SeImpersonatePrivilege` activo lleva directamente a Juicy Potato; un PATH con directorios escribibles abre la puerta al DLL hijacking; credenciales reutilizadas entre usuarios permiten movimiento lateral. Esta fase de enumeración no es un trámite — es la base sobre la que se construye toda la estrategia de escalada.

# 🗂️ Comunicación con Procesos — Network Services y Named Pipes

## 🧠 Concepto clave
Los procesos en ejecución son uno de los mejores vectores de escalada de privilegios. Aunque un proceso no corra como administrador, puede abrir vías indirectas hacia privilegios elevados. Los dos mecanismos principales de comunicación entre procesos que interesan aquí son los **network sockets** y los **named pipes**.

## 📌 Puntos importantes

### Access Tokens

En Windows, cada proceso o hilo tiene asociado un **access token** que describe su contexto de seguridad: qué usuario lo está ejecutando y qué privilegios tiene. Cuando un usuario se autentica, el sistema verifica sus credenciales contra la base de datos de seguridad y le asigna un token. Cada vez que ese usuario interactúa con un proceso, se presenta una copia del token para determinar su nivel de acceso. Esto es fundamental porque si se consigue ejecutar código dentro de un proceso con un token privilegiado, se heredan esos privilegios.

---

### Servicios de red — Enumeración de conexiones activas

La forma más común de interactuar con procesos es a través de sockets de red (HTTP, SMB, DNS, etc.). `netstat -ano` muestra todas las conexiones TCP/UDP activas y los puertos en escucha, junto al PID responsable de cada uno.

Lo más valioso que hay que buscar son **servicios escuchando en direcciones de loopback** (`127.0.0.1` y `::1`) que no son accesibles desde fuera de la máquina. Estos servicios suelen estar menos protegidos porque sus desarrolladores asumen que al no ser accesibles por red, no representan un riesgo — un error de concepto que frecuentemente abre vías de escalada local.

En el ejemplo de los apuntes, el puerto `14147` escuchando solo en `127.0.0.1` corresponde a la **interfaz administrativa de FileZilla**. Conectarse a ese puerto puede permitir extraer contraseñas FTP almacenadas e incluso crear un share FTP apuntando a `C:\` bajo el usuario que ejecuta FileZilla, que podría ser Administrador.

#### Otros ejemplos de servicios locales explotables

Dos casos especialmente relevantes en entornos reales:

- **Splunk Universal Forwarder** — Se instala en endpoints para enviar logs a Splunk. En su configuración por defecto no requería autenticación, permitiendo a cualquiera desplegar aplicaciones y ejecutar código. Para empeorar las cosas, corría como `SYSTEM` por defecto. Herramienta de referencia: **SplunkWhisperer2**.

- **Erlang Port (25672)** — Erlang es un lenguaje diseñado para computación distribuida que expone un puerto de red para que otros nodos se unan al cluster. La autenticación se basa en una **cookie** — y muchas aplicaciones que usan Erlang emplean cookies débiles (RabbitMQ usa `rabbit` por defecto) o las almacenan en ficheros de configuración mal protegidos. Aplicaciones afectadas: SolarWinds, RabbitMQ, CouchDB.

---

### Named Pipes

Los **named pipes** son el otro mecanismo de comunicación entre procesos en Windows. Funcionan como archivos almacenados en memoria que se vacían tras ser leídos. A diferencia de los sockets de red, la comunicación ocurre localmente entre procesos del mismo sistema.

Windows implementa los named pipes con un modelo cliente-servidor: el proceso que crea el pipe es el servidor, y el que se conecta a él es el cliente. Pueden funcionar en modo **half-duplex** (solo el cliente escribe) o **duplex** (comunicación bidireccional). Cada conexión activa a un named pipe genera una nueva instancia del pipe, todas compartiendo el mismo nombre pero con buffers de datos independientes.

Un ejemplo muy conocido del uso de named pipes es **Cobalt Strike**, que los utiliza para cada comando ejecutado por el beacon. El flujo es: el beacon crea un pipe (ej. `\\.\pipe\msagent_12`), lanza un nuevo proceso que ejecuta el comando y redirige su output al pipe, y el servidor lee ese output. La ventaja es que si el proceso hijo es detectado o crashea, el beacon principal no se ve afectado. Los operadores de Cobalt Strike frecuentemente renombran sus pipes para hacerse pasar por aplicaciones legítimas — un caso clásico es usar `mojo` para imitar Chrome. Un pipe con nombre `mojo` en un sistema sin Chrome instalado es una señal clara de actividad sospechosa.

#### Enumeración de Named Pipes

Para listar los named pipes activos se pueden usar dos métodos:

- **PipeList** (Sysinternals) — Muestra nombre del pipe, instancias activas y máximo de instancias permitidas.
- **PowerShell con `gci`** — Lista el contenido de `\\.\pipe\` como si fuera un directorio.

Una vez identificados los pipes, el siguiente paso es revisar sus permisos mediante la **DACL** (Discretionary Access List) con **AccessChk** (también de Sysinternals). La DACL determina quién puede leer, escribir, modificar o ejecutar sobre ese recurso. Un pipe con permisos de escritura para grupos amplios como `Everyone` o `Authenticated Users` es un vector de escalada potencial.

#### Ejemplo de ataque — WindscribeService Named Pipe

Un caso real documentado es el de **WindscribeService**: su named pipe tenía `FILE_ALL_ACCESS` (todos los permisos posibles) asignado al grupo `Everyone`, lo que significa que cualquier usuario autenticado podía escribir en él. Al ser WindscribeService un servicio que corre con privilegios elevados, esta misconfiguration permitía escalar directamente a SYSTEM.

El proceso para encontrar este tipo de vulnerabilidades es buscar todos los named pipes con permisos de escritura usando `accesschk.exe -w \pipe\* -v` y revisar cuáles tienen permisos excesivos.

---

## 🛠️ Comandos / Herramientas

> [!tip] Ver conexiones activas y puertos en escucha con su PID
> ```cmd
> netstat -ano
> ```
> `-a` — Muestra todas las conexiones y puertos en escucha.
> `-n` — Muestra IPs y puertos en formato numérico.
> `-o` — Incluye el PID del proceso responsable de cada conexión.

> [!tip] Listar named pipes activos con PipeList (Sysinternals)
> ```cmd
> pipelist.exe /accepteula
> ```
> `/accepteula` — Acepta automáticamente el EULA de Sysinternals para evitar el popup interactivo.

> [!tip] Listar named pipes activos con PowerShell
> ```powershell
> gci \\.\pipe\
> ```
> `gci` — Alias de `Get-ChildItem`. Lista el contenido de `\\.\pipe\` mostrando todos los named pipes activos como si fueran archivos en un directorio.

> [!tip] Revisar los permisos DACL de un named pipe específico
> ```cmd
> accesschk.exe /accepteula \\.\Pipe\lsass -v
> ```
> `/accepteula` — Acepta el EULA automáticamente.
> `\\.\Pipe\lsass` — Named pipe a analizar. Se puede sustituir por cualquier otro pipe.
> `-v` — Modo verbose, muestra los permisos detallados de cada entrada en la DACL.

> [!tip] Buscar todos los named pipes con permisos de escritura
> ```cmd
> accesschk.exe -w \pipe\* -v
> ```
> `-w` — Filtra mostrando solo los objetos sobre los que el usuario actual tiene permisos de escritura.
> `\pipe\*` — Aplica la búsqueda a todos los named pipes del sistema.
> `-v` — Muestra información detallada de permisos.

> [!tip] Revisar permisos de un named pipe específico (ej. WindscribeService)
> ```cmd
> accesschk.exe -accepteula -w \pipe\WindscribeService -v
> ```

## 🔗 Relaciones / Contexto

Este tema conecta directamente con los **access tokens** y el privilege `SeImpersonatePrivilege`. Cuando se encuentra un proceso de red vulnerable (como Splunk o FileZilla) o un named pipe mal configurado, el objetivo es ejecutar código en el contexto de ese proceso para heredar su token — y si ese token pertenece a SYSTEM o a un administrador, la escalada está prácticamente hecha. Herramientas como Juicy Potato o Rogue Potato explotan exactamente esta mecánica.

# 🗂️ Privilegios en Windows — Visión General y Abuso para Escalada

## 🧠 Concepto clave
Los **privilegios** en Windows son derechos asignados a cuentas que permiten realizar operaciones específicas sobre el sistema local. Entender cómo funcionan, cómo se asignan y cuáles son abusables es fundamental para la escalada de privilegios, tanto en hosts aislados como en entornos Active Directory.

## 📌 Puntos importantes

### Privilegios vs. Derechos de acceso

Es importante no confundir estos dos conceptos:

- **Privilegios** — Derechos sobre operaciones del sistema: cargar drivers, hacer debug de procesos, apagar el sistema, etc.
- **Derechos de acceso** — Permisos sobre objetos concretos (archivos, carpetas, claves de registro): leer, escribir, ejecutar.

Los privilegios se almacenan en una base de datos y se asignan al usuario a través de su **access token** en el momento del login. Cada vez que el usuario intenta realizar una acción privilegiada, el sistema consulta ese token para verificar si tiene el privilegio necesario y si está habilitado. La mayoría de privilegios vienen **deshabilitados por defecto**, y algunos solo se activan al abrir una consola elevada (como administrador).

---

### El proceso de autorización de Windows

Todo objeto en Windows que puede ser autenticado — usuarios, cuentas de equipo, procesos, grupos — es un **security principal**, identificado de forma única por un **SID (Security Identifier)** que se le asigna al crearse y no cambia durante toda su vida.

Cuando un usuario intenta acceder a un objeto (por ejemplo, una carpeta en un share), el sistema compara su **access token** — que incluye su SID, los SIDs de sus grupos, y su lista de privilegios — contra las **ACEs (Access Control Entries)** de la **DACL** del objeto. En función de esa comparación, se concede o deniega el acceso. Todo este proceso ocurre de forma prácticamente instantánea. En escalada de privilegios, el objetivo es precisamente manipular o insertarse en este proceso de autorización.

---

### Grupos con privilegios elevados

Ciertos grupos de Windows otorgan a sus miembros privilegios muy potentes que pueden abusarse para escalar. Algunos de los más relevantes:

| Grupo | Riesgo / Abuso potencial |
|---|---|
| **Default Administrators** (Domain Admins, Enterprise Admins) | Grupos "super" con control total |
| **Server Operators** | Pueden modificar servicios, acceder a shares SMB y hacer backup de archivos |
| **Backup Operators** | Pueden iniciar sesión localmente en DCs, hacer shadow copies de SAM/NTDS, leer el registro remotamente y acceder al sistema de archivos del DC vía SMB — deben considerarse equivalentes a Domain Admins |
| **Print Operators** | Pueden iniciar sesión local en DCs y engañar a Windows para cargar un driver malicioso |
| **Hyper-V Administrators** | Si hay DCs virtualizados, estos admins deben considerarse Domain Admins |
| **Account Operators** | Pueden modificar cuentas y grupos no protegidos del dominio |
| **Remote Desktop Users** | Sin permisos útiles por defecto, pero frecuentemente se les conceden derechos adicionales de acceso RDP |
| **Remote Management Users** | Pueden conectarse a DCs mediante PSRemoting |
| **Group Policy Creator Owners** | Pueden crear GPOs, aunque necesitan permisos adicionales para vincularlas |
| **Schema Admins** | Pueden modificar el esquema de Active Directory y añadir backdoors a objetos por defecto |
| **DNS Admins** | Pueden cargar una DLL maliciosa en un DC y usarla como mecanismo de persistencia, o crear registros WPAD para ataques de red |

---

### User Rights Assignment — Privilegios clave

Además de la pertenencia a grupos, los privilegios pueden asignarse directamente a cuentas mediante Group Policy. Los más relevantes desde el punto de vista ofensivo son:

| Privilegio | Descripción y abuso potencial |
|---|---|
| `SeBackupPrivilege` | Permite saltarse permisos de archivos con el pretexto de hacer backup — puede usarse para leer cualquier archivo del sistema |
| `SeRestorePrivilege` | Permite saltarse permisos al restaurar archivos — puede usarse para sobreescribir archivos críticos del sistema |
| `SeDebugPrivilege` | Permite hacer attach a cualquier proceso, incluso sin ser su propietario — permite inyectar en procesos SYSTEM como lsass |
| `SeImpersonatePrivilege` | Permite suplantar a otro usuario tras autenticarse — base de ataques como Juicy Potato, Rogue Potato o Lonely Potato para escalar a SYSTEM |
| `SeLoadDriverPrivilege` | Permite cargar y descargar drivers — un driver malicioso corre en modo kernel con privilegios máximos |
| `SeTakeOwnershipPrivilege` | Permite tomar posesión de cualquier objeto del sistema (archivos, claves de registro, procesos...) |
| `SeSecurityPrivilege` | Permite gestionar los logs de auditoría y seguridad — puede usarse para borrar rastros |
| `SeTcbPrivilege` | Permite asumir la identidad de cualquier usuario y acceder a sus recursos |

---

### Estados de los privilegios y UAC

Un privilegio puede estar asignado a un usuario pero aparecer en estado **Disabled**. Esto significa que existe en el access token pero no está activo — no puede usarse hasta que se habilite explícitamente. Windows no ofrece un cmdlet nativo para habilitar privilegios, por lo que se necesitan scripts específicos para manipular el token:

- 📄 [Enable-Privilege.ps1 — PoshPrivilege (PowerShell Gallery)](https://www.powershellgallery.com/packages/PoshPrivilege/0.3.0.0/Content/Scripts%5CEnable-Privilege.ps1) — Script para habilitar privilegios específicos desde PowerShell.
- 📄 [Adjusting Token Privileges in PowerShell — Lee Holmes](https://www.leeholmes.com/adjusting-token-privileges-in-powershell/) — Explicación detallada de cómo manipular los privilegios del token directamente desde PowerShell.

El **UAC (User Account Control)**, introducido en Windows Vista, añade otra capa: incluso los administradores locales operan por defecto con privilegios reducidos y necesitan elevar explícitamente para acceder a su token completo. La diferencia entre una consola no elevada y una elevada para un administrador local es drástica — en la no elevada, la mayoría de privilegios poderosos simplemente no aparecen.

Un usuario estándar tiene un conjunto de privilegios mínimo:
- `SeChangeNotifyPrivilege` — Habilitado por defecto
- `SeIncreaseWorkingSetPrivilege` — Deshabilitado

Mientras que un miembro de **Backup Operators**, aunque restringido por UAC, ya tiene `SeShutdownPrivilege`, lo que le permite apagar un Domain Controller si consigue iniciar sesión localmente — lo que causaría una interrupción masiva de servicios.

---

### Detección (perspectiva defensiva)

Desde el lado defensivo, el **evento 4672** de Windows ("Special privileges assigned to new logon") es clave para detectar el abuso de privilegios sensibles. Se genera cada vez que se asignan privilegios especiales a una nueva sesión de logon, y puede configurarse para alertar cuando ciertos privilegios aparecen en cuentas que normalmente no deberían tenerlos.

- 📄 [Evento 4672 — Documentación oficial de Microsoft](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4672) — Referencia completa del evento, campos que registra y cómo interpretarlo.
- 📄 [Windows Privilege Abuse: Auditing, Detection and Defense — Palantir](https://blog.palantir.com/windows-privilege-abuse-auditing-detection-and-defense-3078a403d74e) — Artículo en profundidad sobre cómo detectar y prevenir el abuso de privilegios en Windows, muy recomendado tanto para red team como blue team.

---

## 🛠️ Comandos / Herramientas

> [!tip] Ver todos los privilegios asignados al usuario actual
> ```cmd
> whoami /priv
> ```
> `/priv` — Lista todos los privilegios del usuario actual con su estado (Enabled/Disabled). Ejecutar en consola elevada para ver el listado completo de un administrador.

## 🔗 Relaciones / Contexto

`SeImpersonatePrivilege` es probablemente el privilegio más frecuentemente abusado en entornos reales — se asigna por defecto a cuentas de servicio y de red, y es la base de toda la familia de ataques Potato (Juicy, Rogue, Lonely). `SeDebugPrivilege` permite atacar directamente procesos como lsass para extraer credenciales. `SeBackupPrivilege` y `SeRestorePrivilege` permiten leer y sobreescribir cualquier archivo del sistema ignorando los ACLs. Estos privilegios se tratarán en profundidad en secciones posteriores del módulo.