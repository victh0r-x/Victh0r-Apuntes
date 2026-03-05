tags: 
_____
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- -n -Pn --min-rate 5000 -vvv 172.17.0.2 -oN ports
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Anexos/Pasted image 20251017050258.png]]

Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV -p21,22,80,3306 172.17.0.2 -oN version -n -Pn --min-rate 5000
```

![[Anexos/Pasted image 20251017050402.png]]

Vamos a acceder al servicio web para ver de qué se trata:

![[Anexos/Pasted image 20251017050719.png]]

Vemos una página de capybaras, donde además podemos ver que tenemos un usuario descubierto: **capybara**. 
Voy a intentar hacer un ataque de fuerza bruta con hydra al servicio mysql y al servicio ftp para ver qué consigo:

```bash
hydra -l capybara -P /usr/share/wordlists/rockyou.txt -I mysql://172.17.0.2
```

![[Anexos/Pasted image 20251017052306.png]]
 
![[Anexos/Pasted image 20251017053607.png]]

 >NOTA: Si alguna vez algo del lado del cliente da este error, podemos agregar el parámetro --skip_ssl
 
 ![[Anexos/Pasted image 20251017053935.png]]

Ahora vamos a echar un ojo a la base de datos con los siguientes comandos:

```sql
show databases;
show tables;
SELECT * from registraton;
```

![[Anexos/Pasted image 20251017054352.png]]

Ahora que tenemos un usuario y una contraseña, vamos a intentar crackearla con la web de crackstation:

![[Anexos/Pasted image 20251017054707.png]]

Vemos que la contraseña se ha reutilizado y sigue siendo password1. Ahora vamos a probarla con el **servicio** **ssh** y el usuario **balulero**:




