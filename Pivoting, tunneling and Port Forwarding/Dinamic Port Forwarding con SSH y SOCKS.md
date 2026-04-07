
## 🧠 Concepto clave

El **port forwarding con SSH** permite redirigir tráfico desde un puerto local de la máquina atacante hacia un servicio en una red remota no directamente accesible, usando el canal cifrado SSH como transporte. Existen tres modalidades principales, cada una con un caso de uso distinto:

| Modalidad | Flag SSH | Flujo del tráfico | Caso de uso |
|---|---|---|---|
| **Local Port Forwarding** | `-L` | Local → SSH server → destino remoto | Acceder a un servicio interno desde el atacante |
| **Remote Port Forwarding** | `-R` | Remoto → SSH server → destino local | Exponer un listener del atacante en el host comprometido |
| **Dynamic Port Forwarding** | `-D` | Local SOCKS proxy → SSH server → cualquier destino | Enrutar tráfico arbitrario hacia toda una red interna |

> [!info] **SOCKS (Socket Secure)** 📄 [SOCKS — Wikipedia](https://en.wikipedia.org/wiki/SOCKS)
> Protocolo de red de capa de sesión que actúa como proxy genérico para tráfico TCP y UDP. A diferencia de los proxies HTTP (que entienden el protocolo HTTP), SOCKS es agnóstico al protocolo de aplicación — puede tunelizar cualquier tráfico TCP sin importar si es HTTP, RDP, SMB, MySQL, etc. Existen dos versiones: **SOCKS4** (solo TCP, sin autenticación) y **SOCKS5** (TCP + UDP, soporte de autenticación). En pivoting, SSH con `-D` crea un servidor SOCKS en el atacante que enruta el tráfico a través del host comprometido hacia la red interna.

> [!info] **proxychains**
> Herramienta de Linux que intercepta las llamadas de red de cualquier aplicación y las redirige a través de uno o más proxies configurados en `/etc/proxychains.conf`. Permite usar herramientas sin soporte nativo de proxy (como `nmap`, `msfconsole`, `xfreerdp`, `crackmapexec`) a través de un túnel SOCKS establecido con SSH. La línea de configuración necesaria en `/etc/proxychains.conf` es `socks4 127.0.0.1 9050` (o el puerto que se haya configurado).

---

## 📌 Escenario de laboratorio

El entorno de trabajo tiene la siguiente topología:
```
[Atacante 10.10.15.x] ←SSH→ [Ubuntu pivot 10.129.202.64 / 172.16.5.129] ←→ [Red interna 172.16.5.0/23]
                                                                                     ↓
                                                                         [DC01 Windows 172.16.5.19]
```

El atacante no tiene ruta directa hacia `172.16.5.0/23`. El servidor Ubuntu es dual-homed y actúa como pivot host. El objetivo es alcanzar `172.16.5.19` (DC01) tunelizando el tráfico a través del Ubuntu.

---

## 📌 SSH Local Port Forwarding (-L)

El local port forwarding redirige un puerto local del atacante hacia un servicio específico en la red del SSH server. Es la modalidad más simple: se sabe exactamente qué servicio se quiere alcanzar y en qué puerto.

**Caso de uso:** el servidor Ubuntu tiene MySQL corriendo en `localhost:3306` — no accesible desde fuera. Con local port forwarding, el puerto `3306` del Ubuntu queda disponible en el `1234` del atacante.

**Paso 1 — Verificar el estado del servicio en el objetivo:**

> [!tip] Escanear puertos en el pivot host para identificar servicios locales
> ```bash
> nmap -sT -p22,3306 10.129.202.64
> ```
> `-sT` — TCP Connect Scan: completa el three-way handshake. Necesario cuando no se tienen privilegios de root para SYN scan, o cuando se escanea a través de proxychains.
> `-p22,3306` — Escanea solo los puertos 22 (SSH) y 3306 (MySQL).

El resultado muestra puerto 22 abierto y 3306 cerrado — MySQL no es accesible directamente desde el atacante, pero puede estar escuchando en localhost del servidor Ubuntu.

**Paso 2 — Establecer el túnel SSH con local port forwarding:**

> [!tip] Redirigir el puerto 3306 del Ubuntu al puerto 1234 local
> ```bash
> ssh -L 1234:localhost:3306 ubuntu@10.129.202.64
> ```
> `-L 1234:localhost:3306` — Local port forwarding: todo tráfico que llegue al puerto `1234` de nuestra máquina local se reenvía a `localhost:3306` **visto desde el servidor SSH** (el Ubuntu). `localhost` aquí significa el propio Ubuntu, no la máquina atacante.
> `ubuntu@10.129.202.64` — Usuario y dirección del SSH server (el pivot host).

La sintaxis general del flag `-L` es: `-L [IP_local:]PUERTO_LOCAL:HOST_DESTINO:PUERTO_DESTINO`

> [!tip] Reenviar múltiples puertos en una sola conexión SSH
> ```bash
> ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.64
> ```
> Cada flag `-L` adicional añade un nuevo mapeo de puertos en la misma conexión SSH. Aquí se reenvía MySQL (3306→1234) y Apache (80→8080) simultáneamente.

**Paso 3 — Verificar que el túnel está activo:**

> [!tip] Confirmar el port forward con netstat
> ```bash
> netstat -antp | grep 1234
> ```
> `-a` — Muestra todas las conexiones y puertos en escucha.
> `-n` — Muestra IPs y puertos en formato numérico (sin resolución de nombres).
> `-t` — Solo conexiones TCP.
> `-p` — Muestra el proceso propietario de cada socket.

La salida debe mostrar el proceso `ssh` escuchando en `127.0.0.1:1234`.

> [!tip] Confirmar el port forward con nmap
> ```bash
> nmap -v -sV -p1234 localhost
> ```
> `-sV` — Detección de versiones: si el túnel funciona correctamente, nmap identificará el servicio real detrás del puerto (MySQL 8.0.x en este caso, no simplemente un puerto abierto genérico).

---

## 📌 SSH Dynamic Port Forwarding (-D) + SOCKS Proxy

Cuando no se sabe qué servicios hay en la red interna, o se necesita acceso genérico a múltiples hosts y puertos, el local port forwarding no es práctico — habría que crear un `-L` por cada servicio. El **dynamic port forwarding** resuelve esto: crea un proxy SOCKS local que enruta **cualquier tráfico TCP** hacia la red interna a través del túnel SSH.

**Identificar que el Ubuntu es dual-homed:**

> [!tip] Verificar interfaces de red del pivot host para identificar redes alcanzables
> ```bash
> # En el servidor Ubuntu (ya comprometido)
> ifconfig
> ip a
> ```

La salida revela:
- `ens192: 10.129.202.64` — interfaz hacia el atacante
- `ens224: 172.16.5.129` — interfaz hacia la red interna `172.16.5.0/23`

La red `172.16.5.0/23` no es directamente alcanzable desde el atacante — necesita pivoting a través del Ubuntu.

**Paso 1 — Establecer el túnel SSH con dynamic port forwarding:**

> [!tip] Crear un proxy SOCKS en el puerto 9050 mediante SSH dynamic forwarding
> ```bash
> ssh -D 9050 ubuntu@10.129.202.64
> ```
> `-D 9050` — Dynamic port forwarding: SSH crea un servidor SOCKS en `127.0.0.1:9050` de la máquina atacante. Todo tráfico enviado a ese puerto SOCKS se tuneliza a través del SSH server (Ubuntu) y se enruta hacia el destino final desde la perspectiva del Ubuntu. El puerto `9050` es el estándar de TOR/proxychains, aunque puede usarse cualquier puerto libre.

**Paso 2 — Configurar proxychains:**

> [!tip] Verificar y configurar /etc/proxychains.conf
> ```bash
> tail -4 /etc/proxychains.conf
> ```
> La última línea debe contener `socks4 127.0.0.1 9050`. Si no está, añadirla:
> ```bash
> echo "socks4 127.0.0.1 9050" | sudo tee -a /etc/proxychains.conf
> ```
> `socks4` — Versión del protocolo SOCKS. Usar `socks5` si el servidor lo soporta (SSH dynamic forwarding soporta ambos).
> `127.0.0.1 9050` — Dirección y puerto del proxy SOCKS local creado por SSH.

**Paso 3 — Descubrir hosts en la red interna:**

> [!tip] Escaneo de hosts activos en la red interna a través del proxy SOCKS
> ```bash
> proxychains nmap -v -sn 172.16.5.1-200
> ```
> `proxychains` — Intercepta las llamadas de red de nmap y las redirige por el proxy SOCKS configurado.
> `-sn` — Ping scan: solo comprueba si los hosts están activos, sin escanear puertos.

> [!important] A través de proxychains, nmap **solo puede realizar TCP Connect Scans completos** (`-sT`). Los SYN scans (`-sS`) y otros tipos de escaneo que usan paquetes parciales no funcionan porque proxychains no puede manejar paquetes TCP incompletos. Además, los host-alive checks basados en ICMP no funcionan contra targets Windows porque Windows Defender bloquea ICMP por defecto — usar siempre `-Pn` al escanear Windows a través de proxychains.

**Paso 4 — Escanear puertos en un host específico de la red interna:**

> [!tip] Enumeración de puertos en el DC01 Windows a través del proxy SOCKS
> ```bash
> proxychains nmap -v -Pn -sT 172.16.5.19
> ```
> `-Pn` — Omite el ping/host-alive check. Imprescindible para hosts Windows a través de proxychains.
> `-sT` — TCP Connect Scan: el único tipo de escaneo compatible con proxychains.
> La salida de proxychains muestra cada conexión con el formato `|S-chain|-<>-127.0.0.1:9050-<><>-172.16.5.19:PUERTO-<><>-OK` para puertos abiertos y `<--timeout` para cerrados/filtrados.

---

## 📌 Usar herramientas de pentesting a través de proxychains

Una vez establecido el proxy SOCKS, cualquier herramienta puede usarse contra la red interna simplemente prefijando `proxychains`:

**Metasploit a través del proxy SOCKS:**

> [!tip] Lanzar msfconsole a través del proxy SOCKS
> ```bash
> proxychains msfconsole
> ```
> Todo el tráfico de red de Metasploit se enruta automáticamente a través del proxy SOCKS. Permite usar módulos auxiliares, exploits y post-explotación contra hosts de la red interna.

> [!tip] Usar el módulo rdp_scanner para verificar RDP en el DC01
> ```bash
> # Dentro de msfconsole (lanzado con proxychains)
> use auxiliary/scanner/rdp/rdp_scanner
> set RHOSTS 172.16.5.19
> run
> ```
> El módulo `rdp_scanner` confirma si el puerto 3389 responde al protocolo RDP y extrae información del host: nombre de equipo, dominio, versión de OS y si requiere NLA (Network Level Authentication).

**Conexión RDP a través del proxy SOCKS:**

> [!tip] Conectar por RDP a un host de la red interna mediante proxychains
> ```bash
> proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
> ```
> `proxychains` — Enruta la conexión RDP a través del proxy SOCKS y el túnel SSH hacia la red interna.
> `/v:172.16.5.19` — IP del host Windows de destino en la red interna.
> `/u:victor /p:pass@123` — Credenciales de acceso.
> Al primer acceso será necesario aceptar el certificado RDP del servidor.

---

## 📌 Resumen del flujo completo
```
[Atacante]                    [Ubuntu pivot]              [Red interna 172.16.5.0/23]
    │                               │                               │
    │  ssh -D 9050 ubuntu@Ubuntu    │                               │
    │──────────────────────────────►│                               │
    │                               │                               │
    │  proxychains nmap 172.16.5.19 │                               │
    │──────►[SOCKS:9050]            │                               │
    │        │  [túnel SSH cifrado] │                               │
    │        └──────────────────────►──────────────────────────────►│
    │                               │   nmap TCP connect scan       │
    │                               │◄──────────────────────────────│
    │◄──────[SOCKS:9050]────────────│                               │
    │  resultados del scan          │                               │
```

> [!important] El proxy SOCKS creado por `ssh -D` vive mientras dure la sesión SSH. Si la sesión SSH se cierra o cae, el proxy desaparece y todas las conexiones activas a través de proxychains se interrumpen. Para pivoting persistente o de larga duración, usar `ssh -fND 9050 ubuntu@10.129.202.64` — el flag `-f` manda el proceso SSH al background y `-N` indica que no se ejecutará ningún comando remoto (solo el túnel).

---

## 🔗 Relaciones / Contexto

El dynamic port forwarding con SSH + proxychains es la técnica de pivoting más versátil y fácil de desplegar cuando se tiene acceso SSH al pivot host — no requiere subir ninguna herramienta adicional, usa infraestructura legítima del sistema operativo y el tráfico va cifrado dentro del canal SSH, dificultando su detección. Es la técnica base que se usa en el eCPPT y en engagements reales antes de recurrir a herramientas más especializadas como chisel o socat. La limitación principal es que solo soporta TCP Connect Scan completo a través de proxychains — para técnicas más avanzadas como SYN scan o UDP scan a través del pivot, se necesita Metasploit AutoRoute o chisel con un servidor SOCKS5 completo.