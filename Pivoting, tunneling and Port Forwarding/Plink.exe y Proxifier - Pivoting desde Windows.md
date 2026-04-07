
## 🧠 Concepto clave

Hasta ahora todo el pivoting se ha ejecutado desde un **attack host Linux**. Pero hay dos escenarios donde se trabaja desde Windows:

- **Windows como pivot comprometido** — se gana acceso a un host Windows y se necesita usarlo como pivot sin subir herramientas externas (OPSEC).
- **Windows como attack host personal** — el pentester trabaja desde una máquina Windows en lugar de Kali/Parrot.

En ambos casos, el equivalente Windows de `ssh -D` es **Plink**, y el equivalente de proxychains es **Proxifier**.

| Herramienta Linux | Equivalente Windows | Función |
|---|---|---|
| `ssh -D 9050` | `plink -ssh -D 9050` | Dynamic port forward / SOCKS proxy |
| `proxychains` | **Proxifier** | Redirigir tráfico de aplicaciones a través de SOCKS |
| `ssh -L` | `plink -L` | Local port forward |
| `ssh -R` | `plink -R` | Remote port forward |

---

## 📌 Plink — PuTTY Link

> [!info] **plink.exe** 📄 [PuTTY Download Page](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
> Herramienta de línea de comandos SSH incluida en el paquete PuTTY para Windows. Es el equivalente exacto al comando `ssh` de Linux — mismos flags, mismas funcionalidades — pero en formato `.exe` para Windows. No requiere instalación: es un binario standalone que puede ejecutarse directamente desde cualquier ruta. Admite port forwarding local (`-L`), remoto (`-R`) y dinámico (`-D`), transferencia de archivos con `pscp.exe` (equivalente a `scp`), y ejecución de comandos remotos. Antes de Windows 10 Fall 2018, era la única forma de tener SSH en Windows sin instalar software adicional.

> [!tip] Crear dynamic port forward SOCKS con plink desde Windows
> ```cmd
> plink -ssh -D 9050 ubuntu@10.129.15.50
> ```
> `-ssh` — Forzar el protocolo SSH (plink también soporta Telnet y Raw — este flag evita ambigüedades).
> `-D 9050` — Dynamic port forward: crea un proxy SOCKS en `127.0.0.1:9050` del host Windows local. Idéntico a `ssh -D 9050`.
> `ubuntu@10.129.15.50` — Usuario y dirección del SSH server (el pivot host Ubuntu).

> [!important] La primera vez que plink se conecta a un host, muestra un prompt de confianza del host key que **bloquea la ejecución** hasta que el usuario responde `y`. En scripts o uso automatizado esto es un problema. Para aceptar automáticamente sin interacción: `echo y | plink -ssh -D 9050 ubuntu@10.129.15.50`. En entornos de producción, lo correcto es añadir el host key previamente con `plink -hostkey FINGERPRINT`.

> [!tip] Port forwarding local con plink — equivalente a ssh -L
> ```cmd
> plink -ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.15.50
> ```
> `-L 3389:172.16.5.19:3389` — Local port forward: `localhost:3389` en el Windows atacante → `172.16.5.19:3389` a través del Ubuntu pivot. Permite conectar `mstsc.exe` a `127.0.0.1:3389`.

> [!tip] Remote port forwarding con plink — equivalente a ssh -R
> ```cmd
> plink -ssh -R 8080:0.0.0.0:80 ubuntu@10.129.15.50
> ```
> `-R 8080:0.0.0.0:80` — El Ubuntu pivot escucha en `:8080` y reenvía al atacante Windows en `:80`.

> [!tip] Ejecutar plink en background sin bloquear la terminal
> ```cmd
> start /B plink -ssh -D 9050 ubuntu@10.129.15.50
> ```
> `start /B` — Lanza el proceso en background (equivalente a `&` en bash). La terminal queda libre mientras plink mantiene el túnel activo.

---

## 📌 Proxifier — Proxy para aplicaciones Windows

> [!info] **Proxifier** 📄 [proxifier.com](https://www.proxifier.com/)
> Herramienta Windows que intercepta las conexiones de red de cualquier aplicación del sistema y las redirige a través de un proxy SOCKS o HTTPS. El equivalente Windows de `proxychains`. La diferencia clave con proxychains es que Proxifier funciona a nivel de sistema operativo mediante un driver — **no requiere que la aplicación soporte proxies** — mientras que proxychains requiere que la aplicación use llamadas estándar de libc que pueda interceptar. Soporta SOCKS4, SOCKS5, HTTPS y encadenamiento de proxies.

La configuración para usarlo con plink se hace en dos pasos desde la interfaz gráfica:

**Paso 1 — Crear el perfil del proxy SOCKS:**

Ir a `Profile → Proxy Servers → Add` y configurar:
- Server: `127.0.0.1`
- Port: `9050`
- Protocol: `SOCKS5` (o SOCKS4 si plink usó versión 4)

**Paso 2 — Definir las reglas de routing:**

Ir a `Profile → Proxification Rules` y añadir una regla que envíe el tráfico deseado a través del proxy. Por ejemplo, redirigir todo el tráfico de `mstsc.exe` al proxy SOCKS creado por plink.

---

## 📌 Flujo completo — RDP a host interno desde Windows attack host
```
[Windows Attack Host]
    │  plink crea SOCKS en 127.0.0.1:9050
    │  Proxifier redirige mstsc.exe → 127.0.0.1:9050
    ▼
[Ubuntu pivot 10.129.15.50]
    │  SSH forward
    ▼
[Windows target 172.16.5.19:3389]
```

**Paso 1 — Tunnel SSH con plink:**

> [!tip] Crear el proxy SOCKS desde Windows con plink
> ```cmd
> echo y | plink -ssh -D 9050 ubuntu@10.129.15.50
> ```

**Paso 2 — Configurar Proxifier** con `127.0.0.1:9050` como servidor SOCKS5.

**Paso 3 — Lanzar RDP hacia el host interno:**

> [!tip] Conectar por RDP al host interno a través de Proxifier + plink
> ```cmd
> mstsc.exe /v:172.16.5.19
> ```
> Con Proxifier activo, `mstsc.exe` intenta conectar directamente a `172.16.5.19:3389` — Proxifier intercepta esa conexión y la redirige al proxy SOCKS de plink, que la reenvía al Ubuntu pivot, que finalmente la entrega al Windows interno. Todo transparente para `mstsc.exe`.

---

## 📌 Plink como herramienta de "living off the land"

El valor principal de plink en un engagement no es para el attack host del pentester — para eso hay mejores opciones. Su valor está en el escenario **pivot desde un Windows comprometido**:

- El host Windows comprometido ya tiene plink instalado (entorno corporativo con PuTTY presente).
- No se pueden subir herramientas adicionales sin alertar al AV/EDR.
- plink.exe es un binario legítimo firmado por Simon Tatham — menos probable de ser flagueado.
- Se puede encontrar en shares de red corporativos junto con otras herramientas de administración.

> [!important] Plink es software legítimo de administración de sistemas, pero su uso en un proceso de pivoting generará eventos de red anómalos (conexión SSH saliente desde un host que normalmente no hace SSH). En entornos con SIEM activo, una conexión SSH desde un workstation de usuario hacia una IP externa es una alerta de alta prioridad. Evaluar el riesgo de detección antes de usarlo.

---

## 🔗 Relaciones / Contexto

Plink + Proxifier es la combinación Windows nativa equivalente a `ssh -D` + proxychains en Linux. Cubre el escenario específico de assessments donde el attack host es Windows o donde el pivot comprometido es Windows y no se puede (o no se quiere) subir chisel o socat. En la práctica moderna, Windows 10/11 ya incluye un cliente OpenSSH nativo (`ssh.exe` en `C:\Windows\System32\OpenSSH\`), 📄 [documentación oficial](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_overview) por lo que en sistemas actualizados `ssh -D 9050 ubuntu@pivot` funciona directamente desde CMD/PowerShell sin necesitar plink. Plink sigue siendo relevante en sistemas Windows más antiguos (pre-2018) o en entornos donde OpenSSH no está habilitado.