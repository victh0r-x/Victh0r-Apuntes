
## 🧠 Concepto clave

Sshuttle es la herramienta de pivoting SSH más transparente de todas: una vez activa, **no necesitas proxychains ni modificar ningún comando** — el tráfico hacia la red interna fluye directamente como si estuvieras conectado a esa red. Se comporta como una VPN ligera que usa SSH como transporte.

| Herramienta | Requiere proxychains | Modificar comandos | Protocolo |
|---|---|---|---|
| `ssh -D` + proxychains | Sí | Sí (`proxychains nmap...`) | SOCKS4/5 |
| chisel + proxychains | Sí | Sí | HTTP/WebSocket |
| Meterpreter autoroute | Solo para tools externas | Sí | Meterpreter |
| **sshuttle** | **No** | **No** | **SSH + iptables NAT** |

El mecanismo interno es elegante: sshuttle manipula **iptables en el atacante** para redirigir todo el tráfico TCP destinado a las subredes especificadas hacia un redirector local, que lo reenvía a través de SSH al pivot host. Para el sistema operativo y las herramientas, la red interna parece directamente accesible.

> [!info] **sshuttle** 📄 [github.com/sshuttle/sshuttle](https://github.com/sshuttle/sshuttle)
> Herramienta Python que combina SSH tunneling con manipulación de iptables para crear una VPN transparente sobre SSH. Requiere Python3 en el pivot host (no necesita instalar nada adicional en el pivot — sshuttle sube automáticamente su componente servidor con Python). Solo funciona sobre SSH — no soporta proxies HTTPS ni TOR. No soporta UDP (solo TCP). Requiere `sudo` en el atacante para manipular iptables. Es la opción más cómoda cuando se tienen credenciales SSH del pivot y se quiere trabajar con herramientas que no soportan proxychains (como algunos escáneres o clientes gráficos).

---

## 📌 Instalación

> [!tip] Instalar sshuttle en el atacante
> ```bash
> sudo apt-get install sshuttle
>
> # O directamente desde pip
> pip3 install sshuttle
>
> # O desde el repositorio
> git clone https://github.com/sshuttle/sshuttle.git
> cd sshuttle && sudo python3 setup.py install
> ```
> sshuttle solo se instala en el **atacante** — en el pivot host no se instala nada. sshuttle sube automáticamente su componente servidor mediante SSH usando Python3, que en la mayoría de servidores Linux ya está presente.

---

## 📌 Uso básico

### Escenario
```
[Atacante 10.10.14.18]
    │  sshuttle gestiona iptables localmente
    │  SSH hacia Ubuntu pivot
    ▼
[Ubuntu pivot 10.129.202.64]
    │  sshuttle server (subido automáticamente vía SSH)
    ▼
[Red interna 172.16.5.0/23]
    └── Windows DC01 172.16.5.19
```

> [!tip] Lanzar sshuttle con autenticación por contraseña
> ```bash
> sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
> ```
> `sudo` — Necesario para que sshuttle pueda modificar las reglas de iptables en el atacante.
> `-r ubuntu@10.129.202.64` — **Remote**: usuario y dirección del pivot host SSH. Es el único parámetro de conexión necesario — sshuttle gestiona el resto automáticamente.
> `172.16.5.0/23` — Subred a enrutar a través del pivot. Todo el tráfico TCP hacia esta red pasará por el túnel SSH. Se pueden especificar múltiples subredes separadas por espacios: `172.16.5.0/23 10.10.10.0/24`.
> `-v` — Verbose: muestra los comandos iptables que sshuttle ejecuta y el estado de la conexión.

> [!tip] Lanzar sshuttle con clave privada SSH
> ```bash
> sudo sshuttle -r ubuntu@10.129.202.64 --ssh-cmd "ssh -i ~/.ssh/id_rsa" 172.16.5.0/23
> ```
> `--ssh-cmd` — Permite pasar el comando SSH completo con opciones adicionales, como especificar la clave privada con `-i`.

> [!tip] Lanzar sshuttle en background
> ```bash
> sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v --daemon --pidfile /tmp/sshuttle.pid
>
> # Para terminarlo:
> sudo kill $(cat /tmp/sshuttle.pid)
> ```
> `--daemon` — Ejecuta sshuttle en background como daemon.
> `--pidfile` — Guarda el PID del proceso para poder terminarlo limpiamente después.

---

## 📌 Qué hace sshuttle con iptables

Al lanzarse, sshuttle crea automáticamente reglas NAT en iptables del atacante. El output con `-v` lo muestra claramente:
```bash
fw: iptables -w -t nat -N sshuttle-12300          # Crea cadena NAT propia
fw: iptables -w -t nat -I OUTPUT 1 -j sshuttle-12300      # Intercepta tráfico saliente
fw: iptables -w -t nat -I PREROUTING 1 -j sshuttle-12300  # Intercepta tráfico entrante
fw: iptables -w -t nat -A sshuttle-12300 -j RETURN --dest 127.0.0.1/32 -p tcp  # Excluye loopback
fw: iptables -w -t nat -A sshuttle-12300 -j REDIRECT --dest 172.16.5.0/32 -p tcp --to-ports 12300
# ^ Redirige todo TCP a 172.16.5.0/23 al puerto local 12300 (el redirector de sshuttle)
```

Al terminar sshuttle (Ctrl+C), **limpia automáticamente todas las reglas** que creó. No deja residuos en iptables.

---

## 📌 Usando herramientas directamente — sin proxychains

La ventaja principal de sshuttle respecto a todas las otras técnicas de pivoting: los comandos son exactamente iguales a si estuvieras en la misma red que el objetivo.

> [!tip] Escanear con nmap directamente — sin proxychains
> ```bash
> # Con proxychains (otros métodos)
> proxychains nmap -Pn -sT -p3389 172.16.5.19
>
> # Con sshuttle — exactamente igual que si fuera una red local
> sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn
> ```
> `-sT` — TCP Connect Scan. Aunque con sshuttle no es estrictamente obligatorio como con proxychains, sigue siendo más fiable que SYN scan a través de un túnel.
> `-Pn` — Sin ping previo. Recomendable para hosts Windows que bloquean ICMP.

El output del nmap con sshuttle devuelve resultados completos incluyendo detección de servicio, versión y scripts NSE — algo imposible con proxychains que solo soporta TCP Connect:
```
PORT     STATE SERVICE       VERSION
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: INLANEFREIGHT
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: inlanefreight.local
```

> [!tip] RDP directo al host interno sin proxychains
> ```bash
> xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
> ```

> [!tip] Enumerar SMB directamente en la red interna
> ```bash
> crackmapexec smb 172.16.5.0/24
> smbclient //172.16.5.19/C$ -U victor
> ```

> [!tip] Acceder a servicios web internos directamente en el navegador
> ```bash
> firefox http://172.16.5.100:8080
> curl http://172.16.5.100/api/v1/users
> ```

---

## 📌 Limitaciones de sshuttle

> [!important] Limitaciones técnicas a tener en cuenta
> - **Solo TCP** — No soporta UDP. Herramientas que usan UDP (DNS lookups nativos, algunos escáneres) no funcionarán a través del túnel.
> - **Solo SSH** — A diferencia de chisel o socat, no funciona sobre HTTPS ni otros transportes. Requiere acceso SSH al pivot con credenciales válidas.
> - **Requiere Python3 en el pivot** — sshuttle sube automáticamente su componente servidor usando Python. Si el pivot no tiene Python3, falla. En servidores Linux modernos esto rara vez es un problema.
> - **Requiere sudo en el atacante** — La manipulación de iptables requiere privilegios de root. En sistemas donde no se tiene sudo, no es viable.
> - **No funciona en Windows** — ni como cliente ni en pivots Windows. Solo Linux/macOS en el lado del atacante.

---

## 📌 Comparativa final de métodos de pivoting SSH

| Método | Proxychains necesario | UDP | Instalación en pivot | Sudo atacante | Windows pivot |
|---|---|---|---|---|---|
| `ssh -D` + proxychains | Sí | No | No | No | Solo cliente SSH |
| `ssh -R` | No (port forward) | No | No | No | Solo cliente SSH |
| chisel SOCKS | Sí | Sí (SOCKS5) | Binario chisel | No | Sí |
| socat redirector | No (puerto específico) | Sí | Binario socat | No | No |
| Meterpreter + autoroute | Sí (tools externas) | No | Payload Meterpreter | No | Sí |
| plink + Proxifier | No (Proxifier) | No | plink.exe | No | Solo Windows |
| **sshuttle** | **No** | **No** | **Python3 (ya presente)** | **Sí** | **No** |

---

## 🔗 Relaciones / Contexto

Sshuttle es la herramienta de pivoting SSH más cómoda para trabajar en assessments donde el pivot host tiene SSH y Python3 — que cubre la inmensa mayoría de servidores Linux modernos. Su gran ventaja es la **transparencia total**: no hay que recordar añadir `proxychains` a cada comando, no hay que configurar nada en las herramientas, no hay que preocuparse por si la herramienta es compatible con proxychains. Se lanza una vez y se olvida. La principal limitación es UDP — si el assessment requiere enumerar servicios UDP internos (DNS, SNMP, NetBIOS) habrá que combinarla con otra técnica como chisel o Meterpreter. En el contexto del eCPPT, sshuttle es ideal para fases de enumeración extensa de redes internas donde se quiere máxima comodidad, mientras que chisel/socat son preferibles cuando no hay SSH o cuando se necesita operar en modo más sigiloso y controlado.