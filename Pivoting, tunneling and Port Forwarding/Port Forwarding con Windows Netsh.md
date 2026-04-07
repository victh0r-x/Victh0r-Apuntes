
## 🧠 Concepto clave

`netsh.exe` es una herramienta de administración de red nativa de Windows — no requiere descargar ni instalar nada. Entre sus muchas funciones, incluye un módulo `portproxy` que permite crear reglas de port forwarding TCP directamente a nivel del sistema operativo. Es el equivalente Windows nativo de `socat TCP4-LISTEN:PUERTO,fork TCP4:DESTINO:PUERTO` pero sin necesitar ningún binario externo.

En un engagement donde se gana acceso a una workstation Windows (por ejemplo a través de phishing o ingeniería social), `netsh` permite usar ese host como pivot hacia redes internas sin subir ninguna herramienta adicional — pura técnica de **living off the land**.

| Herramienta | OS | Requiere descarga | Equivalente función |
|---|---|---|---|
| `socat` | Linux | Sí (o preinstalado) | Port forwarding TCP |
| `ssh -L` | Linux/Windows | Sí (cliente SSH) | Port forwarding local |
| `chisel` | Ambos | Sí (binario) | Tunneling + SOCKS |
| **`netsh portproxy`** | **Windows** | **No — nativo** | **Port forwarding TCP** |

> [!info] **netsh.exe — Network Shell**
> Herramienta de línea de comandos nativa de Windows para configurar y monitorizar parámetros de red. Usa un sistema de contextos y subcontextos: `interface`, `firewall`, `wlan`, `http`, etc. El subcontexto `interface portproxy` gestiona específicamente las reglas de reenvío de puertos TCP a nivel del sistema operativo. Las reglas creadas persisten entre reinicios del sistema, a diferencia de túneles SSH o socat que mueren al cerrar el proceso.

---

## 📌 Escenario
```
[Atacante 10.10.14.18]
    │  quiere acceder al RDP de 172.16.5.25
    │  no tiene ruta directa a 172.16.5.0/23
    ▼
[Windows pivot — workstation IT 10.129.15.150 / 172.16.5.19]
    │  netsh portproxy: escucha :8080 → reenvía a 172.16.5.25:3389
    │  tiene acceso a ambas redes
    ▼
[Host interno Windows 172.16.5.25]
    │  RDP activo en :3389
    │  solo accesible desde 172.16.5.0/23
```

El atacante conecta al puerto `8080` del pivot Windows. Netsh intercepta esa conexión y la reenvía transparentemente al puerto `3389` del host interno `172.16.5.25`.

---

## 📌 Paso 1 — Crear la regla de port forwarding con netsh

> [!tip] Crear regla portproxy v4tov4 en el pivot Windows
> ```cmd
> netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25
> ```
>
> `interface portproxy` — Subcontexto de netsh que gestiona las reglas de reenvío de puertos TCP.
> `add v4tov4` — **Añadir** una regla de tipo IPv4-to-IPv4. Otras variantes posibles: `v4tov6`, `v6tov4`, `v6tov6` para entornos con IPv6.
> `listenport=8080` — Puerto en el que el pivot Windows escuchará conexiones entrantes del atacante. Se elige 8080 para no interferir con RDP local (3389) ni levantar sospechas inmediatas.
> `listenaddress=10.129.15.150` — Interfaz IP del pivot donde escuchar. Al especificar la IP concreta en lugar de `0.0.0.0`, la regla solo aplica a conexiones que lleguen por esa interfaz específica — la que da al atacante.
> `connectport=3389` — Puerto destino al que se reenviará el tráfico recibido — el puerto RDP del host interno.
> `connectaddress=172.16.5.25` — IP del host interno destino al que se reenviarán las conexiones. Debe ser alcanzable desde el pivot Windows a través de la red interna `172.16.5.0/23`.

El comando no produce output si se ejecuta correctamente — la ausencia de error es la confirmación.

---

## 📌 Paso 2 — Verificar que la regla se creó correctamente

> [!tip] Listar todas las reglas portproxy activas en el sistema
> ```cmd
> netsh.exe interface portproxy show v4tov4
> ```
>
> `show v4tov4` — Muestra todas las reglas de reenvío IPv4-to-IPv4 actualmente configuradas en el sistema.

El output confirma la regla activa:
```
Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
10.129.15.150   8080        172.16.5.25     3389
```

La regla está activa: cualquier conexión TCP que llegue a `10.129.15.150:8080` será reenviada a `172.16.5.25:3389`.

---

## 📌 Paso 3 — Configurar el firewall de Windows para permitir el puerto

> [!important] Crear la regla portproxy **no abre automáticamente el puerto en el firewall de Windows**. Si el Windows Firewall está activo (habitual en workstations corporativas), el puerto 8080 estará bloqueado para conexiones entrantes aunque la regla portproxy exista. Es necesario añadir una regla de firewall explícita que permita el tráfico entrante en ese puerto.

> [!tip] Añadir regla al firewall de Windows para permitir conexiones entrantes en el puerto 8080
> ```cmd
> netsh.exe advfirewall firewall add rule name="pivot-8080" protocol=TCP dir=in localport=8080 action=allow
> ```
>
> `advfirewall firewall add rule` — Añade una nueva regla al Windows Defender Firewall (Advanced Firewall).
> `name="pivot-8080"` — Nombre descriptivo de la regla — aparecerá en la lista de reglas del firewall. Elegir un nombre que no llame la atención en una auditoría del firewall (por ejemplo `"WindowsUpdateService"` sería más sigiloso en un engagement real).
> `protocol=TCP` — La regla aplica solo a tráfico TCP.
> `dir=in` — **Dirección entrante** — controla el tráfico que llega al sistema. Necesario para que el atacante pueda conectar al puerto 8080 del pivot.
> `localport=8080` — Puerto local al que aplica la regla — el mismo que configuramos en el portproxy.
> `action=allow` — Acción: **permitir** el tráfico (en lugar de bloquearlo o pedir confirmación).

---

## 📌 Paso 4 — Conectar al host interno desde el atacante

Con la regla portproxy activa y el firewall configurado, el atacante puede conectar directamente al puerto `8080` del pivot Windows:

> [!tip] Conectar por RDP al host interno a través del port forward de netsh
> ```bash
> # Desde el atacante Linux
> xfreerdp /v:10.129.15.150:8080 /u:victor /p:pass@123
> ```
>
> `/v:10.129.15.150:8080` — Conectar al pivot Windows en el puerto `8080` donde netsh está escuchando y reenviando. Desde la perspectiva del atacante, es como si el RDP estuviese en el propio pivot — netsh hace la traducción de forma completamente transparente.
> `/u:victor` — Usuario del host interno `172.16.5.25` (no del pivot).
> `/p:pass@123` — Contraseña del usuario del host interno.

El tráfico RDP fluye así:
```
xfreerdp → 10.129.15.150:8080 → [netsh portproxy] → 172.16.5.25:3389
```

---

## 📌 Gestión de reglas portproxy

> [!tip] Listar todas las reglas portproxy (todos los tipos)
> ```cmd
> netsh.exe interface portproxy show all
> ```
>
> `show all` — Muestra todas las reglas portproxy activas independientemente de su tipo (v4tov4, v4tov6, etc.).

> [!tip] Eliminar una regla portproxy específica — limpieza post-explotación
> ```cmd
> netsh.exe interface portproxy delete v4tov4 listenport=8080 listenaddress=10.129.15.150
> ```
>
> `delete v4tov4` — Elimina una regla de tipo IPv4-to-IPv4.
> `listenport=8080` — Puerto de escucha de la regla a eliminar.
> `listenaddress=10.129.15.150` — Dirección de escucha de la regla a eliminar. La combinación `listenaddress + listenport` identifica unívocamente la regla.

> [!tip] Eliminar también la regla de firewall — limpieza post-explotación
> ```cmd
> netsh.exe advfirewall firewall delete rule name="pivot-8080"
> ```
>
> `delete rule name="pivot-8080"` — Elimina la regla de firewall por su nombre. Siempre limpiar tanto la regla portproxy como la regla de firewall al terminar el engagement o al rotar el pivot.

> [!important] Las reglas `portproxy` de netsh **persisten entre reinicios del sistema** — se almacenan en el registro de Windows en `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy\v4tov4\`. Esto es útil para mantener persistencia del pivot, pero también significa que si no se limpian manualmente, quedan como artefacto detectable por el blue team incluso después de cerrar la sesión. La limpieza post-explotación es obligatoria.

---

## 📌 Casos de uso adicionales con netsh portproxy

> [!tip] Redirigir múltiples servicios internos simultáneamente
> ```cmd
> REM RDP al host interno
> netsh.exe interface portproxy add v4tov4 listenport=13389 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25
>
> REM SMB al host interno
> netsh.exe interface portproxy add v4tov4 listenport=4445 listenaddress=10.129.15.150 connectport=445 connectaddress=172.16.5.25
>
> REM WinRM al host interno
> netsh.exe interface portproxy add v4tov4 listenport=5986 listenaddress=10.129.15.150 connectport=5985 connectaddress=172.16.5.25
> ```
> Cada `add` crea una regla independiente. Se pueden tener múltiples reglas activas simultáneamente apuntando a diferentes hosts y puertos internos. Se recomienda usar puertos locales altos (>1024) y diferentes de los originales para reducir conflictos con servicios existentes en el pivot.

---

## 📌 Comparativa netsh portproxy vs otras técnicas en Windows

| Criterio | netsh portproxy | plink -L | chisel |
|---|---|---|---|
| Requiere descarga | No — nativo | Sí (plink.exe) | Sí (chisel.exe) |
| Persiste tras reinicio | Sí | No | No |
| SOCKS proxy | No | Sí (`-D`) | Sí (`R:socks`) |
| Múltiples puertos | Sí (múltiples add) | Sí (múltiples `-L`) | Sí (múltiples args) |
| Detección por AV/EDR | Muy baja (binario firmado MS) | Baja (binario legítimo) | Media (binario externo) |
| Requiere privilegios | Admin local | No | No |
| Cifrado del tráfico | No | Sí (SSH) | Sí (HTTPS opcional) |

---

## 🔗 Relaciones / Contexto

Netsh portproxy es la técnica de pivoting con menor huella en Windows porque usa exclusivamente herramientas del propio sistema operativo firmadas por Microsoft — el comportamiento ideal de living off the land. El único requisito es disponer de privilegios de administrador local en el pivot Windows, lo que en workstations corporativas IT admin (como el escenario descrito) suele estar garantizado. La desventaja frente a chisel o plink es que netsh no crea un proxy SOCKS — solo hace port forwarding uno a uno hacia servicios específicos. Para acceder a rangos completos de red interna con múltiples hosts y puertos arbitrarios, seguirá siendo necesario combinar netsh con otras técnicas o usar chisel como alternativa más versátil.