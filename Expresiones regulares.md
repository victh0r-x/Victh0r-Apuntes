___
## Tabla resumen

![](assets/Expresiones%20regulares-img-07-03-2026.png)
## 🧠 Concepto clave

Una **expresión regular** (regex o regexp) es un patrón de búsqueda que describe un conjunto de cadenas de texto. No es un lenguaje de programación ni una herramienta — es una **sintaxis universal** que entienden decenas de herramientas (`grep`, `sed`, `awk`, `find`, Python, Perl, etc.) para buscar, validar, extraer y transformar texto.

La idea fundamental es simple: en lugar de buscar exactamente `"password123"`, puedes describir el patrón `"una palabra seguida de exactamente 3 dígitos"` y encontrar `password123`, `username456`, `secretkey789`, etc. de una sola vez.

| Concepto | Qué es |
|---|---|
| **Patrón** | La expresión regular en sí — la descripción abstracta de lo que buscas |
| **Match / Coincidencia** | Una cadena de texto que satisface el patrón |
| **Motor de regex** | El componente software que interpreta el patrón y lo aplica al texto. Cada herramienta tiene el suyo con pequeñas diferencias |
| **BRE (Basic Regular Expressions)** | Sintaxis básica — la que usa `grep` por defecto. Algunos metacaracteres necesitan escape con `\` |
| **ERE (Extended Regular Expressions)** | Sintaxis extendida — la que usa `grep -E`, `egrep`, `awk`. Más cómoda, menos escapes necesarios |
| **PCRE (Perl Compatible Regular Expressions)** | La más potente — la que usa `grep -P`, Python, Perl. Añade lookaheads, lookbehinds, grupos nombrados y más |

> [!important] Las expresiones regulares **no son lo mismo** que los glob patterns del shell (`*.txt`, `file?.sh`). Los globs son expansión del shell para nombres de archivo. Las regex son para buscar dentro del **contenido** del texto. Son sistemas completamente distintos aunque compartan algunos símbolos como `*` y `?` con significados diferentes.

---

## 📌 Metacaracteres — Los bloques de construcción

Los metacaracteres son los caracteres con significado especial en regex. Todo lo demás se trata como texto literal.

### El punto `.` — Cualquier carácter

`.` representa **cualquier carácter único** excepto el salto de línea (`\n`).
```
Patrón:  c.t
Coincide: cat, cut, cot, c3t, c t, c@t
No coincide: ct (falta el carácter del medio), caat (son dos caracteres)
```
```bash
echo -e "cat\ncut\ncot\nct\ncaat" | grep "c.t"
# Resultado: cat, cut, cot
```

Para buscar un punto literal, escaparlo con `\`:
```bash
echo "version 1.0" | grep "1\.0"   # Coincide con "1.0" literal
echo "version 1.0" | grep "1.0"    # También coincide con "1X0", "1_0", etc.
```

---

### Anclas `^` y `$` — Posición en la línea

Las anclas no buscan caracteres — buscan **posiciones** en el texto.

| Ancla | Significa |
|---|---|
| `^` | Inicio de línea |
| `$` | Fin de línea |
| `^$` | Línea vacía |
| `^\s*$` | Línea vacía o que solo contiene espacios |
```bash
# Líneas que empiezan por "root"
grep "^root" /etc/passwd

# Líneas que terminan en ".conf"
grep "\.conf$" archivo.txt

# Líneas vacías
grep "^$" archivo.txt

# Contar líneas no vacías
grep -c "." archivo.txt
```

---

### Cuantificadores — Cuántas veces se repite algo

Los cuantificadores se aplican al elemento que les precede (un carácter, un grupo, una clase).

| Cuantificador | Significado | ERE/PCRE | BRE |
|---|---|---|---|
| `*` | 0 o más veces | `*` | `*` |
| `+` | 1 o más veces | `+` | `\+` |
| `?` | 0 o 1 vez (opcional) | `?` | `\?` |
| `{n}` | Exactamente n veces | `{n}` | `\{n\}` |
| `{n,}` | n o más veces | `{n,}` | `\{n,\}` |
| `{n,m}` | Entre n y m veces | `{n,m}` | `\{n,m\}` |
```bash
# * — cero o más "a"
echo -e "b\nab\naab\naaab" | grep -E "a*b"
# Coincide todo: b (cero "a"), ab, aab, aaab

# + — una o más "a" (requiere al menos una)
echo -e "b\nab\naab\naaab" | grep -E "a+b"
# Coincide: ab, aab, aaab (no "b" porque no hay ninguna "a")

# ? — la "s" es opcional
echo -e "color\ncolour" | grep -E "colou?r"
# Coincide: color y colour

# {3} — exactamente 3 dígitos
echo -e "12\n123\n1234" | grep -E "[0-9]{3}"
# Coincide: 123, 1234 (contiene 3 dígitos consecutivos)

# {3} anclado — exactamente 3 dígitos y nada más
echo -e "12\n123\n1234" | grep -E "^[0-9]{3}$"
# Coincide solo: 123
```

> [!important] **Greedy vs Lazy (codicioso vs perezoso)**
> Por defecto los cuantificadores son **greedy** (codiciosos) — intentan coincidir con el máximo texto posible. `.*` en `<b>texto</b>` coincidiría con todo `<b>texto</b>` y no solo con `texto`. En PCRE, añadir `?` después del cuantificador lo hace **lazy** (perezoso): `.*?` coincide con el mínimo posible. Esto es crítico para parsear HTML, JSON o cualquier texto con delimitadores repetidos.

---

### Clases de caracteres `[...]`

Una clase de caracteres define un **conjunto de caracteres aceptables** para una posición. Solo puede coincidir con **un único carácter** del conjunto.
```
[abc]     — coincide con "a", "b" o "c"
[a-z]     — cualquier letra minúscula (rango)
[A-Z]     — cualquier letra mayúscula
[0-9]     — cualquier dígito
[a-zA-Z]  — cualquier letra (mayúscula o minúscula)
[a-zA-Z0-9] — cualquier letra o dígito (alfanumérico)
[^abc]    — cualquier carácter EXCEPTO "a", "b" o "c" (negación)
[^0-9]    — cualquier carácter que NO sea dígito
```
```bash
# Encontrar líneas con vocales
grep "[aeiou]" archivo.txt

# Encontrar líneas que contengan solo letras y números
grep "^[a-zA-Z0-9]*$" archivo.txt

# Encontrar IPs (simplificado)
grep -E "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}" archivo.txt

# Caracteres especiales dentro de clases
# El guión "-" debe ir al principio o al final para ser literal
[a-z-]    # letras minúsculas o guión
[-a-z]    # igual, guión al principio
# El "^" solo es negación si va justo al principio
[a^b]     # coincide con "a", "^" o "b" — el ^ no es negación aquí
```

---

### Clases POSIX — Atajos predefinidos

Las clases POSIX son atajos estándar dentro de clases de caracteres, especialmente útiles para textos con caracteres no ASCII (acentos, etc.):

| Clase POSIX | Equivalente aproximado | Qué coincide |
|---|---|---|
| `[:alpha:]` | `[a-zA-Z]` | Letras |
| `[:digit:]` | `[0-9]` | Dígitos |
| `[:alnum:]` | `[a-zA-Z0-9]` | Letras y dígitos |
| `[:space:]` | `[ \t\n\r]` | Espacios en blanco |
| `[:upper:]` | `[A-Z]` | Mayúsculas |
| `[:lower:]` | `[a-z]` | Minúsculas |
| `[:punct:]` | Signos de puntuación | `!`, `.`, `,`, `;`, etc. |
| `[:print:]` | Caracteres imprimibles | Todo excepto control |
| `[:blank:]` | `[ \t]` | Espacio y tabulador |
```bash
# Uso — notar el doble corchete
grep "[[:alpha:]]" archivo.txt
grep "^[[:digit:]]*$" archivo.txt
grep "[[:upper:]][[:lower:]]*" archivo.txt  # Palabra capitalizada
```

---

### Secuencias de escape — Atajos en PCRE

En `grep -P` (PCRE) y otras herramientas compatibles:

| Escape | Qué coincide |
|---|---|
| `\d` | Dígito — equivale a `[0-9]` |
| `\D` | No dígito — equivale a `[^0-9]` |
| `\w` | Carácter de palabra — `[a-zA-Z0-9_]` |
| `\W` | No carácter de palabra — `[^a-zA-Z0-9_]` |
| `\s` | Espacio en blanco — `[ \t\n\r\f]` |
| `\S` | No espacio en blanco |
| `\b` | Límite de palabra (word boundary) |
| `\B` | No límite de palabra |
| `\t` | Tabulador |
| `\n` | Salto de línea |
| `\r` | Retorno de carro |
```bash
# \d — dígitos
echo "abc 123 def" | grep -P "\d+"
# Coincide: 123

# \w — caracteres de palabra
echo "hola mundo!" | grep -Po "\w+"
# Coincide: hola, mundo (excluye el !)

# \b — límite de palabra (no consume caracteres)
echo "cat concatenate category" | grep -oP "\bcat\b"
# Coincide solo: cat (la palabra completa, no "cat" dentro de otras palabras)

# \s — espacios
echo "hola   mundo" | grep -P "hola\s+mundo"
# Coincide: "hola   mundo" (uno o más espacios)
```

---

### Alternancia `|` — O lógico
```
gato|perro    — coincide con "gato" O "perro"
```
```bash
grep -E "error|warning|critical" /var/log/syslog
grep -E "^(root|admin|www-data):" /etc/passwd
```

En BRE necesita escape: `grep "error\|warning"`

---

### Grupos `(...)` — Agrupar y capturar

Los paréntesis tienen dos funciones: **agrupar** (para aplicar cuantificadores a varios caracteres) y **capturar** (para referenciar la coincidencia después).
```bash
# Agrupar para aplicar cuantificador
echo "hahaha" | grep -E "(ha)+"
# Coincide: hahaha (el grupo "ha" repetido 1 o más veces)

# Alternancia dentro de grupo
echo "colour" | grep -E "colo(u|)r"
# Coincide: colour y color (la "u" es opcional)

# Grupos no capturantes (?:...) en PCRE — agrupa sin capturar
echo "foobar" | grep -P "(?:foo)(bar)"
```

---

### Backreferences `\1`, `\2`... — Referenciar grupos capturados

Permiten referenciar dentro del mismo patrón lo que capturó un grupo anterior. Fundamental para detectar repeticiones.
```bash
# Detectar palabras duplicadas consecutivas
echo "the the cat" | grep -P "\b(\w+)\s+\1\b"
# Coincide: "the the"

# Detectar etiquetas HTML que abren y cierran igual
echo "<b>texto</b>" | grep -P "<(\w+)>.*</\1>"
# Coincide porque \1 es "b" — <b>...</b>
```

---

### Lookahead y Lookbehind (PCRE) — Aserciones de posición

Las aserciones de posición comprueban si algo está delante o detrás **sin consumir caracteres** — no forman parte de la coincidencia, solo condicionan si se produce.

| Sintaxis | Nombre | Significado |
|---|---|---|
| `(?=patrón)` | Lookahead positivo | Lo que sigue debe coincidir con `patrón` |
| `(?!patrón)` | Lookahead negativo | Lo que sigue NO debe coincidir con `patrón` |
| `(?<=patrón)` | Lookbehind positivo | Lo que precede debe coincidir con `patrón` |
| `(?<!patrón)` | Lookbehind negativo | Lo que precede NO debe coincidir con `patrón` |
```bash
# Lookahead positivo — "foo" seguido de "bar" (extrae solo "foo")
echo "foobar foobaz" | grep -oP "foo(?=bar)"
# Coincide: foo (solo el primero, porque el segundo no tiene "bar" después)

# Lookahead negativo — "foo" NO seguido de "bar"
echo "foobar foobaz" | grep -oP "foo(?!bar)"
# Coincide: foo (el de "foobaz")

# Lookbehind positivo — número precedido de "$"
echo "precio: $42 y 99 euros" | grep -oP "(?<=\$)\d+"
# Coincide: 42 (solo el número después del $)

# Lookbehind negativo — número NO precedido de "$"
echo "precio: $42 y 99 euros" | grep -oP "(?<!\$)\b\d+\b"
# Coincide: 99
```

---

## 🛠️ Herramientas que usan regex en Linux

### grep — Búsqueda en archivos y streams

> [!info] **grep (Global Regular Expression Print)**
> Herramienta fundamental de Unix para buscar líneas que coincidan con un patrón en archivos o stdin. Es la herramienta regex más usada en Linux. Existen tres versiones: `grep` (BRE por defecto), `egrep` = `grep -E` (ERE), `fgrep` = `grep -F` (strings literales, sin regex). La opción `-P` activa PCRE en versiones modernas de grep.

**Opciones más importantes de grep:**

| Flag | Función |
|---|---|
| `-E` | Activar ERE (Extended Regular Expressions) |
| `-P` | Activar PCRE (Perl Compatible) |
| `-i` | Case-insensitive |
| `-v` | Invertir — mostrar líneas que NO coinciden |
| `-o` | Mostrar solo la parte que coincide (no la línea completa) |
| `-n` | Mostrar número de línea |
| `-c` | Contar coincidencias (número de líneas) |
| `-l` | Mostrar solo nombres de archivos con coincidencias |
| `-r` / `-R` | Búsqueda recursiva en directorios |
| `-A n` | Mostrar n líneas After (después) de la coincidencia |
| `-B n` | Mostrar n líneas Before (antes) de la coincidencia |
| `-C n` | Mostrar n líneas de Context (antes y después) |
| `--color` | Colorear las coincidencias |
| `-w` | Coincidir solo palabras completas |
| `-x` | Coincidir solo líneas completas |
| `-m n` | Parar tras n coincidencias |
```bash
# Búsqueda básica
grep "error" /var/log/syslog

# Case-insensitive
grep -i "error" /var/log/syslog

# Recursivo en directorio
grep -r "password" /etc/

# Solo la parte que coincide
echo "IP: 192.168.1.100 conectada" | grep -oE "[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}"
# Resultado: 192.168.1.100

# Invertir — líneas sin comentarios y no vacías
grep -v "^#" /etc/ssh/sshd_config | grep -v "^$"

# Contexto alrededor de la coincidencia
grep -C 3 "error" /var/log/syslog

# Contar ocurrencias
grep -c "error" /var/log/syslog

# Buscar en múltiples archivos
grep -r "passwd" /etc/ --include="*.conf"

# Palabras completas — no coincide con "passwords" buscando "password"
grep -w "password" archivo.txt
```

---

### sed — Stream Editor

> [!info] **sed (Stream EDitor)**
> Editor de texto no interactivo que procesa el texto línea a línea aplicando comandos. Su uso más común con regex es la **sustitución** (`s/patrón/reemplazo/flags`), pero también puede eliminar líneas, insertar texto, y más. Opera sobre stdin o archivos y escribe en stdout — no modifica archivos en su lugar a menos que se use `-i`.

**Sintaxis de sustitución:**
```
sed 's/patrón/reemplazo/flags'
```

| Flag de sustitución | Función |
|---|---|
| `g` | Global — sustituir todas las ocurrencias (no solo la primera) |
| `i` | Case-insensitive |
| `n` | Solo la n-ésima ocurrencia (ej: `2` = solo la segunda) |
| `p` | Imprimir la línea si hubo sustitución (con `-n` muestra solo las modificadas) |
```bash
# Sustitución básica (solo primera ocurrencia por línea)
echo "hola hola mundo" | sed 's/hola/bye/'
# Resultado: bye hola mundo

# Sustitución global (todas las ocurrencias)
echo "hola hola mundo" | sed 's/hola/bye/g'
# Resultado: bye bye mundo

# Case-insensitive
echo "Hola HOLA hola" | sed 's/hola/bye/gi'
# Resultado: bye bye bye

# Usar grupos capturados en el reemplazo con \1, \2
echo "Juan García" | sed 's/\(.*\) \(.*\)/\2, \1/'
# Resultado: García, Juan

# Con ERE (-E) — sin necesitar escapes en grupos
echo "Juan García" | sed -E 's/(.*) (.*)/\2, \1/'
# Mismo resultado: García, Juan

# Eliminar líneas que coinciden con un patrón
sed '/^#/d' /etc/ssh/sshd_config   # Eliminar comentarios

# Eliminar líneas vacías
sed '/^$/d' archivo.txt

# Eliminar líneas vacías Y comentarios
sed '/^[[:space:]]*#/d; /^$/d' archivo.txt

# Modificar archivo en su lugar (con backup)
sed -i.bak 's/old/new/g' archivo.txt
# Crea archivo.txt.bak con el original y modifica archivo.txt

# Mostrar solo líneas modificadas
sed -n 's/error/ERROR/p' archivo.txt

# Sustitución en rango de líneas
sed '10,20s/foo/bar/g' archivo.txt   # Solo líneas 10 a 20

# Añadir línea después de coincidencia
sed '/patrón/a\nueva línea' archivo.txt

# Añadir línea antes de coincidencia
sed '/patrón/i\nueva línea' archivo.txt

# Extraer sección entre dos patrones
sed -n '/INICIO/,/FIN/p' archivo.txt
```

---

### awk — Procesamiento de campos

> [!info] **awk**
> Lenguaje de procesamiento de texto orientado a columnas/campos. Divide cada línea en campos usando un delimitador (por defecto espacios/tabuladores) y permite operar sobre ellos. Usa regex para filtrar líneas y para operar sobre campos. Es especialmente potente para procesar salidas de comandos con formato tabular (logs, `/etc/passwd`, salidas de `ps`, `netstat`, etc.).
```bash
# Sintaxis básica
awk '/patrón/ { acción }' archivo

# Imprimir líneas que coinciden con patrón
awk '/error/' /var/log/syslog

# Imprimir campo específico de líneas que coinciden
awk '/error/ { print $1, $5 }' /var/log/syslog
# $1 = primer campo, $5 = quinto campo, $NF = último campo

# Separador personalizado — extraer usuarios de /etc/passwd
awk -F: '{ print $1 }' /etc/passwd
# -F: — usa ":" como separador de campos

# Usuarios con UID mayor a 1000 (usuarios reales)
awk -F: '$3 >= 1000 { print $1, $3 }' /etc/passwd

# Regex en campos específicos
awk -F: '$7 ~ /bash/' /etc/passwd
# $7 ~ /bash/ — el campo 7 (shell) contiene "bash"

# Negación de regex en campo
awk -F: '$7 !~ /nologin/' /etc/passwd

# Contar líneas que coinciden
awk '/error/ { count++ } END { print count }' /var/log/syslog

# Sumar valores de una columna
awk '{ sum += $3 } END { print sum }' archivo.txt

# Extraer IPs del log
awk '{ match($0, /[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/, ip); if(ip[0]) print ip[0] }' /var/log/auth.log
```

---

### find — Búsqueda de archivos con regex

> [!info] **find con -regex**
> `find` soporta regex con la opción `-regex` para filtrar rutas completas de archivos. Por defecto usa BRE; con `-regextype posix-extended` usa ERE.
```bash
# Buscar archivos .log o .txt
find /var/log -regextype posix-extended -regex ".*\.(log|txt)"

# Buscar archivos que empiecen por "err" o "warn"
find /var/log -regextype posix-extended -regex ".*(err|warn).*"

# Combinado con -name (glob, más simple para nombres)
find /etc -name "*.conf" -type f
```

---

## 🎯 Patrones regex de referencia — Casos de uso reales

### Validación y extracción de formatos comunes
```bash
# Dirección IP v4
grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" archivo.txt

# Dirección IP v4 (más precisa — 0-255 por octeto, PCRE)
grep -oP "\b(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)(\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}\b" archivo.txt

# Dirección de email
grep -oE "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" archivo.txt

# URL (http/https)
grep -oE "https?://[a-zA-Z0-9./_?=&%-]+" archivo.txt

# Hash MD5 (32 hexadecimales)
grep -oE "\b[a-fA-F0-9]{32}\b" archivo.txt

# Hash SHA1 (40 hexadecimales)
grep -oE "\b[a-fA-F0-9]{40}\b" archivo.txt

# Hash SHA256 (64 hexadecimales)
grep -oE "\b[a-fA-F0-9]{64}\b" archivo.txt

# Fecha formato YYYY-MM-DD
grep -oE "[0-9]{4}-[0-1][0-9]-[0-3][0-9]" archivo.txt

# Fecha formato DD/MM/YYYY
grep -oE "[0-3][0-9]/[0-1][0-9]/[0-9]{4}" archivo.txt

# Hora HH:MM:SS
grep -oE "[0-2][0-9]:[0-5][0-9]:[0-5][0-9]" archivo.txt

# Número de tarjeta de crédito (simplificado)
grep -oE "\b[0-9]{4}[[:space:]-]?[0-9]{4}[[:space:]-]?[0-9]{4}[[:space:]-]?[0-9]{4}\b" archivo.txt

# Número de puerto (1-65535)
grep -oP "\b(6553[0-5]|655[0-2][0-9]|65[0-4][0-9]{2}|6[0-4][0-9]{3}|[1-5][0-9]{4}|[0-9]{1,4})\b" archivo.txt

# Dirección MAC
grep -oE "([0-9a-fA-F]{2}:){5}[0-9a-fA-F]{2}" archivo.txt

# UUID
grep -oE "[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}" archivo.txt

# Variable de entorno o credencial (clave=valor)
grep -oP "(?i)(password|passwd|secret|key|token|api_key)\s*=\s*\S+" archivo.txt
```

---

### Patrones útiles en pentesting y administración
```bash
# Extraer usuarios de /etc/passwd con shell bash
grep -P ".*bash$" /etc/passwd | cut -d: -f1

# Encontrar contraseñas en logs y configs
grep -rP "(?i)(password|passwd|pwd)\s*[=:]\s*\S+" /etc/ 2>/dev/null

# Extraer hashes NTLM de output de secretsdump
grep -oP "[a-fA-F0-9]{32}" secretsdump_output.txt

# Encontrar IPs privadas en logs
grep -oE "(10\.[0-9]{1,3}|172\.(1[6-9]|2[0-9]|3[01])|192\.168)\.[0-9]{1,3}\.[0-9]{1,3}" archivo.txt

# Extraer dominios de un archivo
grep -oP "(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}" archivo.txt

# Líneas de log que contengan error o warning (case-insensitive)
grep -iE "(error|warning|critical|fail)" /var/log/syslog

# Puertos en estado LISTEN de netstat
netstat -tlnp | grep -oP ":\K[0-9]+" | sort -n | uniq

# Procesos corriendo como root
ps aux | grep -E "^root"

# Buscar archivos SUID
find / -perm -4000 2>/dev/null | grep -v "^/proc"
```

---

## 📐 Diferencias BRE vs ERE vs PCRE — Tabla de referencia

| Característica | BRE (grep) | ERE (grep -E) | PCRE (grep -P) |
|---|---|---|---|
| Grupo capturante | `\(...\)` | `(...)` | `(...)` |
| Alternancia | `\|` | `\|` o `|` | `|` |
| `+` (uno o más) | `\+` | `+` | `+` |
| `?` (opcional) | `\?` | `?` | `?` |
| `{n,m}` | `\{n,m\}` | `{n,m}` | `{n,m}` |
| `\d`, `\w`, `\s` | ✗ | ✗ | ✓ |
| Lookahead/lookbehind | ✗ | ✗ | ✓ |
| Grupo no capturante | ✗ | ✗ | `(?:...)` |
| Backreferences | `\1` | `\1` | `\1` |
| Modo multiline | ✗ | ✗ | `(?m)` |

> [!tip] Regla práctica: usar siempre `grep -E` o `grep -P` en lugar de `grep` a secas. ERE es más cómodo que BRE (menos escapes), y PCRE añade `\d`, `\w`, lookaheads y más potencia. La única razón para usar BRE es compatibilidad con sistemas muy antiguos.

---

## 🔧 Modificadores de modo (PCRE)

En PCRE se pueden activar modos especiales dentro del patrón:

| Modificador | Función |
|---|---|
| `(?i)` | Case-insensitive desde ese punto |
| `(?m)` | Multiline: `^` y `$` coinciden al inicio/fin de cada línea (no solo del string completo) |
| `(?s)` | Dotall: `.` también coincide con `\n` |
| `(?x)` | Extended: ignora espacios y comentarios en el patrón — permite escribir regex legibles |
```bash
# Case-insensitive inline
echo "Error ERROR error" | grep -oP "(?i)error"

# Regex legible con modo (?x) — los espacios y # comentarios se ignoran
grep -P "(?x)
    \b          # inicio de palabra
    [0-9]{1,3}  # 1 a 3 dígitos
    \.          # punto literal
    [0-9]{1,3}  # 1 a 3 dígitos
    \.          # punto literal
    [0-9]{1,3}  # 1 a 3 dígitos
    \.          # punto literal
    [0-9]{1,3}  # 1 a 3 dígitos
    \b          # fin de palabra
" archivo.txt
```

---

## 🧪 Flujo de construcción de una regex — Metodología

Construir una regex compleja de golpe es un error frecuente. El enfoque correcto es **incremental**:

**Ejemplo: extraer el campo "user" de un log de auth.log**

Línea de ejemplo:
```
Mar  7 14:23:01 servidor sshd[12345]: Failed password for invalid user admin from 192.168.1.50 port 54321 ssh2
```
```bash
# Paso 1 — Filtrar las líneas relevantes
grep "Failed password" /var/log/auth.log

# Paso 2 — Extraer la parte que nos interesa
grep -oP "for (invalid user )?\K\S+" /var/log/auth.log
# \K — descarta todo lo que hay antes (como lookbehind)
# (invalid user )? — el texto "invalid user " es opcional
# \S+ — uno o más caracteres no-espacio (el nombre de usuario)

# Paso 3 — Añadir la IP también
grep -oP "for (invalid user )?\K\S+(?= from)" /var/log/auth.log
# (?= from) — lookahead: el usuario va seguido de " from"

# Paso 4 — Extraer usuario e IP juntos
grep -oP "for (invalid user )?\K(\S+) from (\S+)" /var/log/auth.log
```

> [!tip] Herramientas online para probar regex en tiempo real: **regex101.com** (soporta PCRE, Python, JS, con explicación de cada parte), **regexr.com** (visual e interactivo). Son imprescindibles para desarrollar y depurar patrones complejos.

---

## 🔗 Relaciones / Contexto

Las expresiones regulares son una habilidad transversal que multiplica la utilidad de prácticamente cualquier herramienta de línea de comandos. En el contexto de seguridad ofensiva, dominar regex permite: extraer credenciales de volcados de logs con precisión quirúrgica, parsear salidas de herramientas como nmap o nikto para automatizar pipelines, escribir reglas de detección en SIEM (Wazuh, Splunk, ELK), y procesar grandes volúmenes de datos de reconocimiento en segundos. La combinación `grep -oP + \K + lookahead/lookbehind` es especialmente potente para extracción de datos en pentesting — permite extraer exactamente el dato que necesitas sin texto sobrante, listo para pasar a la siguiente herramienta del pipeline.