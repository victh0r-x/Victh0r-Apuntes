____

![](assets/Introduction%20to%20Pivoting,%20Tunneling,%20and%20Port%20Forwarding-img-09-03-2026.png)

## 🧠 Concepto clave

Durante un pentest, red team engagement o evaluación de Active Directory, es habitual llegar a un punto donde se tienen credenciales, claves SSH, hashes o tokens de acceso válidos para moverse al siguiente objetivo, pero ese objetivo **no es directamente alcanzable desde la máquina atacante**. En ese momento entra en juego el pivoting: usar un host ya comprometido como trampolín para alcanzar segmentos de red previamente inaccesibles.

| Concepto | Definición operativa |
|---|---|
| **Pivoting** | Usar hosts comprometidos para cruzar fronteras de red y acceder a segmentos que no serían alcanzables directamente. Objetivo concreto: profundizar en la red atravesando la segmentación |
| **Tunneling** | Encapsular tráfico de red dentro de otro protocolo para enrutarlo a través de él, ocultando su naturaleza real. Subconjunto del pivoting orientado a la evasión y el C2 |
| **Lateral Movement** | Técnica para ampliar el acceso a más hosts, aplicaciones y servicios dentro de un segmento de red ya alcanzable, frecuentemente combinada con escalada de privilegios entre hosts |

La distinción entre los tres conceptos es importante aunque se usen a veces como sinónimos:

- **Lateral Movement** expande el acceso **en anchura** dentro de un segmento ya accesible — por ejemplo, reusar credenciales de administrador local en otros hosts del mismo segmento.
- **Pivoting** expande el acceso **en profundidad** — cruzar hacia un segmento de red diferente que no era alcanzable desde el punto de entrada original.
- **Tunneling** se centra en la **obfuscación del tráfico** — encapsular comunicaciones (C2, exfiltración, instrucciones) dentro de protocolos legítimos como HTTP/S o SSH para evadir detección.

> [!important] Un host comprometido que sirve de punto de salto recibe varios nombres en la industria según el contexto: **pivot host**, **proxy**, **foothold**, **beachhead system** o **jump host**. Todos se refieren al mismo concepto — un sistema comprometido que nos da acceso a otra capa de la red.

---

## 📌 Lateral Movement en detalle

El Lateral Movement describe el conjunto de técnicas para moverse entre hosts dentro de un entorno de red ya alcanzable, con el objetivo de ampliar el acceso y frecuentemente escalar privilegios. La clave es que opera **dentro de un segmento de red ya visible** para el atacante.

**Ejemplo práctico:** tras comprometer la cuenta de administrador local de un host Windows, se realiza un network scan y se encuentran tres hosts adicionales en el mismo segmento. Se prueban las mismas credenciales de administrador local en los tres hosts (técnica conocida como **pass-the-hash** o simplemente reutilización de credenciales). Uno de ellos comparte la misma cuenta de administrador local — se obtiene acceso lateral a ese host sin necesidad de explotar ninguna vulnerabilidad adicional.

Referencias para profundizar:
- MITRE ATT&CK — Táctica TA0008: Lateral Movement
- Palo Alto Networks — explicación de Lateral Movement en contexto de threat intelligence

---

## 📌 Pivoting en detalle

El pivoting resuelve el problema de la **segmentación de red**: un objetivo está en un segmento diferente al que el atacante tiene acceso directo. Se necesita un host comprometido que tenga conectividad con **ambos** segmentos (dual-homed) o al menos con el segmento objetivo.

**Ejemplo práctico:** durante un engagement, la red del cliente estaba segmentada física y lógicamente — la red corporativa y la red operacional (OT/ICS) eran completamente separadas. Se identificó una workstation de ingeniería dual-homed con una NIC en cada red — su función era administrar equipos industriales y generar informes corporativos. Al comprometer ese host, se obtuvo acceso a la red operacional que de otro modo hubiera sido inaccesible.

> [!info] **Host dual-homed**
> Equipo con más de una interfaz de red física o virtual conectada a redes diferentes. Es el candidato ideal como pivot host porque tiene conectividad simultánea con dos o más segmentos de red. En la enumeración post-explotación, verificar siempre las interfaces de red (`ip a`, `ipconfig /all`) y las rutas de red (`ip route`, `route print`) para identificar si el host comprometido tiene acceso a redes adicionales no visibles desde la máquina atacante.

---

## 📌 Tunneling en detalle

El tunneling encapsula el tráfico real dentro de otro protocolo para ocultarlo a sistemas de detección. La analogía es exacta: es como esconder una llave dentro de un peluche antes de enviarlo — quien inspecciona el paquete ve un juguete inofensivo, no la llave.

**Ejemplo práctico:** para mantener Command and Control (C2) sobre hosts comprometidos en una red monitorizada, el tráfico de C2 se encapsuló dentro de peticiones HTTP/S GET y POST con aspecto de tráfico web normal. Un analista de seguridad vería tráfico aparentemente legítimo hacia un sitio web. Solo si el paquete estaba correctamente formado según el protocolo del C2, el servidor de control lo procesaba — de lo contrario se redirigía a un sitio web real para cubrir las apariencias.

Aplicaciones comunes de tunneling en pentesting:

| Técnica | Protocolo exterior | Tráfico real encapsulado | Herramientas |
|---|---|---|---|
| SSH tunneling | SSH | TCP arbitrario | `ssh -L`, `ssh -R`, `ssh -D` |
| HTTP/S C2 | HTTP/HTTPS | Instrucciones C2 y datos de exfiltración | Cobalt Strike, Metasploit |
| DNS tunneling | DNS | Datos de exfiltración | dnscat2, iodine |
| ICMP tunneling | ICMP echo | TCP/datos arbitrarios | ptunnel-ng |
| SOCKS proxy | SOCKS4/5 | Tráfico TCP/UDP arbitrario | chisel, socat, proxychains |

---

## 📌 Conceptos de red necesarios para pivoting

Antes de trabajar con herramientas de pivoting, es fundamental tener claros los siguientes conceptos de red:

> [!info] **Port Forwarding**
> Técnica que redirige el tráfico destinado a un puerto en una máquina hacia un puerto en otra máquina diferente. Permite hacer accesible desde la máquina atacante un servicio que solo escucha en la red interna del host comprometido. Existen tres tipos principales: **Local Port Forwarding** (acceso desde el atacante a un servicio remoto), **Remote Port Forwarding** (exponer un puerto del atacante en el host comprometido) y **Dynamic Port Forwarding** (crear un proxy SOCKS que enruta todo el tráfico).

> [!info] **SOCKS Proxy**
> Protocolo de proxy de red (Socket Secure) que permite que el tráfico de cualquier aplicación sea enrutado a través de él hacia una red remota. En pivoting, se crea un proxy SOCKS en la máquina atacante que tuneliza el tráfico a través del host comprometido hacia la red interna. Combinado con `proxychains`, permite usar herramientas como `nmap`, `curl` o `crackmapexec` contra hosts de redes internas como si se tuviera conectividad directa.

> [!info] **proxychains**
> Herramienta de Linux que fuerza a cualquier aplicación TCP a enviar su tráfico a través de uno o más proxies SOCKS o HTTP configurados en `/etc/proxychains.conf`. Permite usar herramientas que no tienen soporte nativo de proxy a través de túneles establecidos con chisel, SSH o socat. La configuración define la cadena de proxies por la que pasará el tráfico — puede ser un solo proxy o una cadena de varios para pivoting multi-hop.

---

## 🔗 Relaciones / Contexto

El pivoting y tunneling son habilidades imprescindibles para cualquier evaluación de redes corporativas reales, donde la segmentación es la defensa principal contra el movimiento lateral. En el contexto del eCPPT — el siguiente objetivo en el camino de certificaciones — el pivoting es uno de los requisitos técnicos centrales del examen: se espera capacidad para encadenar múltiples pivots a través de varios segmentos de red usando herramientas como `socat`, `chisel`, SSH dynamic forwarding y `proxychains`. La diferencia entre un pentest superficial y uno que demuestra impacto real en una organización frecuentemente reside en la capacidad de cruzar la segmentación de red y alcanzar activos críticos — servidores de bases de datos, controladores de dominio, redes OT/ICS — que están deliberadamente aislados del exterior.