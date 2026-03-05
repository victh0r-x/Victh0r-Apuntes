tags: #smbmap #id_rsa #sudo
___
## Reconocimiento

```bash
ping -c 1 172.17.0.2
```

![[Anexos/Pasted image 20251029134103.png]]
Estamos ante un sistema linux, así que seguimos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- --min-rate 5000 -n -Pn -vvv -oN ports 172.17.0.3
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251029134357.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV --min-rate 5000 -vvv -n -Pn -p21,22,139,445 -vvv -oN version 172.17.0.2
```

![[Anexos/Pasted image 20251029134455.png]]

Vemos que tenemos el login anonymous activado en el servicio ftp, así que vamos a acceder y descargarnos el fichero nota.txt:

![[Anexos/Pasted image 20251029134607.png]]

![[Anexos/Pasted image 20251029134638.png]]

Aquí tenemos a dos posibles usuarios: Macarena y donald. Con smbmap solo vemos a macarena:

![[Anexos/Pasted image 20251029135011.png]]

Usamos el módulo smb_login de metasploit para hacer fuerza bruta al usuario macarena:

![[Anexos/Pasted image 20251029135841.png]]

![[Anexos/Pasted image 20251029135804.png]]

Resulta que la contraseña era donald.. Revisamos con smbmap los recursos compartidos de macarena:

```bash
smbmap -u macarena -p donald -H 172.17.0.2 -r macarena
```

![[Anexos/Pasted image 20251029141552.png]]

Nos descargamos el fichero user.txt con smbclient:

```bash
smbclient //172.17.0.2/macarena -U 'macarena%donald' -p 445 -c 'get "user.txt"' user.txt
```

![[Pasted image 20251029141701.png]]

![[Anexos/Pasted image 20251029142112.png]]

![[Pasted image 20251029142057.png]]

![[Pasted image 20251029143024.png]]

```bash
ssh-keygen -t rsa -b 4096
```

Subo el archivo con la clave pública al directorio que acabo de crear en el home de macarena .ssh:

![[Anexos/Pasted image 20251029145031.png]]

Ahora me conecto con mi archivo clave privada:

```bash
ssh -i id_rsa macarena@172.17.0.2
```

![[Anexos/Pasted image 20251029145207.png]]

En el directorio /home/secret se encuentra ese archivo hash, que lo vamos a decodificar en base32 y luego base64

![[Pasted image 20251029150341.png]]

![[Pasted image 20251029150447.png]]

Ahora ya podemos ejecutar el comando sudo -l y vemos lo siguiente:

![[Pasted image 20251029150716.png]]

![[Anexos/Pasted image 20251029150829.png]]

![[Pasted image 20251029151625.png]]

![[Anexos/Pasted image 20251029151757.png]]

