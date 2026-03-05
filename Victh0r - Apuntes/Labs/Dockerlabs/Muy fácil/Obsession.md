tags: #hydra #fuzzing  #escalation #muy-facil 
________________
Empezamos haciendo un escaneo básico de puertos con namp: 

```bash
nmap -sSCV -p- --min-rate 5000 -n -Pn -vvv --open -oN scan 172.17.0.2
```

### Parámetros del comando:
______________
- `nmap` — Herramienta de escaneo de red (Network Mapper).
- `-sS` — SYN scan: envía SYN sin completar handshake (rápido y relativamente sigiloso).
- `-sC` — Ejecuta scripts "default" (`--script=default`) para obtener información adicional.
- `-sV` — Detección de versiones: intenta identificar el software y versión del servicio.
- `-p-` — Escanea todos los puertos TCP (1–65535).
- `--min-rate 5000` — Forzar mínimo 5000 paquetes/segundo (acelera pero es ruidoso).
- `-n` — No resolver DNS (evita consultas DNS, más rápido).
- `-Pn` — Tratar host como "up" y omitir ping/host discovery.
- `-vvv` — Verbose muy detallado (más información de progreso).
- `--open` — Mostrar solo puertos en estado `open` (filtra cerrados/filtrados).
- `-oN scan` — Guardar la salida en formato normal en el fichero `scan`.
- `172.17.0.2` — Dirección IP objetivo a escanear.
______________________
![[Anexos/Pasted image 20251010055139.png]]

No vemos nada muy interesante así que vamos a ver que corre por el servicio web:

![[Anexos/Pasted image 20251010061916.png]]

La URL mostrada, nos lleva a una web externa de github, por lo que no es relevante para continuar.
Lo que si es relevante, es el formulario que se encuentra más abajo:

![[Anexos/Pasted image 20251010062148.png]]

Vamos a interceptar esta petición con burpsuite para ver como se está procesando por detrás:

![[Anexos/Pasted image 20251010062432.png]]

Antes de continuar por aquí, vamos a aplicar fuzzing para descubrir si hay directorios o archivos ocultos:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.xml,.txt,.html -t 40 -o dirs.txt
```

### Parámetros del comando:
___________________
- `gobuster` — Herramienta para fuerza/brute-force de directorios, DNS y hosts.
- `dir` — Modo de gobuster para buscar directorios/archivos en una URL.
- `-u http://172.17.0.2/` — URL base objetivo donde se realizan las peticiones.
- `-w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` — Ruta a la wordlist que contiene los nombres a probar.
- `-x .php,.xml,.txt,.html` — Extensiones que se añadirán a cada entrada de la wordlist (.php, .xml, .txt, .html).
- `-t 40` — Número de threads concurrentes (40 peticiones paralelas; alto paralelismo, ruidoso).
- `-o dirs.txt` — Archivo donde se guardan los resultados encontrados (salida).
_________________________
![[Anexos/Pasted image 20251010062650.png]]

```
http://172.17.0.2/important/
```

![[Anexos/Pasted image 20251010062735.png]]

```
http://172.17.0.2/backup/
```

![[Anexos/Pasted image 20251010062845.png]]

Voy a descargarme ambos archivos a mi directorio /content con el comando curl:

```bash
curl -O -X GET http://172.17.0.2/backup/backup.txt
curl -O -X GET http://172.17.0.2/important/important.md
```

En el archivo important.md no encontramos nada valioso, pero en el archivo backup.txt si:

![[Anexos/Pasted image 20251010064724.png]]

Este usuario ya lo habíamos encontrado en la web principal, abajo del todo con el siguiente correo: russoski@dockerlabs.es
Si no hubiese encontrado este archivo, posiblemente habría probado hacer fuerza bruta.
Tenemos que fijarnos que en la línea de texto dice que es su usuario para todos los servicios, lo que con lo cual podemos intuir que sirve tanto para el servicio ftp como para el ssh. Vamos a probar primero con el servicio ftp usando hydra:

```bash
hydra -l russoski -P /usr/share/wordlists/rockyou.txt ftp://172.17.0.2 -I
```

![[Anexos/Pasted image 20251010065728.png]]

![[Anexos/Pasted image 20251010065827.png]]

Hemos encontrado que para ambos servicios se usa la misma contraseña. Entramos primero al servicio ftp:

![[Anexos/Pasted image 20251010070601.png]]

Vemos dos directorios, con algunos archivos pero nada útil, así que nos conectamos por ssh y probamos hacer un **sudo -l**:

![[Anexos/Pasted image 20251010071702.png]]

Ya vemos claramente que tenemos permisos para ejecutar vim como sudo sin contraseña, así que vamos a GTFOBins a ver qué podemos hacer:

![[Anexos/Pasted image 20251010071842.png]]

Ejecutamos el comando en la máquina:

![[Anexos/Pasted image 20251010071921.png]]

Somos root!!