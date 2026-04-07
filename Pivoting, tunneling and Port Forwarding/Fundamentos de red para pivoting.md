___
## 🧠 Concepto clave

El pivoting efectivo requiere entender con precisión cómo fluye el tráfico en una red: qué interfaces tiene un host comprometido, a qué redes tiene acceso, cómo decide el sistema operativo por dónde enviar cada paquete, y qué puertos y protocolos pueden ser aprovechados para tunelizar tráfico sin ser bloqueados. Estos conceptos no son teóricos — en cada host comprometido, la primera acción es leer la configuración de red para identificar oportunidades de pivoting.

| Concepto | Relevancia para pivoting |
|---|---|
| **IP + NIC** | Un host con múltiples NICs tiene acceso a múltiples redes — candidato directo a pivot host |
| **Tabla de routing** | Revela qué redes son alcanzables desde el host comprometido y por qué interfaz |
| **Subnetting** | Permite identificar qué hosts pertenecen a cada segmento de red al que tenemos acceso |
| **Puertos y protocolos** | Los puertos permitidos en el firewall son los canales disponibles para tunelizar tráfico |

---

## 📌 Direccionamiento IP y tarjetas de red (NICs)

Todo host que se comunica en una red tiene al menos una dirección IP asignada a una **NIC (Network Interface Controller)**. Un host puede tener múltiples NICs — físicas o virtuales — y por tanto múltiples IPs en redes distintas. Identificar hosts **dual-homed** o **multi-homed** es el primer paso para encontrar oportunidades de pivoting.

> [!info] **NIC (Network Interface Controller)**
> Componente hardware o virtual que conecta un equipo a una red. Cada NIC tiene una dirección MAC única asignada de fábrica y una dirección IP asignada por DHCP o de forma estática. Los servidores, routers, switches e impresoras frecuentemente tienen IPs estáticas. Los hosts de usuario obtienen su IP de un servidor DHCP. Un host con más de una NIC activa tiene conectividad simultánea con más de una red — es la condición necesaria para actuar como pivot host.

### Enumeración de NICs en Linux

> [!tip] Ver interfaces de red, IPs asignadas y estadísticas de tráfico
> ```bash
> ifconfig
> # Alternativa moderna
> ip a
> ip addr show
> ```
> La salida muestra cada interfaz con su identificador (`eth0`, `eth1`, `tun0`, `lo`), dirección IPv4, máscara de subred, dirección IPv6 y contadores de tráfico.

Interpretar la salida de `ifconfig` en un host comprometido:
```
eth0: inet 134.122.100.200  — IP pública, interfaz hacia Internet
eth1: inet 10.106.0.172     — IP privada, interfaz hacia red interna
tun0: inet 10.10.15.54      — Interfaz de túnel VPN activa
lo:   inet 127.0.0.1        — Loopback, solo tráfico local
```

> [!important] La presencia de una interfaz `tun` o `tap` indica que hay una VPN activa en el host comprometido. Esto puede significar que el host tiene acceso a redes adicionales a través del túnel VPN — explorar la tabla de routing para ver qué redes se anuncian a través de esa interfaz.

**Tipos de IP según visibilidad:**

| Tipo | Rango | Enrutamiento |
|---|---|---|
| Pública | Cualquier IP fuera de los rangos privados | Enrutable en Internet — visible desde fuera |
| Privada clase A | `10.0.0.0/8` | Solo enrutable en redes internas |
| Privada clase B | `172.16.0.0/12` | Solo enrutable en redes internas |
| Privada clase C | `192.168.0.0/16` | Solo enrutable en redes internas |
| Loopback | `127.0.0.0/8` | Solo comunicación interna al propio host |

### Enumeración de NICs en Windows

> [!tip] Ver interfaces de red y direcciones IP en Windows
> ```cmd
> ipconfig
> ipconfig /all
> ```
> `ipconfig` — Muestra IP, máscara y gateway de cada adaptador.
> `/all` — Añade MAC address, servidor DHCP, servidores DNS, lease info — mucho más detalle, útil para mapear la infraestructura.

La salida de `ipconfig` revela adaptadores con **Media disconnected** (sin cable/sin conexión) y adaptadores con IPs asignadas. Los adaptadores con IPs son los que importan para pivoting. Un adaptador con nombre como `NordLynx`, `Cisco AnyConnect` o `TAP-Windows` indica software VPN instalado — potencial acceso adicional a redes remotas.

---

## 📌 Routing — Cómo decide un host por dónde enviar el tráfico

La **tabla de routing** es la estructura que el sistema operativo consulta para decidir por qué interfaz y hacia qué gateway enviar cada paquete en función de su IP de destino. En el contexto de pivoting, la tabla de routing del host comprometido revela exactamente a qué redes tiene acceso — y es la información más valiosa tras obtener acceso a un nuevo host.

> [!info] **Tabla de routing**
> Estructura de datos mantenida por el sistema operativo que contiene entradas de la forma `red_destino → gateway → interfaz`. Cuando un paquete sale del host, el OS busca la entrada más específica (longest prefix match) que coincida con la IP de destino. Si no hay ninguna entrada específica, el paquete se envía al **default gateway** (gateway de último recurso). Los hosts aprenden rutas de tres formas: interfaces directamente conectadas (automático), rutas estáticas (configuración manual) y protocolos de routing dinámico (OSPF, EIGRP, BGP — típico en routers dedicados).

> [!tip] Ver la tabla de routing en Linux
> ```bash
> netstat -r
> ip route
> ip route show
> ```
> `netstat -r` — Muestra la tabla de routing en formato clásico con columnas Destination, Gateway, Genmask, Flags e Iface.
> `ip route` — Formato moderno, equivalente. Más limpio y con más información en algunos casos.

Interpretación de una tabla de routing típica en un pivot host:
```
Destination     Gateway         Genmask         Flags   Iface
default         178.62.64.1     0.0.0.0         UG      eth0    ← todo lo no específico sale por eth0 hacia Internet
10.10.10.0      10.10.14.1      255.255.254.0   UG      tun0    ← red de laboratorio HTB, vía túnel VPN
10.10.14.0      0.0.0.0         255.255.254.0   U       tun0    ← red del túnel VPN, directamente conectada
10.106.0.0      0.0.0.0         255.255.240.0   U       eth1    ← red interna, directamente conectada
10.129.0.0      10.10.14.1      255.255.0.0     UG      tun0    ← otra red de laboratorio, vía VPN
178.62.64.0     0.0.0.0         255.255.192.0   U       eth0    ← red del ISP, directamente conectada
```

Flags relevantes: `U` (ruta activa), `G` (hay un gateway — no es una red directamente conectada), `H` (ruta a host específico).

> [!tip] Ver la tabla de routing en Windows
> ```cmd
> route print
> netstat -r
> ```
> `route print` — Muestra las tablas de routing IPv4 e IPv6 en formato Windows, incluyendo rutas persistentes y de interfaz.

> [!tip] Añadir una ruta estática para alcanzar una red interna a través del pivot host
> ```bash
> # Linux — añadir ruta hacia 172.16.5.0/24 a través de 10.129.0.1
> ip route add 172.16.5.0/24 via 10.129.0.1
> sudo ip route add 172.16.5.0/24 dev tun0
>
> # Windows
> route add 172.16.5.0 mask 255.255.255.0 10.129.0.1
> ```
> `via 10.129.0.1` — Gateway a través del cual alcanzar la red destino — normalmente la IP del pivot host en nuestra red.
> `dev tun0` — Interfaz por la que enviar el tráfico cuando no hay gateway explícito.

---

## 📌 Subnetting — Identificar redes y hosts alcanzables

Entender subnetting permite, dado el IP y la máscara de un host comprometido, calcular qué otros hosts pertenecen a su misma red y son directamente alcanzables sin pasar por un router.

| Notación CIDR | Máscara | Hosts utilizables | Uso típico |
|---|---|---|---|
| `/8` | `255.0.0.0` | 16.777.214 | Redes clase A corporativas grandes |
| `/16` | `255.255.0.0` | 65.534 | Redes de campus o datacenter |
| `/24` | `255.255.255.0` | 254 | Segmento LAN estándar |
| `/25` | `255.255.255.128` | 126 | Segmento dividido |
| `/30` | `255.255.255.252` | 2 | Enlace punto a punto |

> [!tip] Calcular la red a la que pertenece un host y su rango de IPs
> ```bash
> # Dado: IP 10.106.0.172, máscara 255.255.240.0 (/20)
> # Red: 10.106.0.0/20
> # Rango de hosts: 10.106.0.1 — 10.106.15.254
> # Broadcast: 10.106.15.255
>
> # Con ipcalc (Linux)
> ipcalc 10.106.0.172/20
>
> # Con Python rápido
> python3 -c "import ipaddress; n=ipaddress.ip_interface('10.106.0.172/20'); print(n.network)"
> ```

En pivoting, este cálculo es clave: si un host comprometido tiene `eth1: 10.106.0.172/20`, sabemos que la red `10.106.0.0/20` es directamente alcanzable desde ese host y debemos escanearla para descubrir más objetivos.

---

## 📌 Protocolos, servicios y puertos en el contexto de pivoting

Los firewalls de red filtran tráfico basándose en puertos y protocolos. Al construir túneles de pivoting, debemos usar puertos y protocolos que el firewall permita para evitar ser bloqueados.

> [!info] **Puerto lógico**
> Identificador numérico (0–65535) asignado por software a una aplicación o servicio. No tiene existencia física — es una abstracción del sistema operativo para multiplexar conexiones de red. Los puertos bien conocidos (0–1023) están asignados a servicios estándar (HTTP:80, HTTPS:443, SSH:22, DNS:53). Los puertos registrados (1024–49151) se asignan a aplicaciones específicas. Los puertos efímeros (49152–65535) los usa el OS para conexiones de cliente salientes.

Puertos frecuentemente permitidos en firewalls corporativos y aprovechables para tunneling:

| Puerto | Protocolo | Uso legítimo | Uso en pivoting |
|---|---|---|---|
| 80 | HTTP | Tráfico web | Tunelizar C2 y datos en peticiones HTTP |
| 443 | HTTPS | Tráfico web cifrado | Tunelizar C2 en TLS — muy difícil de inspeccionar |
| 22 | SSH | Administración remota segura | SSH tunneling (local, remote, dynamic forwarding) |
| 53 | DNS | Resolución de nombres | DNS tunneling para exfiltración |
| 3128 | HTTP proxy | Proxy corporativo | Tunelizar tráfico a través del proxy existente |

> [!important] Al configurar listeners para reverse shells o conexiones de callback en pivoting, elegir puertos que probablemente estén permitidos en el firewall de salida: **443** es el más seguro (tráfico HTTPS legítimo), seguido de **80** (HTTP) y **53** (DNS). Puertos no estándar como 4444 o 8080 pueden ser bloqueados o alertar a sistemas IDS/IPS. Adaptar siempre el puerto del listener al entorno detectado.

> [!tip] Verificar conectividad a un puerto específico desde el host comprometido
> ```bash
> # Comprobar si el puerto 443 de nuestra máquina atacante es alcanzable
> curl -sk https://10.10.14.3:443
> nc -zv 10.10.14.3 443
>
> # Verificar qué puertos están abiertos hacia el exterior
> for port in 22 53 80 443 8080 8443; do
>     (echo >/dev/tcp/10.10.14.3/$port) 2>/dev/null && echo "$port OPEN" || echo "$port CLOSED"
> done
> ```
> Esta verificación es crítica antes de configurar un reverse shell o un túnel — confirma que el tráfico saliente en ese puerto no está bloqueado.

---

## 📌 Comandos de enumeración de red post-explotación — checklist

Tras comprometer un host, ejecutar sistemáticamente estos comandos para identificar oportunidades de pivoting:

> [!tip] Enumeración completa de red en un host Linux comprometido
> ```bash
> # Interfaces y direcciones IP
> ip a
> ifconfig
>
> # Tabla de routing — redes alcanzables
> ip route
> netstat -r
>
> # Conexiones activas y puertos en escucha
> ss -tulnp
> netstat -tulnp
>
> # Hosts conocidos en la red local (ARP cache)
> arp -a
> ip neigh
>
> # Resolución DNS — puede revelar nombres de hosts internos
> cat /etc/resolv.conf
> cat /etc/hosts
>
> # Interfaces de red con más detalle
> cat /proc/net/fib_trie   # Tabla de routing del kernel
> cat /proc/net/arp        # Cache ARP del kernel
> ```

> [!tip] Enumeración completa de red en un host Windows comprometido
> ```cmd
> rem Interfaces y direcciones IP
> ipconfig /all
>
> rem Tabla de routing
> route print
> netstat -r
>
> rem Conexiones activas y puertos en escucha
> netstat -ano
>
> rem ARP cache — hosts conocidos en red local
> arp -a
>
> rem Hosts en /etc/hosts equivalente
> type C:\Windows\System32\drivers\etc\hosts
>
> rem Configuración DNS
> ipconfig /displaydns
> nslookup -type=any _ldap._tcp.dc._msdcs.dominio.local
> ```

---

## 🔗 Relaciones / Contexto

Estos conceptos son la base sobre la que se construyen todas las técnicas de pivoting del módulo. La secuencia práctica en un engagement es siempre la misma: comprometer host → `ip a` / `ifconfig` para ver NICs → `ip route` para ver redes alcanzables → `arp -a` para ver hosts ya conocidos en esas redes → decidir qué herramienta de pivoting usar (socat, chisel, SSH forwarding, Metasploit AutoRoute) en función de las restricciones del entorno. Documentar meticulosamente cada IP, cada subred y cada ruta descubierta es imprescindible en assessments complejos con múltiples pivots — un mapa de red actualizado evita perder el rastro de qué segmentos ya han sido explorados y cuáles quedan pendientes.