# 🗂️ John the Ripper — Guía Maestra Completa

## 🧠 Concepto clave

**John the Ripper** (comúnmente llamado "John" o "JtR") es una de las herramientas de crackeo de contraseñas y hashes más veteranas y completas del mundo de la seguridad ofensiva. Desarrollada originalmente por Openwall Project, existe en dos versiones principales:

| Versión | Descripción |
|---|---|
| **John the Ripper (Community)** | Versión original open source, instalada por defecto en Kali Linux como `john` |
| **John the Ripper Jumbo** | Fork mantenido por la comunidad con soporte para cientos de formatos adicionales — es la versión recomendada. Se instala desde GitHub o con `apt install john` en Kali (ya incluye Jumbo) |

La herramienta permite atacar contraseñas usando tres modos principales: **wordlist** (diccionario), **incremental** (fuerza bruta pura) y **single crack** (basado en información del usuario). Soporta más de 400 formatos de hash distintos y tiene un ecosistema de herramientas auxiliares (`*2john`) para convertir prácticamente cualquier archivo protegido a un formato que John pueda procesar.

> [!info] **Hashcat vs John the Ripper**
> Ambas son herramientas de crackeo, pero con filosofías distintas. **Hashcat** usa la GPU y es enormemente más rápida para hashes simples (MD5, SHA1, NTLM). **John** es más versátil, corre en CPU, soporta más formatos complejos out-of-the-box y tiene el ecosistema `*2john` integrado. En CTFs, John suele ser la opción más rápida de usar por su comodidad. Para crackeo serio con grandes wordlists, Hashcat en GPU gana en velocidad.

---

## 📌 Instalación y archivos importantes

> [!tip] Instalar John the Ripper Jumbo desde repositorios
> ```bash
> sudo apt install john
> ```

> [!tip] Instalar John the Ripper Jumbo desde código fuente (máxima compatibilidad)
> ```bash
> git clone https://github.com/openwall/john
> cd john/src
> ./configure && make -s clean && make -sj4
> cd ../run
> ./john --list=formats
> ```
> `./configure` — Configura la compilación detectando las capacidades del sistema (OpenMP, OpenCL, etc.).
> `make -sj4` — Compila usando 4 núcleos en paralelo.
> `./john` — El binario compilado estará en `john/run/`.

### Archivos y directorios clave

| Archivo | Ubicación típica | Función |
|---|---|---|
| `john` | `/usr/sbin/john` | Binario principal |
| `john.pot` | `~/.john/john.pot` | **Potfile** — almacena todas las contraseñas ya crackeadas para no repetir trabajo |
| `john.log` | `~/.john/john.log` | Log de sesiones anteriores |
| `password.lst` | `/usr/share/john/password.lst` | Wordlist por defecto incluida con John |
| `john.conf` | `/etc/john/john.conf` | Archivo de configuración global con reglas, modos y charset |

> [!important] El **potfile** (`john.pot`) es crítico — John no volverá a crackear hashes que ya estén en él. Si reinicias un ataque y no obtienes resultados, comprueba primero si el hash ya está crackeado con `john --show`.

---

## 🔧 Herramientas auxiliares `*2john`

Estas utilidades convierten archivos protegidos por contraseña a un formato de hash que John puede procesar. Son el primer paso en cualquier ataque a archivos cifrados.

### Listado completo de conversores

| Herramienta | Qué convierte | Ejemplo de uso |
|---|---|---|
| `ssh2john` | Claves privadas SSH con passphrase (`id_rsa`, `id_ed25519`, `id_ecdsa`) | `ssh2john id_rsa > hash` |
| `zip2john` | Archivos ZIP protegidos con contraseña | `zip2john archivo.zip > hash` |
| `rar2john` | Archivos RAR protegidos con contraseña | `rar2john archivo.rar > hash` |
| `7z2john` | Archivos 7-Zip protegidos | `7z2john archivo.7z > hash` |
| `pdf2john` | PDFs protegidos con contraseña | `pdf2john archivo.pdf > hash` |
| `office2john` | Documentos Office 2007+ (.docx, .xlsx, .pptx) cifrados | `office2john doc.docx > hash` |
| `keepass2john` | Bases de datos KeePass (.kdbx) | `keepass2john db.kdbx > hash` |
| `bitlocker2john` | Volúmenes BitLocker | `bitlocker2john -i disco.img > hash` |
| `dmg2john` | Imágenes de disco macOS (.dmg) cifradas | `dmg2john imagen.dmg > hash` |
| `gpg2john` | Claves GPG/PGP privadas cifradas | `gpg2john clave.gpg > hash` |
| `truecrypt2john` | Volúmenes TrueCrypt/VeraCrypt | `truecrypt2john -t volumen > hash` |
| `keychain2john` | Keychains de macOS | `keychain2john archivo.keychain > hash` |
| `pwsafe2john` | Bases de datos Password Safe | `pwsafe2john db.psafe3 > hash` |
| `hccapx2john` | Capturas WPA/WPA2 (handshakes WiFi) | `hccapx2john captura.hccapx > hash` |
| `wpapcap2john` | Capturas pcap de handshakes WiFi | `wpapcap2john captura.pcap > hash` |
| `putty2john` | Claves privadas PuTTY (.ppk) | `putty2john clave.ppk > hash` |
| `krb2john` | Tickets Kerberos (AS-REP, TGS) | `krb2john ticket.kirbi > hash` |
| `mozilla2john` | Bases de datos de contraseñas de Firefox | `mozilla2john ~/.mozilla/firefox/perfil/ > hash` |
| `lastpass2john` | Exportaciones de LastPass | `lastpass2john export.xml > hash` |
| `1password2john` | Vaults de 1Password | `1password2john vault.agilekeychain > hash` |
| `hashcat2john` | Convierte hashes de formato Hashcat a John | `hashcat2john hash.hc > hash` |

> [!tip] Usar ssh2john para extraer el hash de una clave SSH cifrada
> ```bash
> ssh2john id_rsa > hash_ssh
> ssh2john id_ed25519 >> hash_ssh
> ```
> `id_rsa` — Clave privada SSH cifrada. Si no tiene passphrase, ssh2john lo indicará y no generará hash.
> `>` — Redirige la salida al archivo `hash_ssh`.
> `>>` — Append, añade al archivo sin sobreescribir — útil para combinar varios hashes.

> [!tip] Usar zip2john para extraer el hash de un ZIP protegido
> ```bash
> zip2john archivo.zip > hash_zip
> ```
> Genera un hash por cada archivo comprimido dentro del ZIP que esté cifrado.

> [!tip] Usar keepass2john para extraer el hash de una base de datos KeePass
> ```bash
> keepass2john database.kdbx > hash_keepass
> ```
> Si la base de datos usa un keyfile además de contraseña: `keepass2john -k keyfile database.kdbx > hash_keepass`

---

## 🔍 Identificación de hashes

Antes de atacar un hash, hay que saber qué tipo es para usar el formato correcto con John.

### Método 1 — john --list=formats

> [!tip] Listar todos los formatos de hash soportados por John
> ```bash
> john --list=formats
> john --list=formats | grep -i md5
> john --list=formats | grep -i sha
> john --list=formats | grep -i ntlm
> ```
> `--list=formats` — Muestra todos los formatos soportados con sus nombres exactos (los que se pasan a `--format=`).
> `grep -i md5` — Filtra por nombre de algoritmo, ignorando mayúsculas.

### Método 2 — hash-identifier

> [!info] **hash-identifier**
> Herramienta incluida en Kali que analiza la longitud y estructura de un hash para identificar qué algoritmo lo generó. No es perfecta — muchos algoritmos producen hashes de la misma longitud — pero es un buen punto de partida. Para casos ambiguos, `hashid` o `hashcat --example-hashes` dan información más detallada.

> [!tip] Identificar el tipo de un hash desconocido
> ```bash
> hash-identifier
> hash-identifier "5f4dcc3b5aa765d61d8327deb882cf99"
> ```
> Tras ejecutarlo, pegar el hash cuando lo solicite. Devuelve una lista de posibles algoritmos ordenados por probabilidad.

### Método 3 — hashid

> [!tip] Identificar hash con hashid (más preciso que hash-identifier)
> ```bash
> hashid "5f4dcc3b5aa765d61d8327deb882cf99"
> hashid -m "5f4dcc3b5aa765d61d8327deb882cf99"
> ```
> `-m` — Muestra también el modo equivalente de Hashcat junto al nombre del algoritmo — muy útil para decidir entre John y Hashcat.

### Estructura de un hash para John

Los hashes para John tienen este formato general:
```
usuario:$formato$parametros$salt$hash
```

Algunos ejemplos reales:

| Tipo | Ejemplo de formato |
|---|---|
| MD5 | `5f4dcc3b5aa765d61d8327deb882cf99` |
| SHA1 | `5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8` |
| SHA256 | `5e884898da28047151d0e56f8dc629277...` |
| NTLM | `8846f7eaee8fb117ad06bdd830b7586c` |
| bcrypt | `$2b$12$Xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| SHA512crypt (Linux) | `$6$salt$hash` |
| MD5crypt (Linux) | `$1$salt$hash` |
| SHA256crypt (Linux) | `$5$salt$hash` |
| SSH (ed25519) | `$sshng$0$16$...` |
| KeePass | `$keepass$*2*...` |
| ZIP | `$pkzip2$...` |

> [!tip] Identificar el formato de un hash de /etc/shadow directamente
> ```bash
> cat /etc/shadow | grep -v "!" | grep -v "*"
> ```
> Las líneas con `$1$` son MD5crypt, `$5$` SHA256crypt, `$6$` SHA512crypt. John los detecta automáticamente si no se especifica `--format`.

---

## ⚔️ Modos de ataque

### Modo 1 — Wordlist (Diccionario)

El modo más común. John prueba cada palabra de una lista como posible contraseña. Se puede combinar con **reglas** para transformar las palabras (añadir números, cambiar mayúsculas, sustituir caracteres, etc.).

> [!tip] Ataque por diccionario básico
> ```bash
> john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```
> `hash.txt` — Archivo con el hash o hashes a crackear.
> `--wordlist=` — Ruta a la wordlist. RockYou (14M+ contraseñas) es la más usada en CTFs.

> [!tip] Ataque por diccionario con reglas de transformación
> ```bash
> john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules
> john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules=Jumbo
> john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules=KoreLogic
> ```
> `--rules` — Aplica el conjunto de reglas por defecto (definidas en `john.conf`). Transforma cada palabra: añade números al final, cambia letras por símbolos (`a→@`, `e→3`), capitaliza, etc.
> `--rules=Jumbo` — Conjunto de reglas extendido incluido en la versión Jumbo — mucho más agresivo.
> `--rules=KoreLogic` — Reglas de KoreLogic, especialmente efectivas para contraseñas corporativas.

> [!tip] Ataque por diccionario especificando el formato del hash
> ```bash
> john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=NT
> john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=sha512crypt
> john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=bcrypt
> ```
> `--format=` — Especifica el algoritmo del hash. Si John no lo detecta automáticamente, es obligatorio indicarlo. Usar exactamente el nombre que devuelve `--list=formats`.

---

### Modo 2 — Incremental (Fuerza Bruta)

John prueba todas las combinaciones posibles de caracteres. Extremadamente lento para contraseñas largas, pero garantiza encontrar la contraseña si se le da tiempo suficiente. Solo recomendable para contraseñas cortas (≤6 caracteres).

> [!tip] Ataque de fuerza bruta incremental básico
> ```bash
> john hash.txt --incremental
> john hash.txt --incremental=Digits
> john hash.txt --incremental=Alpha
> john hash.txt --incremental=Alnum
> ```
> `--incremental` — Activa el modo incremental con el charset por defecto (todos los caracteres ASCII imprimibles).
> `--incremental=Digits` — Solo dígitos del 0 al 9. Para PINs o contraseñas numéricas.
> `--incremental=Alpha` — Solo letras (mayúsculas y minúsculas).
> `--incremental=Alnum` — Letras y números, sin símbolos.

> [!tip] Limitar la longitud de la fuerza bruta
> ```bash
> john hash.txt --incremental --min-length=4 --max-length=6
> ```
> `--min-length=4` — No prueba combinaciones de menos de 4 caracteres.
> `--max-length=6` — No prueba combinaciones de más de 6 caracteres. Útil para acotar el espacio de búsqueda.

---

### Modo 3 — Single Crack

John genera variaciones basadas en la información del campo de usuario del hash (nombre de usuario, GECOS, etc.). Es el modo más rápido y se recomienda ejecutarlo primero siempre.

> [!tip] Ataque en modo Single Crack
> ```bash
> john hash.txt --single
> john hash.txt --single --format=sha512crypt
> ```
> `--single` — Activa el modo single crack. John extrae el nombre de usuario del hash y genera variantes: mayúsculas, números añadidos, inversión, leet speak, etc. Muy efectivo cuando la contraseña está basada en el nombre del usuario.

---

### Modo 4 — Pipe (desde stdin)

Permite alimentar a John con contraseñas desde la salida estándar de otro comando — útil para usar herramientas de generación de wordlists como `crunch` o `cupp`.

> [!tip] Usar John con wordlist generada dinámicamente desde crunch
> ```bash
> crunch 6 6 0123456789 | john hash.txt --stdin --format=NT
> ```
> `crunch 6 6 0123456789` — Genera todas las combinaciones de 6 dígitos.
> `--stdin` — John lee las contraseñas desde stdin en lugar de un archivo.

---

## 🎯 Formatos de hash más comunes y cómo atacarlos

### Hashes de sistema Linux (/etc/shadow)

> [!tip] Crackear hashes de /etc/shadow
> ```bash
> # Combinar /etc/passwd y /etc/shadow primero
> unshadow /etc/passwd /etc/shadow > combined.txt
> john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```
> `unshadow` — Herramienta incluida con John que combina `/etc/passwd` y `/etc/shadow` en el formato que John espera. Es necesario para que John pueda asociar el hash con el nombre de usuario.

### Hashes NTLM y LM (Windows)

> [!tip] Crackear hashes NTLM extraídos de Windows (formato secretsdump)
> ```bash
> # Formato de secretsdump: usuario:RID:LMhash:NThash:::
> echo "Administrator:500:aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c:::" > ntlm.txt
> john ntlm.txt --format=NT --wordlist=/usr/share/wordlists/rockyou.txt
> ```
> `--format=NT` — Especifica el formato NTLM (también funciona `--format=ntlm`).
> El hash LM (`aad3b435...`) es el hash vacío — indica que LM está deshabilitado, solo hay NTLM.

> [!tip] Crackear solo la parte NT de un dump de secretsdump
> ```bash
> grep -oP '(?<=:::)[^:]*' secretsdump_output.txt > nt_hashes.txt
> # O extraer directamente la columna del NT hash
> cut -d: -f4 secretsdump_output.txt > nt_hashes.txt
> john nt_hashes.txt --format=NT --wordlist=/usr/share/wordlists/rockyou.txt
> ```

### Hashes de Kerberos (AS-REP Roasting y Kerberoasting)

> [!tip] Crackear hashes AS-REP (AS-REP Roasting)
> ```bash
> john asrep_hashes.txt --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt
> ```
> `--format=krb5asrep` — Formato específico para hashes capturados con AS-REP Roasting (usuarios con `DONT_REQUIRE_PREAUTH`).

> [!tip] Crackear hashes TGS (Kerberoasting)
> ```bash
> john tgs_hashes.txt --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt
> ```
> `--format=krb5tgs` — Formato para tickets TGS capturados con Kerberoasting.

### Hashes SSH

> [!tip] Crackear passphrase de clave SSH (flujo completo)
> ```bash
> ssh2john id_rsa > hash_ssh.txt
> john hash_ssh.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```

### Archivos ZIP

> [!tip] Crackear contraseña de ZIP (flujo completo)
> ```bash
> zip2john archivo.zip > hash_zip.txt
> john hash_zip.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```

### Archivos RAR

> [!tip] Crackear contraseña de RAR (flujo completo)
> ```bash
> rar2john archivo.rar > hash_rar.txt
> john hash_rar.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```

### KeePass (.kdbx)

> [!tip] Crackear contraseña maestra de KeePass (flujo completo)
> ```bash
> keepass2john database.kdbx > hash_keepass.txt
> john hash_keepass.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```

### PDFs y documentos Office

> [!tip] Crackear contraseña de PDF o documento Office (flujo completo)
> ```bash
> pdf2john documento.pdf > hash_pdf.txt
> office2john documento.docx > hash_office.txt
> john hash_pdf.txt --wordlist=/usr/share/wordlists/rockyou.txt
> john hash_office.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```

### Hashes bcrypt

> [!tip] Crackear hashes bcrypt
> ```bash
> echo '$2b$12$HASH_COMPLETO_AQUI' > hash_bcrypt.txt
> john hash_bcrypt.txt --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt
> ```
> `--format=bcrypt` — Bcrypt es muy lento por diseño (tiene un cost factor). El crackeo es significativamente más lento que MD5 o SHA1. Para bcrypt, Hashcat con GPU es mucho más recomendable.

### Hashes MD5 y SHA puros

> [!tip] Crackear hashes MD5, SHA1, SHA256 en texto plano
> ```bash
> echo "5f4dcc3b5aa765d61d8327deb882cf99" > hash_md5.txt
> john hash_md5.txt --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt

> echo "5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8" > hash_sha1.txt
> john hash_sha1.txt --format=Raw-SHA1 --wordlist=/usr/share/wordlists/rockyou.txt

> john hash_sha256.txt --format=Raw-SHA256 --wordlist=/usr/share/wordlists/rockyou.txt
> ```
> `Raw-MD5`, `Raw-SHA1`, `Raw-SHA256` — Formatos para hashes sin salt. El prefijo `Raw-` diferencia los hashes puros de las variantes con salt como `md5crypt`.

### Handshakes WiFi WPA/WPA2

> [!tip] Crackear handshake WPA/WPA2 capturado con airodump-ng
> ```bash
> # Convertir captura .cap a formato John
> wpapcap2john captura.cap > hash_wifi.txt
> john hash_wifi.txt --wordlist=/usr/share/wordlists/rockyou.txt
> ```

---

## 🔧 Opciones y parámetros generales

### Gestión de sesiones

Una sesión en John es un ataque con nombre propio que puede pausarse y reanudarse. Imprescindible para ataques largos.

> [!tip] Iniciar un ataque con nombre de sesión
> ```bash
> john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --session=mi_ataque
> ```
> `--session=mi_ataque` — Asigna un nombre a la sesión. John guardará el progreso en `~/.john/mi_ataque.rec`.

> [!tip] Pausar y reanudar una sesión
> ```bash
> # Pausar: Ctrl+C durante la ejecución
> # Reanudar:
> john --restore=mi_ataque
> john --restore  # reanuda la última sesión
> ```
> `--restore=mi_ataque` — Reanuda la sesión guardada con ese nombre desde el punto donde se pausó.

> [!tip] Ver el estado de una sesión en curso (desde otra terminal)
> ```bash
> john --status
> john --status=mi_ataque
> ```
> Muestra el progreso actual: velocidad (c/s = candidates per second), porcentaje completado y tiempo estimado.

### Ver contraseñas crackeadas

> [!tip] Mostrar todas las contraseñas ya crackeadas del potfile
> ```bash
> john hash.txt --show
> john hash.txt --show --format=NT
> ```
> `--show` — Lee el potfile y muestra las contraseñas crackeadas para los hashes del archivo especificado. Si no muestra nada, el hash no está en el potfile aún — no significa que no se haya crackeado nunca, puede que el potfile haya sido borrado.

> [!tip] Ver el contenido completo del potfile
> ```bash
> cat ~/.john/john.pot
> ```

### Opciones de rendimiento

> [!tip] Controlar el uso de CPU y núcleos
> ```bash
> john hash.txt --wordlist=rockyou.txt --fork=4
> john hash.txt --wordlist=rockyou.txt --node=1/4
> ```
> `--fork=4` — Lanza 4 procesos en paralelo, uno por núcleo. Mejora significativamente la velocidad en sistemas multinúcleo.
> `--node=1/4` — Para distribuir el trabajo entre varias máquinas: este nodo procesa 1/4 del espacio de búsqueda.

> [!tip] Limitar el tiempo de ejecución
> ```bash
> john hash.txt --wordlist=rockyou.txt --max-run-time=3600
> ```
> `--max-run-time=3600` — Detiene John automáticamente tras 3600 segundos (1 hora). Útil para ataques programados.

### Opciones de wordlist avanzadas

> [!tip] Usar múltiples wordlists combinadas
> ```bash
> cat wordlist1.txt wordlist2.txt wordlist3.txt | john hash.txt --stdin
> ```

> [!tip] Generar wordlist basada en el objetivo con CUPP y pasarla a John
> ```bash
> python3 cupp.py -i
> john hash.txt --wordlist=victima.txt --rules=Jumbo
> ```
> CUPP (Common User Passwords Profiler) genera una wordlist personalizada basada en datos personales del objetivo (nombre, fecha de nacimiento, pareja, mascota, etc.).

---

## 📏 Reglas (Rules) — Transformación de wordlists

Las reglas son uno de los puntos fuertes de John. Permiten transformar cada palabra de la wordlist de formas sistemáticas para cubrir variantes comunes de contraseñas.

Las reglas se definen en `john.conf` bajo secciones como `[List.Rules:Wordlist]`. Algunos ejemplos de transformaciones:

| Regla | Efecto | Ejemplo |
|---|---|---|
| `c` | Capitaliza la primera letra | `password` → `Password` |
| `u` | Convierte todo a mayúsculas | `password` → `PASSWORD` |
| `l` | Convierte todo a minúsculas | `PASSWORD` → `password` |
| `r` | Invierte la palabra | `password` → `drowssap` |
| `d` | Duplica la palabra | `pass` → `passpass` |
| `$1` | Añade `1` al final | `password` → `password1` |
| `^!` | Añade `!` al principio | `password` → `!password` |
| `sa@` | Sustituye `a` por `@` | `password` → `p@ssword` |
| `se3` | Sustituye `e` por `3` | `password` → `passw0rd` |

> [!tip] Crear reglas personalizadas en john.conf
> ```bash
> # Añadir al final de /etc/john/john.conf o ~/.john/john.conf:
> [List.Rules:MiRegla]
> cAz"[0-9]"
> Az"[!@#$]"
> ```
> ```bash
> john hash.txt --wordlist=rockyou.txt --rules=MiRegla
> ```

> [!tip] Ver todas las reglas disponibles
> ```bash
> john --list=rules
> ```

---

## 🔑 Wordlists recomendadas

| Wordlist | Ubicación / Fuente | Cuándo usar |
|---|---|---|
| **rockyou.txt** | `/usr/share/wordlists/rockyou.txt` | Primera opción siempre — 14M contraseñas reales |
| **rockyou-75.txt** | Versión reducida de RockYou | Para ataques rápidos iniciales |
| **SecLists** | `/usr/share/seclists/Passwords/` | Gran variedad de listas especializadas |
| **kaonashi** | GitHub | Excelente para hashes NTLM |
| **weakpass** | weakpass.com | Colecciones enormes de contraseñas reales |
| **hashesorg** | hashes.org | Contraseñas de brechas reales |
| **Custom CUPP** | Generada con CUPP | Cuando se conoce info personal del objetivo |
| **Mentalist** | Aplicación GUI | Generación de wordlists basadas en patrones |

---

## 🔄 Flujos completos por escenario

### Escenario 1 — Clave SSH con passphrase
```bash
# 1. Convertir la clave al formato John
ssh2john id_rsa > hash_ssh.txt

# 2. Ataque con diccionario
john hash_ssh.txt --wordlist=/usr/share/wordlists/rockyou.txt

# 3. Ver resultado
john hash_ssh.txt --show
```

### Escenario 2 — Hash NTLM de Windows (post-explotación)
```bash
# 1. Extraer solo los NT hashes del output de secretsdump
grep -v ":::" secretsdump.txt > limpio.txt
cut -d: -f4 secretsdump.txt > nt_hashes.txt

# 2. Crackear
john nt_hashes.txt --format=NT --wordlist=/usr/share/wordlists/rockyou.txt --rules=Jumbo

# 3. Ver resultados
john nt_hashes.txt --format=NT --show
```

### Escenario 3 — Hash de /etc/shadow (Linux)
```bash
# 1. Combinar passwd y shadow
unshadow /etc/passwd /etc/shadow > combined.txt

# 2. Ataque con diccionario + reglas
john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules

# 3. Ver resultados
john combined.txt --show
```

### Escenario 4 — Base de datos KeePass
```bash
# 1. Extraer hash
keepass2john database.kdbx > hash_keepass.txt

# 2. Crackear
john hash_keepass.txt --wordlist=/usr/share/wordlists/rockyou.txt

# 3. Ver resultado
john hash_keepass.txt --show
```

### Escenario 5 — Hash AS-REP (Kerberoasting / AS-REP Roasting)
```bash
# Los hashes vienen directamente de herramientas como GetNPUsers.py
# Formato: $krb5asrep$23$usuario@DOMINIO:hash...

john asrep.txt --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt --rules=Jumbo
john asrep.txt --format=krb5asrep --show
```

### Escenario 6 — Contraseña desconocida, hash desconocido
```bash
# 1. Identificar el tipo de hash
hash-identifier "HASH_AQUI"
hashid -m "HASH_AQUI"

# 2. Probar en modo single primero (más rápido)
john hash.txt --single

# 3. Ataque con diccionario
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# 4. Diccionario con reglas
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --rules=Jumbo

# 5. Si todo falla, fuerza bruta acotada
john hash.txt --incremental=Alnum --max-length=8
```

---

## 🔗 Relaciones / Contexto

John the Ripper no es solo una herramienta de CTF — es una pieza fundamental del workflow de post-explotación en pentesting real. Tras obtener hashes de `/etc/shadow`, volcados NTLM con secretsdump, tickets Kerberos con Impacket, o claves SSH cifradas, John es habitualmente el siguiente paso. Los hashes crackeados permiten: movimiento lateral con credenciales reales, reutilización de contraseñas en otros servicios, acceso a gestores de contraseñas como KeePass, o simplemente demostrar al cliente el impacto real de una contraseña débil. En el contexto de Active Directory, crackear hashes NTLM del Administrador del dominio o hashes TGS de cuentas de servicio puede significar el compromiso total del entorno.