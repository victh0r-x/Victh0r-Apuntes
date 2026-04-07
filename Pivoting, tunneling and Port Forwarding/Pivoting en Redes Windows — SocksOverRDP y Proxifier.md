
## 🧠 Concepto clave

En engagements donde la infraestructura es completamente Windows y no hay acceso SSH disponible, las técnicas de pivoting habituales (chisel, socat, ssh -D) pueden no ser viables. **SocksOverRDP** resuelve este problema usando una característica nativa del protocolo RDP llamada **Dynamic Virtual Channels (DVC)** — canales de comunicación bidireccionales integrados en RDP que normalmente transportan datos de portapapeles, audio y dispositivos periféricos, pero que SocksOverRDP redirige para transportar tráfico SOCKS arbitrario.

La ventaja clave es que el tráfico de pivoting viaja encapsulado dentro de una sesión RDP legítima — algo que prácticamente ningún firewall corporativo bloquea porque RDP es un protocolo de administración estándar en entornos Windows.

| Herramienta | OS | Transporte | Requiere |
|---|---|---|---|
| chisel | Linux/Windows | HTTP/WebSocket | Binario externo |
| ssh -D | Linux/Windows | SSH | Cliente SSH |
| netsh portproxy | Windows | TCP nativo | Privilegios admin |
| **SocksOverRDP** | **Windows** | **RDP (DVC)** | **Sesión RDP activa** |

> [!info] **SocksOverRDP**
> Herramienta que aprovecha los Dynamic Virtual Channels del protocolo RDP para crear un proxy SOCKS dentro de una sesión RDP activa. Consta de dos componentes: `SocksOverRDP-Plugin.dll` — se registra en el host origen como plugin RDP y captura el tráfico SOCKS — y `SocksOverRDP-Server.exe` — corre en el host destino de la sesión RDP y actúa como el endpoint SOCKS que enruta el tráfico hacia la red interna.

> [!info] **Proxifier** 📄 [proxifier.com](https://www.proxifier.com)
> Equivalente Windows de proxychains. Intercepta las conexiones de red de cualquier aplicación del sistema y las redirige a través de un proxy SOCKS o HTTPS sin que la aplicación necesite soportar proxies de forma nativa. En este escenario actúa como el cliente que envía todo el tráfico al proxy SOCKS creado por SocksOverRDP.

---

## 📌 Escenario
```
[Atacante Windows — 10.129.x.x]
    │  RDP con SocksOverRDP Plugin
    │  Proxifier → 127.0.0.1:1080
    ▼
[Pivot Windows — 172.16.5.19]
    │  SocksOverRDP-Server.exe
    │  SOCKS proxy en 127.0.0.1:1080
    ▼
[Red interna — 172.16.6.0/24]
    └── Target 172.16.6.155
```

---

## 📌 Paso 1 — Descargar los binarios necesarios en el atacante

Antes de comenzar necesitamos dos herramientas en el host atacante Windows para luego transferirlas a los pivots:

**SocksOverRDP** — disponible en GitHub. Descargar `SocksOverRDPx64.zip` que contiene tanto el plugin DLL como el servidor ejecutable.

**Proxifier Portable** — buscar `ProxifierPE.zip` en la web oficial. La versión portable no requiere instalación — se ejecuta directamente desde cualquier ruta.

---

## 📌 Paso 2 — Transferir y registrar el plugin en el host atacante

Copiamos `SocksOverRDP-Plugin.dll` al host atacante Windows y lo registramos como plugin RDP usando `regsvr32.exe` — la herramienta nativa de Windows para registrar DLLs en el sistema:

> [!info] **regsvr32.exe**
> Herramienta nativa de Windows que registra y desregistra DLLs como componentes COM en el sistema. En este caso registra el plugin de SocksOverRDP para que el cliente RDP (`mstsc.exe`) lo cargue automáticamente en cada sesión RDP que se establezca.

> [!tip] Registrar el plugin SocksOverRDP en el host atacante
> ```cmd
> C:\Users\htb-student\Desktop\SocksOverRDP-x64> regsvr32.exe SocksOverRDP-Plugin.dll
> ```
> `regsvr32.exe` — Registra la DLL como componente COM en el registro de Windows — a partir de este momento `mstsc.exe` cargará el plugin en cada sesión RDP.
> `SocksOverRDP-Plugin.dll` — El plugin que intercepta el tráfico SOCKS y lo encapsula dentro del canal DVC de RDP.

---

## 📌 Paso 3 — Transferir el servidor al pivot y establecer la sesión RDP

Transferimos `SocksOverRDP-Server.exe` al host pivot (`172.16.5.19`) usando cualquier método disponible — SMB, RDP clipboard, HTTP — y lo ejecutamos con privilegios de administrador. Una vez activo, el servidor queda escuchando conexiones SOCKS entrantes desde el canal DVC.

A continuación, desde el host atacante establecemos la sesión RDP hacia el pivot:

> [!tip] Conectar por RDP al pivot con mstsc.exe
> ```cmd
> mstsc.exe /v:172.16.5.19
> # Credenciales: victor:pass@123
> ```
> Al conectar, aparece un prompt confirmando que el plugin SocksOverRDP está activo y escuchando en `127.0.0.1:1080`. Esto indica que el canal DVC está establecido correctamente entre el plugin (atacante) y el servidor (pivot).

---

## 📌 Paso 4 — Verificar el listener SOCKS

Una vez establecida la sesión RDP con el plugin activo, verificamos en el host atacante que el proxy SOCKS está escuchando correctamente en el puerto `1080`:

> [!tip] Confirmar que el listener SOCKS está activo
> ```cmd
> netstat -antb | findstr 1080
> ```
> `netstat -antb` — Muestra todas las conexiones TCP activas (`-a`), en formato numérico (`-n`), con tiempos (`-t`) y el ejecutable responsable de cada conexión (`-b`).
> `findstr 1080` — Filtra las líneas que contienen el puerto `1080` — equivalente a `grep` en Linux.

El output esperado confirma el listener activo:
```
TCP    127.0.0.1:1080    0.0.0.0:0    LISTENING
```

---

## 📌 Paso 5 — Configurar Proxifier para enrutar el tráfico

Con el proxy SOCKS activo en `127.0.0.1:1080`, configuramos Proxifier en el host atacante para que intercepte el tráfico de `mstsc.exe` y lo enrute a través del proxy:

**En Proxifier:**
1. Ir a `Profile → Proxy Servers → Add`
2. Configurar: Server `127.0.0.1`, Port `1080`, Protocol `SOCKS5`
3. Ir a `Profile → Proxification Rules`
4. Añadir regla para que `mstsc.exe` use el proxy configurado

Con Proxifier activo, al lanzar `mstsc.exe` para conectar a `172.16.6.155` (el host en la red interna del pivot), todo el tráfico viaja a través del proxy SOCKS de SocksOverRDP — que lo encapsula en RDP — hasta el servidor en `172.16.5.19` que lo desencapsula y lo enruta hacia `172.16.6.155`.

---

## 📌 Consideraciones de rendimiento RDP

Al gestionar múltiples sesiones RDP simultáneas o trabajar con conexiones lentas, el rendimiento puede degradarse considerablemente. Para optimizarlo:

> [!tip] Reducir el consumo de ancho de banda en sesiones RDP
> En `mstsc.exe` antes de conectar: ir a la pestaña **Experience** y cambiar el perfil de rendimiento a **Modem**. Esto deshabilita efectos visuales, wallpaper, animaciones y otras características que consumen ancho de banda innecesariamente durante el pivoting.

> [!important] Tener múltiples sesiones RDP activas simultáneamente con SocksOverRDP puede generar una carga significativa sobre el ancho de banda disponible, ya que todo el tráfico de pivoting viaja encapsulado dentro de las sesiones RDP. En engagements reales con conexiones lentas, reducir el perfil de rendimiento de RDP al mínimo es prácticamente obligatorio.

---

## 🔗 Relaciones / Contexto

SocksOverRDP es la solución específica para el escenario de pivoting en redes Windows puras donde RDP está disponible pero SSH no lo está — situación muy común en entornos corporativos Windows donde el puerto 22 está bloqueado por política pero el 3389 está abierto para administración remota. Comparado con netsh portproxy (que solo hace port forwarding uno a uno) y plink (que requiere un servidor SSH), SocksOverRDP ofrece un proxy SOCKS completo encapsulado en un protocolo que casi nunca está bloqueado. La combinación con Proxifier replica en Windows exactamente la funcionalidad de proxychains en Linux, haciendo que cualquier aplicación Windows pueda enrutar su tráfico a través del pivot RDP de forma completamente transparente.