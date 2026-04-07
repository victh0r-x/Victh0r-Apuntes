
## 🧠 Concepto clave

El ICMP tunneling encapsula tráfico arbitrario dentro de paquetes ICMP Echo Request/Reply — los mismos paquetes que usa el comando `ping`. Es la técnica de tunneling de último recurso: cuando SSH, HTTP, HTTPS y DNS están todos bloqueados, ICMP frecuentemente sigue permitido porque bloquearlo rompe el diagnóstico de red básico que los administradores necesitan para operar la infraestructura.

| Protocolo | Puerto | Bloqueable fácilmente |
|---|---|---|
| SSH | TCP 22 | Sí |
| HTTP/S | TCP 80/443 | Sí (con inspección) |
| DNS | UDP 53 | Difícil (rompe AD) |
| **ICMP** | **Sin puerto — protocolo L3** | **Muy difícil** |

La diferencia fundamental con todos los demás métodos de tunneling es que ICMP **no usa puertos**. Es un protocolo de capa 3 — no tiene el concepto de puerto origen/destino. Esto hace que las reglas de firewall basadas en puertos sean completamente ineficaces contra él.

> [!info] **ptunnel-ng** 📄 [github.com/utoni/ptunnel-ng](https://github.com/utoni/ptunnel-ng)
> Fork moderno de la herramienta original `ptunnel`. Crea un túnel TCP sobre paquetes ICMP Echo Request/Reply. Funciona en modelo cliente-servidor: el servidor corre en el pivot comprometido y escucha paquetes ICMP entrantes, el cliente corre en el atacante y envía tráfico TCP encapsulado en pings. El tráfico resultante en la red parece exclusivamente tráfico ICMP — un Wireshark mostrará únicamente paquetes ping, no TCP ni SSH. Requiere privilegios de root en ambos lados porque la creación de raw sockets ICMP requiere permisos de kernel.

> [!important] ICMP tunneling **solo funciona si el pivot puede recibir pings desde el atacante**. Si el firewall bloquea ICMP entrante al pivot (lo que algunos entornos muy restrictivos hacen), esta técnica no es viable. Siempre verificar primero con `ping 10.129.202.64` desde el atacante antes de intentar configurar el túnel.

---

## 📌 Escenario
```
[Atacante 10.10.14.18]
    │  ptunnel-ng client — encapsula SSH en pings ICMP
    │  expone puerto local 2222
    ▼  (tráfico visible en red: solo ICMP Echo Request/Reply)
[Ubuntu pivot 10.129.202.64]
    │  ptunnel-ng server — desencapsula ICMP → TCP
    │  reenvía a 10.129.202.64:22 (SSH)
    ▼
[SSH del propio pivot — o red interna con SSH -D]
```

---

## 📌 Instalación y compilación

> [!tip] Clonar ptunnel-ng y compilar con autogen.sh
> ```bash
> git clone https://github.com/utoni/ptunnel-ng.git
> cd ptunnel-ng
> sudo ./autogen.sh
> ```
>
> `git clone` — Descarga el repositorio completo de ptunnel-ng.
> `sudo ./autogen.sh` — Script que genera los archivos de configuración de autotools, configura el proyecto para el sistema actual y compila los binarios. Requiere `sudo` porque ejecuta pasos de configuración del sistema. Genera el binario `ptunnel-ng` en el subdirectorio `src/`.

> [!tip] Compilar ptunnel-ng como binario estático — más portable para transferir al pivot
> ```bash
> sudo apt install automake autoconf -y
> cd ptunnel-ng/
> sed -i '$s/.*/LDFLAGS=-static "${NEW_WD}\/configure" --enable-static $@ \&\& make clean \&\& make -j${BUILDJOBS:-4} all/' autogen.sh
> ./autogen.sh
> ```
>
> `sudo apt install automake autoconf -y` — Instala las herramientas de build necesarias para compilar ptunnel-ng desde código fuente.
> `sed -i '$s/...'` — Modifica la última línea del script `autogen.sh` para añadir el flag `-static` al linker, forzando que todas las librerías se incluyan en el binario en lugar de enlazarse dinámicamente. El resultado es un binario completamente autónomo que funciona en cualquier sistema Linux sin importar qué librerías tenga instaladas.
> `./autogen.sh` — Compila con los flags estáticos añadidos.

> [!tip] Transferir ptunnel-ng al pivot host
> ```bash
> scp -r ptunnel-ng ubuntu@10.129.202.64:~/
> ```
>
> `-r` — Recursive: copia el directorio completo incluyendo el binario compilado en `src/ptunnel-ng`.
> `ubuntu@10.129.202.64:~/` — Directorio home del usuario ubuntu en el pivot.

---

## 📌 Paso 1 — Iniciar el servidor ptunnel-ng en el pivot

> [!tip] Lanzar el servidor ptunnel-ng en el pivot host
> ```bash
> # En el pivot Ubuntu
> cd ptunnel-ng/src/
> sudo ./ptunnel-ng -r10.129.202.64 -R22
> ```
>
> `sudo` — Obligatorio: ptunnel-ng necesita crear raw sockets ICMP, lo que requiere privilegios de root en Linux.
> `-r10.129.202.64` — **Remote host**: IP del pivot mismo — la dirección en la que ptunnel-ng aceptará las conexiones ICMP entrantes del cliente. Debe ser la IP accesible desde el atacante.
> `-R22` — **Remote port**: puerto TCP al que ptunnel-ng reenviará el tráfico desencapsulado. En este caso el puerto SSH del propio pivot. Se puede usar cualquier puerto TCP accesible desde el pivot: `22` para SSH, `3389` para RDP de un host interno, etc.

El servidor confirma que está activo y escuchando paquetes ICMP:
```
[inf]: Starting ptunnel-ng 1.42.
[inf]: Forwarding incoming ping packets over TCP.
[inf]: Ping proxy is listening in privileged mode.
[inf]: Dropping privileges now.
```

---

## 📌 Paso 2 — Conectar el cliente ptunnel-ng desde el atacante

> [!tip] Lanzar el cliente ptunnel-ng en el atacante creando el puerto local 2222
> ```bash
> # En el atacante
> sudo ./ptunnel-ng -p10.129.202.64 -l2222 -r10.129.202.64 -R22
> ```
>
> `sudo` — Igual que el servidor, el cliente necesita root para crear raw sockets ICMP.
> `-p10.129.202.64` — **Proxy host**: IP del servidor ptunnel-ng en el pivot — hacia donde el cliente enviará los paquetes ICMP con el tráfico encapsulado.
> `-l2222` — **Listen port**: puerto TCP local en el atacante donde ptunnel-ng expondrá el servicio tunelizado. Cualquier conexión a `127.0.0.1:2222` en el atacante viajará encapsulada en pings ICMP hasta el pivot, que la desencapsulará y la enviará a `10.129.202.64:22`.
> `-r10.129.202.64` — **Remote host** destino final — debe coincidir con el `-r` configurado en el servidor.
> `-R22` — **Remote port** destino final — el puerto SSH del pivot. Debe coincidir con el `-R` del servidor.

El cliente confirma que está relaying:
```
[inf]: Starting ptunnel-ng 1.42.
[inf]: Relaying packets from incoming TCP streams.
```

---

## 📌 Paso 3 — Usar el túnel ICMP para SSH

Con el túnel activo, SSH se conecta al **puerto local 2222** del atacante — ptunnel-ng se encarga de transportar ese tráfico dentro de pings:

> [!tip] Conectar por SSH a través del túnel ICMP
> ```bash
> ssh -p2222 -l ubuntu 127.0.0.1
> ```
>
> `-p2222` — Puerto de destino SSH: **no el 22 estándar** sino el puerto local que ptunnel-ng expone. La conexión va a `127.0.0.1:2222` → ptunnel-ng client → paquetes ICMP → ptunnel-ng server en pivot → `10.129.202.64:22`.
> `-l ubuntu` — **Login user**: usuario SSH en el pivot. Equivalente a `ubuntu@`.
> `127.0.0.1` — Conectar al **loopback local** — donde ptunnel-ng client está escuchando. No se conecta directamente al pivot.

Si todo está correctamente configurado, la sesión SSH se establece completamente a través de pings ICMP:
```
ubuntu@127.0.0.1's password:
Welcome to Ubuntu 20.04.3 LTS...
ubuntu@WEB01:~$
```

Las estadísticas de sesión en el servidor muestran el tráfico ICMP procesado:
```
[inf]: Incoming tunnel request from 10.10.14.18.
[inf]: Starting new session to 10.129.202.64:22 with ID 20199
Session statistics:
[inf]: I/O: 0.00/0.00 mb ICMP I/O/R: 248/22/0 Loss: 0.0%
```

---

## 📌 Paso 4 — Dynamic port forwarding sobre el túnel ICMP

Una vez establecida la sesión SSH a través del túnel ICMP, se puede añadir dynamic port forwarding para obtener un proxy SOCKS completo hacia la red interna — combinando ICMP tunneling con todas las capacidades de SSH:

> [!tip] SSH con dynamic port forwarding a través del túnel ICMP
> ```bash
> ssh -D 9050 -p2222 -l ubuntu 127.0.0.1
> ```
>
> `-D 9050` — **Dynamic port forwarding**: crea un proxy SOCKS en `127.0.0.1:9050` del atacante. Todo el tráfico enviado a ese proxy saldrá por el pivot hacia la red interna `172.16.5.0/23`.
> `-p2222` — Conectar a través del túnel ICMP de ptunnel-ng (puerto local 2222).
> `-l ubuntu` — Usuario SSH.
> `127.0.0.1` — Loopback local donde ptunnel-ng escucha.

> [!tip] Configurar proxychains y escanear la red interna a través del túnel ICMP
> ```bash
> # /etc/proxychains.conf — última línea:
> # socks4 127.0.0.1 9050
>
> # Escanear el DC interno a través de ICMP → SSH → SOCKS
> proxychains nmap -sV -sT -p3389 172.16.5.19 -Pn
> ```

El flujo completo de datos con dynamic port forwarding activo:
```
proxychains nmap → 127.0.0.1:9050 (SOCKS SSH -D)
    → SSH sobre ptunnel-ng → 127.0.0.1:2222
    → paquetes ICMP Echo Request hacia 10.129.202.64
    → ptunnel-ng server desencapsula → SSH en 10.129.202.64:22
    → SSH -D reenvía hacia 172.16.5.19:3389
```

---

## 📌 Verificación con Wireshark — diferencia visible en el tráfico

La diferencia entre SSH normal y SSH tunelizado por ICMP es dramática y visible en cualquier captura de red:

| Tráfico capturado | SSH normal | SSH sobre ptunnel-ng |
|---|---|---|
| Protocolo visible | TCP + SSHv2 | Solo ICMP Echo Request/Reply |
| Puertos visibles | TCP 22 | Ninguno — ICMP no tiene puertos |
| Payload aparente | Datos SSH cifrados | Datos ping (aparentemente) |
| Detectable como SSH | Sí (puerto 22, banner SSHv2) | No — parece tráfico ping normal |
```bash
# Comando SSH normal — captura muestra TCP+SSH:
ssh ubuntu@10.129.202.64

# Comando SSH sobre ICMP — captura muestra SOLO pings:
ssh -p2222 -l ubuntu 127.0.0.1
```

---

## 📌 Limitaciones de ICMP tunneling

> [!important] Consideraciones antes de usar ptunnel-ng en un engagement
> - **Velocidad limitada** — ICMP no está diseñado para transportar datos arbitrarios. El throughput es considerablemente menor que TCP directo. Operaciones intensivas en datos son lentas.
> - **Requiere root en ambos lados** — La creación de raw sockets ICMP necesita privilegios de kernel tanto en el servidor (pivot) como en el cliente (atacante). Si no se tiene sudo en el pivot, no es viable.
> - **Detectable por volumen** — Aunque el contenido parezca ICMP, un volumen anormalmente alto de paquetes ping o pings con payloads de tamaño inusual (ICMP normal tiene 32-64 bytes de payload) es una señal clara para soluciones de seguridad que analizan patrones de tráfico.
> - **Algunos firewalls inspeccionan ICMP payload** — Firewalls avanzados con Deep Packet Inspection pueden detectar que el payload de los pings no corresponde a datos ICMP legítimos.

---

## 🔗 Relaciones / Contexto

ICMP tunneling con ptunnel-ng es la penúltima línea de defensa en términos de evasión de firewalls — solo superada en sigilo por técnicas como DNS tunneling o esteganografía en protocolos legítimos. En la práctica del eCPPT y engagements reales, se usa cuando todos los otros métodos (SSH, chisel sobre HTTP, rpivot, sshuttle) han sido bloqueados por el firewall o detectados por el blue team. La combinación ptunnel-ng + SSH -D es especialmente potente porque añade un proxy SOCKS completo sobre el canal ICMP, dando acceso total a la red interna con un tráfico que en la captura de red parece exclusivamente tráfico de diagnóstico ping — algo que prácticamente ningún administrador cuestiona.